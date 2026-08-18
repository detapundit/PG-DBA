# Module 3: PostgreSQL Architecture
## Process Model | Memory Architecture | Storage Layout | Query Lifecycle


---

## Part A: Process Architecture

### A.1 The Client-Server Model

PostgreSQL follows a **multi-process** (not multi-threaded) architecture — a deliberate design choice from the original POSTGRES project. Every client connection gets its own OS-level backend process, isolated by the operating system itself. This is a key contrast point vs. MySQL's thread-per-connection model, worth calling out to trainees.

### A.2 The Postmaster

The **postmaster** is the first process started when PostgreSQL boots — the supervisor of everything else. Its jobs:

- Listens for incoming client connections on the configured port (default `5432`)
- Forks a new **backend process** for each accepted connection
- Starts and supervises all background utility processes
- Performs crash recovery on startup by replaying WAL if the previous shutdown wasn't clean

```bash
# Starting PostgreSQL launches the postmaster as PID 1 of the cluster
pg_ctl -D /var/lib/postgresql/17/main start

# The postmaster's PID is recorded here for the lifetime of the cluster
cat /var/lib/postgresql/17/main/postmaster.pid
```

### A.3 Backend (Client-Handling) Processes

Each connected client is served by a dedicated **backend process** (sometimes called a "postgres" server process). It parses, plans, and executes that client's queries, and dies when the client disconnects.

```sql
-- Every session can see its own backend's OS process ID
SELECT pg_backend_pid();

-- And see every connected backend's PID, state, and current query
SELECT pid, usename, datname, state, query
FROM pg_stat_activity;
```

```bash
# The same information from the OS side
ps -ef | grep postgres
```

**Trainer note:** This is a good live-demo moment — open two `psql` sessions, run `SELECT pg_backend_pid();` in each, then show both PIDs sitting side by side in `ps -ef`. It makes the "one process per connection" model concrete.

### A.4 Background (Utility) Processes

Alongside backends, the postmaster runs several permanent background processes:

| Process | Role |
|---|---|
| **checkpointer** | Periodically flushes all dirty shared-buffer pages to disk and writes a checkpoint record to WAL, bounding crash-recovery time |
| **background writer (bgwriter)** | Continuously writes some dirty pages to disk between checkpoints, smoothing I/O spikes |
| **WAL writer (walwriter)** | Flushes WAL buffer contents to the `pg_wal` files on disk periodically, independent of transaction commits |
| **autovacuum launcher + workers** | Automatically runs VACUUM/ANALYZE on tables as they accumulate dead tuples (deep dive in Module 5) |
| **archiver** | Copies completed WAL segments to an archive location when `archive_mode` is on (used for PITR — Module 6) |
| **logical replication launcher + workers** | Drive logical replication subscriptions (Module 7) |
| **syslogger** | Captures stderr output and writes it to the configured log destination |
| ~~stats collector~~ | **Removed in PostgreSQL 15** — cumulative statistics now live in shared memory instead of being collected by a separate UDP-based process and written to temp files. `pg_stat_*` views are unaffected from a query standpoint. |

```sql
-- Checkpointer and background writer activity are directly observable
SELECT * FROM pg_stat_bgwriter;

-- WAL writer / WAL generation stats
SELECT * FROM pg_stat_wal;
```

```bash
# All background processes are visible as separate OS processes
ps -ef | grep postgres
# ... postgres: checkpointer
# ... postgres: background writer
# ... postgres: walwriter
# ... postgres: autovacuum launcher
# ... postgres: logical replication launcher
```

---

## Part B: Memory Architecture

PostgreSQL memory splits cleanly into two categories: memory shared by every process, and memory private to each backend.

### B.1 Shared Memory

Allocated once at postmaster startup and attached to by every backend and background process.

| Area | Parameter | Purpose |
|---|---|---|
| Shared buffer cache | `shared_buffers` | Caches table/index pages in memory to avoid disk reads |
| WAL buffers | `wal_buffers` | Holds WAL records before they're flushed to `pg_wal` files |
| Commit log (CLOG) | *(internal, `pg_xact/`)* | Tracks commit/abort status of every transaction ID |
| Lock space | `max_locks_per_transaction` | Tracks row/table/advisory locks held across the cluster |

