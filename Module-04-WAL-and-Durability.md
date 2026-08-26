# Module 4: WAL & Durability
## Write-Ahead Logging | Checkpoints | fsync & Commit Guarantees | Crash Recovery

*Datapundit — PostgreSQL DBA Training Series*

---

Module 3 ended with a preview: every write touches `shared_buffers` first, and a WAL record is written before that page is ever flushed to its heap file. Module 4 makes that mechanism the main subject — it's the single feature responsible for PostgreSQL's **Durability** guarantee (the "D" in ACID), and it underpins backup/PITR (Module 6) and replication (Module 7) as well.

---

## Part A: The Problem WAL Solves

### A.1 Why Not Just Write Data Pages Directly?

Imagine PostgreSQL had no WAL, and every `UPDATE` wrote its change straight into the table's heap file on disk. Two problems immediately appear:

- **Performance:** data pages are 8 KB and scattered randomly across a large file; flushing one on every commit means constant random I/O.
- **Crash safety:** if the server loses power mid-write, a page can be left **torn** — half old bytes, half new bytes — corrupting the table with no way to know what happened.

### A.2 The WAL Principle

PostgreSQL's answer, inherited from decades of database research: **never let a data page's on-disk copy get ahead of a log record describing the change.**

> *"Write the log first, apply the data later."* — this single rule is what "Write-Ahead Logging" means.

WAL records are appended **sequentially** to a log file — cheap, fast I/O — instead of scattered random writes. The actual heap page update can then happen lazily, whenever convenient, because if the server crashes before that page write happens, PostgreSQL can **replay the WAL record** on restart and reconstruct the change.

```sql
-- The guarantee in one sentence: once this COMMIT returns successfully,
-- the change is durable -- even if the server loses power one microsecond later.
BEGIN;
UPDATE accounts SET balance = balance - 500 WHERE account_id = 'A1';
COMMIT;
```

---

## Part B: WAL Structure

### B.1 WAL Records and LSNs

Every change is written as a **WAL record**, identified by a **Log Sequence Number (LSN)** — a monotonically increasing byte offset into the WAL stream. LSNs are the coordinate system used throughout PostgreSQL for replication lag, backup consistency points, and recovery targets.

```sql
-- Current write position in the WAL stream
SELECT pg_current_wal_lsn();

-- Watch it advance after a write
INSERT INTO employees (first_name, last_name) VALUES ('Ravi', 'Shetty');
SELECT pg_current_wal_lsn();
```

### B.2 WAL Segment Files

WAL is physically stored as a sequence of fixed-size **segment files** (16 MB by default) inside `pg_wal/`, named as 24-character hex filenames encoding the timeline ID and segment number.

```sql
-- Which segment file is the current LSN in?
SELECT pg_walfile_name(pg_current_wal_lsn());

-- List WAL files directly via SQL
SELECT * FROM pg_ls_waldir() ORDER BY modification DESC LIMIT 5;
```

```bash
# Same thing from the shell
ls -la $PGDATA/pg_wal/
```

### B.3 wal_level — Controlling How Much Gets Logged

| Level | What's Logged | Enables |
|---|---|---|
| `minimal` | Only what's needed for crash recovery | Smallest WAL volume; no replication possible |
| `replica` *(default)* | + enough for standby servers to reconstruct data | Streaming replication, PITR/archiving |
| `logical` | + enough to decode row-level changes | Logical replication, CDC tools |

```sql
SHOW wal_level;

-- Changing it requires a restart, and is typically set in postgresql.conf
-- wal_level = replica
```

**Trainer note:** worth flagging early that `wal_level = replica` is the practical default for almost every production cluster, since even a single standby or `pg_basebackup` needs it — `minimal` is rare outside single-node, no-backup toy setups.

---

## Part C: How WAL Gets Written — The Commit Path

### C.1 WAL Buffers → WAL Writer → Disk

Recall from Module 3: WAL records are first written into an in-memory **WAL buffer** (`wal_buffers`), then flushed out to the actual segment files in `pg_wal/` — either by the dedicated **WAL writer** process on a timer, or immediately when a transaction commits.

```sql
SHOW wal_buffers;
```

### C.2 fsync — The Real Durability Guarantee

Writing to a file isn't enough by itself — the OS may still be holding those bytes in its page cache. `fsync` is the system call that forces the OS to actually flush to physical storage. PostgreSQL's `fsync` parameter controls whether this happens at all.

```sql
SHOW fsync;
```

> **This should never be `off` in production.** Turning it off makes writes faster but removes the crash-safety guarantee entirely — a power loss can silently corrupt the cluster. It exists mainly for throwaway test/benchmark instances.

### C.3 synchronous_commit — How Aggressively COMMIT Waits

This parameter controls exactly what a client's `COMMIT` waits for before returning "success" — a direct, tunable tradeoff between durability and commit latency.

| Value | COMMIT returns after... |
|---|---|
| `on` *(default)* | WAL flushed to local disk |
| `remote_apply` | WAL flushed locally **and** applied on the sync standby |
| `remote_write` | WAL flushed locally and written (not necessarily flushed) on the standby |
| `local` | WAL flushed to local disk only, ignoring any standby |
| `off` | COMMIT returns immediately, WAL flushed later — fastest, but a crash can lose recently "committed" transactions |