```sql
-- Inspect current shared memory settings
SHOW shared_buffers;
SHOW wal_buffers;

-- Or query several at once via the catalog
SELECT name, setting, unit, context
FROM pg_settings
WHERE name IN ('shared_buffers', 'wal_buffers', 'max_connections');
```

### B.2 Local (Per-Backend) Memory

Allocated separately inside each backend process for its own query work, freed when the query (or session) ends.

| Parameter | Purpose |
|---|---|
| `work_mem` | Memory for a single sort/hash operation (per operation, per query — can multiply fast with parallel + complex queries) |
| `maintenance_work_mem` | Memory for VACUUM, CREATE INDEX, ALTER TABLE ADD FOREIGN KEY |
| `temp_buffers` | Memory for temporary tables within a session |

```sql
SHOW work_mem;
SHOW maintenance_work_mem;

-- work_mem is commonly tuned per-session for a heavy reporting query
SET work_mem = '256MB';
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, SUM(amount)
FROM orders
GROUP BY customer_id
ORDER BY SUM(amount) DESC;
RESET work_mem;
```

**Trainer note:** This is the natural place to warn trainees about the classic sizing mistake — `work_mem` is *per sort/hash operation*, not per query or per connection. A query with several sorts, run by 200 concurrent connections, can consume `work_mem × operations × connections` in the worst case.

---

## Part C: Physical Storage Layout

### C.1 The PGDATA Directory

Every PostgreSQL cluster lives under one data directory (`PGDATA`), with a well-defined internal structure:

| Path | Contents |
|---|---|
| `base/` | One subdirectory per database, containing that database's table/index files |
| `global/` | Cluster-wide tables (e.g., `pg_database`, `pg_authid`) |
| `pg_wal/` | WAL segment files — the write-ahead log (Module 4) |
| `pg_xact/` | Commit status of every transaction (formerly `pg_clog`) |
| `pg_multixact/` | Status data for multi-transaction row locks |
| `pg_tblspc/` | Symlinks to tablespace locations outside `PGDATA` |
| `postgresql.conf` | Main configuration file |
| `pg_hba.conf` | Client authentication rules |
| `PG_VERSION` | Single-line file recording the major version this data directory was initialized with |

```sql
-- List the base/ subdirectory contents from inside SQL
SELECT pg_ls_dir('base');

-- Ask PostgreSQL where its data directory actually is
SHOW data_directory;
```

```bash
# The same view from the shell
ls -la $PGDATA
```

### C.2 Tablespaces

A **tablespace** lets you store specific databases/tables/indexes on a different filesystem location or storage device — useful for putting hot indexes on faster disks.

```sql
CREATE TABLESPACE fast_ssd LOCATION '/mnt/nvme01/pgdata';

CREATE TABLE audit_log (
    id      BIGSERIAL PRIMARY KEY,
    event   TEXT,
    logged_at TIMESTAMPTZ DEFAULT now()
) TABLESPACE fast_ssd;

-- Move an existing index to a tablespace
ALTER INDEX idx_employees_lastname SET TABLESPACE fast_ssd;
```

### C.3 How a Table Is Actually Stored

Each table/index is stored as one or more physical files, named by OID (object identifier) rather than by table name — and each relation actually has multiple **forks**:

| Fork | Suffix | Purpose |
|---|---|---|
| Main | *(none)* | The actual heap data (rows) |
| Free Space Map | `_fsm` | Tracks available space in each page, for insert placement |
| Visibility Map | `_vm` | Tracks which pages contain only tuples visible to all transactions — lets VACUUM and index-only scans skip pages |
| Init fork | `_init` | Used only for unlogged tables, to reset them after a crash |

Files are capped at **1 GB per segment**; a large table's main fork becomes `16384`, `16384.1`, `16384.2`, and so on.

```sql

-- 1. Create the database
CREATE DATABASE sample_db;

-- 2. Connect to the new database
\c sample_db

-- 3. Create a table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
-- Find the actual on-disk file path and OID for a table
SELECT pg_relation_filepath('employees');
SELECT relfilenode FROM pg_class WHERE relname = 'employees';

-- Size on disk right now
SELECT pg_size_pretty(pg_total_relation_size('employees'));
```

```bash
# Confirm the file exists exactly where PostgreSQL says it does
ls -la $PGDATA/base/<database_oid>/ | grep <relfilenode>
```

---

## Part D: Query Lifecycle — From SQL Text to Result Set

Every query a backend receives passes through four stages:

1. **Parser** — checks SQL syntax, builds a parse tree; catches syntax errors here (`ERROR: syntax error at or near...`)
2. **Rewriter** — applies any rules (e.g., expands views into their underlying query, applies `RULE` definitions)
3. **Planner/Optimizer** — the cost-based planner evaluates possible execution strategies (sequential scan vs. index scan, join order, join algorithm) using table statistics from `ANALYZE`, and picks the cheapest estimated plan
4. **Executor** — walks the chosen plan tree and actually fetches/produces rows

```sql
-- EXPLAIN shows the planner's chosen plan without running the query
EXPLAIN
SELECT customer_id, SUM(amount)
FROM orders
WHERE order_date >= '2026-01-01'
GROUP BY customer_id;

-- EXPLAIN ANALYZE actually runs it and shows real vs. estimated rows/time
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT customer_id, SUM(amount)
FROM orders
WHERE order_date >= '2026-01-01'
GROUP BY customer_id
ORDER BY SUM(amount) DESC;
```

**Trainer note:** This is a strong bridge into a later performance-tuning module — for now, the goal is just for trainees to recognize that "the planner chose this plan because of statistics" is a normal part of the lifecycle, not a black box.

---

## Part E: The Write Path — A Preview

A single `UPDATE` or `INSERT` actually touches several of the pieces above at once, which is worth walking through once here as a preview of Module 4:

1. The backend process modifies the row **in shared_buffers** (not on disk yet)
2. Before that page can be marked dirty and eventually written, a **WAL record** describing the change is written to the **WAL buffer**
3. On `COMMIT`, PostgreSQL guarantees the WAL record is flushed to the `pg_wal/` files on disk (this flush — not the data page write — is what makes the transaction durable)
4. The actual dirty data page in `shared_buffers` is written to its heap file **later**, by the background writer or the next checkpoint — not immediately

```sql
-- You can watch WAL position advance in real time as you commit
SELECT pg_current_wal_lsn();
INSERT INTO employees (first_name, last_name) VALUES ('Test', 'Row');
SELECT pg_current_wal_lsn();  -- LSN has moved forward
```

This separation — "log the change first, write the data page whenever" — is **Write-Ahead Logging**, and it's the single most important mechanism in PostgreSQL for both durability and crash recovery. Module 4 covers it in full depth.

---

## Discussion Questions / Exercises for Trainees

1. Why does PostgreSQL fork a new OS process per connection instead of using threads? What's one operational tradeoff of that design (hint: think about connection pooling and `max_connections`).
2. Run `SELECT pid, state, query FROM pg_stat_activity;` in one session while a long query runs in another. Identify which row is your own backend.
3. Explain why `work_mem` sizing needs to account for concurrent connections and not just a single query.
4. Look up your own cluster's `PGDATA` directory (`SHOW data_directory;`) and list what's inside `pg_wal/` right now.
5. Run `EXPLAIN` on a simple `SELECT ... WHERE` query on an indexed column, then again on an unindexed column. Compare the plans.

## Key Takeaways

- PostgreSQL uses a **multi-process architecture** — a postmaster supervises one backend per client connection plus a fixed set of background utility processes (checkpointer, bgwriter, walwriter, autovacuum, archiver, and since PG15 no separate stats collector).
- Memory splits into **shared memory** (buffer cache, WAL buffers, lock space — sized once at startup) and **per-backend local memory** (`work_mem`, `maintenance_work_mem` — allocated and freed per session/operation).
- On disk, a cluster is one `PGDATA` directory; each table/index is a set of OID-named files split into **forks** (main, FSM, VM) and 1 GB segments.
- Every query passes through **Parse → Rewrite → Plan → Execute**, and `EXPLAIN (ANALYZE)` is the window into that process.
- Writes always touch **shared_buffers first, then WAL** — the actual heap file write happens later. This write-ahead principle is the foundation for Module 4.