```sql
-- Per-session override, common for bulk-load jobs where losing the last
-- few transactions on a crash is an acceptable tradeoff for speed
SET synchronous_commit = off;
INSERT INTO staging_table SELECT * FROM source_table;
RESET synchronous_commit;
```

### C.4 full_page_writes — Guarding Against Torn Pages

After each checkpoint, the *first* modification to any given page also writes the entire 8 KB page image into WAL (not just the changed bytes). This protects against torn pages if the OS/disk write is interrupted mid-page during a crash. It's why WAL volume often spikes right after a checkpoint.

```sql
SHOW full_page_writes;
```

---

## Part D: Checkpoints

### D.1 What a Checkpoint Does

A **checkpoint** is the point where PostgreSQL guarantees all dirty pages up to a certain WAL position have been flushed to their actual heap/index files on disk, and writes a checkpoint record to WAL marking that fact. On restart after a crash, recovery only needs to replay WAL **from the last checkpoint forward** — not from the beginning of time.

```sql
-- Force one manually (rarely needed in normal operation)
CHECKPOINT;

-- Inspect checkpoint activity and timing
SELECT * FROM pg_stat_bgwriter;
```

### D.2 What Triggers a Checkpoint

| Parameter | Default | Meaning |
|---|---|---|
| `checkpoint_timeout` | 5 min | Maximum time between automatic checkpoints |
| `max_wal_size` | 1 GB | Soft cap on WAL volume since the last checkpoint before one is triggered early |
| `checkpoint_completion_target` | 0.9 *(since PG14)* | Spreads the checkpoint's I/O across ~90% of the interval until the next one, smoothing disk load instead of bursting |

```sql
SHOW checkpoint_timeout;
SHOW max_wal_size;
SHOW checkpoint_completion_target;
```

### D.3 The Recovery-Time Tradeoff

More frequent checkpoints → shorter crash-recovery time, but more I/O overhead during normal operation. Less frequent checkpoints → less I/O overhead, but more WAL to replay if the server crashes. This is a core operational tuning decision, revisited properly in the performance-tuning module.

---

## Part E: Crash Recovery

### E.1 pg_control and the Redo Point

A small file, `global/pg_control`, records the location of the last completed checkpoint — PostgreSQL's starting point ("REDO point") for recovery.

```bash
# Inspect the control file's contents directly
pg_controldata $PGDATA
```

### E.2 The Recovery Sequence

On startup after an unclean shutdown, PostgreSQL automatically:

1. Reads `pg_control` to find the last checkpoint's location
2. Replays every WAL record from that point forward, reapplying changes to data pages
3. Once WAL is exhausted, the database is consistent and opens for connections

No manual intervention or DBA action is required for a normal crash — this is fully automatic, and is the entire reason WAL exists in the first place.

```bash
# You can watch this happen in the server log immediately after an unclean restart
tail -f $PGDATA/log/postgresql-*.log
# LOG:  database system was not properly shut down; automatic recovery in progress
# LOG:  redo starts at 0/3000028
# LOG:  redo done at 0/30000F8
# LOG:  database system is ready to accept connections
```

---

## Part F: WAL Archiving — A Preview

Beyond crash recovery, PostgreSQL can be configured to copy completed WAL segments to a separate archive location before they're recycled — the foundation for Point-in-Time Recovery (PITR), covered fully in Module 6.

```sql
SHOW archive_mode;
SHOW archive_command;
```

```ini
# A typical postgresql.conf snippet (not run interactively)
archive_mode = on
archive_command = 'cp %p /mnt/wal_archive/%f'
```

```sql
-- Manually force the current WAL segment to close and become archivable --
-- useful right before taking a base backup
SELECT pg_switch_wal();
```

---

## Discussion Questions / Exercises for Trainees

1. In your own words, explain why sequential WAL writes are faster than random data-page writes, and why that performance difference matters at scale.
2. What's the practical risk of setting `fsync = off` on a production server, even briefly?
3. A reporting team wants faster bulk-load performance and says losing the last few seconds of data on a crash is acceptable. Which parameter from Part C would you discuss with them, and why?
4. Run `SELECT pg_current_wal_lsn();`, perform ten inserts in a loop, and run it again. Explain what the difference in LSN represents physically on disk.
5. If `checkpoint_timeout` is increased from 5 minutes to 30 minutes, what improves and what gets worse? Frame your answer around Part D.3.

## Key Takeaways

- **Write-Ahead Logging** guarantees durability by writing a sequential log record before any data page change reaches disk — the reverse order would risk corruption on crash.
- **LSNs** are PostgreSQL's universal coordinate system for WAL position, used across replication, backup, and recovery.
- `wal_level`, `fsync`, `synchronous_commit`, and `full_page_writes` each control a different durability/performance tradeoff — none of them should be casually changed on a production system without understanding what's being traded away.
- A **checkpoint** bounds crash-recovery time by guaranteeing all changes before it are safely on disk; `checkpoint_timeout`, `max_wal_size`, and `checkpoint_completion_target` (default 0.9 since PG14) control how often and how smoothly checkpoints happen.
- Crash recovery is **fully automatic** — PostgreSQL replays WAL from the last checkpoint on every unclean restart, with no DBA intervention needed.
- WAL archiving extends this same mechanism into **Point-in-Time Recovery**, the subject of Module 6.
