# Module 5: MVCC & VACUUM / Autovacuum
## Row Versioning | Dead Tuples & Bloat | VACUUM Internals | Autovacuum Tuning | Transaction ID Wraparound

*Datapundit — PostgreSQL DBA Training Series*

---

Module 3 introduced MVCC in passing (`xmin`/`xmax` columns) as the reason readers never block writers. Module 5 makes MVCC the main subject — because it's also the reason **VACUUM exists at all**, and unmanaged VACUUM/autovacuum is, in practice, one of the single most common causes of PostgreSQL production incidents. This is a module worth spending real training time on.

---

## Part A: The Problem MVCC Solves

### A.1 The Old Way: Lock-Based Concurrency

Many older/simpler database engines make a **reader wait** if a writer is currently modifying the same row (and vice versa) — using locks to serialize access. This is safe but kills concurrency under mixed read/write load.

### A.2 PostgreSQL's Way: Multi-Version Concurrency Control

Instead of locking rows for reads, PostgreSQL keeps **multiple versions of a row** and lets each transaction see the version that was correct as of *its own snapshot*. The practical result:

- **Readers never block writers**
- **Writers never block readers**
- Writers still block other writers on the *same row* (this hasn't gone away — two `UPDATE`s on the same row are still serialized)

---

## Part B: How MVCC Actually Works — Row Versions

### B.1 Every Row Carries Hidden Bookkeeping Columns

```sql
SELECT xmin, xmax, ctid, employee_id, first_name
FROM employees
WHERE employee_id = 1;
```

| Column | Meaning |
|---|---|
| `xmin` | Transaction ID (XID) that **created** this row version |
| `xmax` | Transaction ID that **deleted/superseded** this row version (0/null if still current) |
| `ctid` | Physical location (page, offset) of this specific row version |

### B.2 UPDATE Is Never Really "In Place"

This is the single most important mental model to give trainees: **PostgreSQL never overwrites a row version.** An `UPDATE` writes a brand-new tuple and marks the old one's `xmax`; a `DELETE` just sets `xmax` on the existing tuple. Nothing is physically removed at the moment of the statement.

```sql
-- Before: one row version, xmax is null (current)
SELECT xmin, xmax, ctid, price FROM products WHERE id = 1;

UPDATE products SET price = 299.00 WHERE id = 1;

-- After: a NEW row version exists (new xmin, new ctid).
-- The OLD version still physically exists on disk with xmax now set --
-- it's a "dead tuple" as of this transaction's commit.
SELECT xmin, xmax, ctid, price FROM products WHERE id = 1;
```

### B.3 Transaction IDs and Snapshots

Every transaction gets a **32-bit XID**. A transaction's snapshot decides visibility roughly as: *"show me row versions whose `xmin` is committed and happened before my snapshot, and whose `xmax` is either empty or not yet committed as of my snapshot."* This is what makes `SELECT` never need a lock to read consistent data.

```sql
SELECT txid_current();
```

---

## Part C: Isolation Levels — Snapshots in Practice

| Level | Snapshot taken |
|---|---|
| **Read Committed** *(default)* | A **new** snapshot for every statement — you can see other transactions' commits between statements in the same transaction |
| **Repeatable Read** | **One** snapshot for the entire transaction — every statement sees the same data no matter what else commits meanwhile |
| **Serializable** | Repeatable Read + active conflict detection (SSI) to guarantee the same result as some serial execution order |

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT SUM(balance) FROM accounts;
-- ... even if another session commits changes to accounts right now ...
SELECT SUM(balance) FROM accounts;  -- identical result, same snapshot
COMMIT;
```

---

## Part D: The Cost of MVCC — Dead Tuples and Bloat

Because old row versions aren't removed automatically, every `UPDATE` and `DELETE` leaves a **dead tuple** behind, taking up space and slowing down scans (the engine still has to skip past dead tuples) and index lookups. Left unmanaged, this is called **table/index bloat**.

```sql
-- See exactly how many dead vs. live tuples a table currently has
SELECT relname, n_live_tup, n_dead_tup, last_vacuum, last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 10;
```

**Trainer note:** A good live demo — run a tight loop of `UPDATE`s on one test row a few thousand times, then query `n_dead_tup` for that table. Trainees can watch bloat accumulate in real time before VACUUM runs.

---

## Part E: VACUUM

### E.1 What VACUUM Actually Does

- Scans the table and marks the space used by dead tuples as **reusable** by future inserts/updates — it does **not** shrink the file on disk or return space to the OS
- Updates the **Visibility Map** (from Module 3) so index-only scans and future vacuums can skip pages known to have no dead tuples
- Updates the **Free Space Map** so inserts know where free space exists

```sql
VACUUM employees;
VACUUM VERBOSE employees;      -- see exactly what it did
VACUUM (ANALYZE) employees;    -- also refresh planner statistics
```

### E.2 VACUUM vs. VACUUM FULL

| | `VACUUM` | `VACUUM FULL` |
|---|---|---|
| Locking | None — runs alongside normal reads/writes | **Exclusive lock** — table is inaccessible for the duration |
| What it does | Marks space reusable in place | Rewrites the entire table into a new file, physically reclaiming disk space |
| When to use | Routine, ongoing maintenance (this is what autovacuum does) | Rarely — only after a one-off event that caused massive bloat, during a maintenance window |

```sql
-- Only when you deliberately accept the downtime and truly need disk space back:
VACUUM FULL employees;
```

**Trainer note:** Strongly emphasize this distinction. New DBAs occasionally reach for `VACUUM FULL` as a routine fix, not realizing it takes an exclusive lock and can cause an outage on a busy production table.

### E.3 ANALYZE — A Related but Separate Job

`ANALYZE` refreshes the planner's statistics (row counts, value distributions) used for query planning (Module 3, Part D). It's bundled into autovacuum's job but is conceptually distinct from reclaiming dead-tuple space.

```sql
ANALYZE employees;
SELECT * FROM pg_stats WHERE tablename = 'employees';
```

---

## Part F: Autovacuum — Doing This Automatically

### F.1 The Autovacuum Launcher and Workers (recap from Module 3)

The autovacuum launcher periodically checks every table against thresholds and dispatches worker processes to run `VACUUM`/`ANALYZE` where needed — this is the default, and correct, way VACUUM should run on virtually every production table.

### F.2 When Autovacuum Decides to Run

```
vacuum threshold = autovacuum_vacuum_threshold + (autovacuum_vacuum_scale_factor × n_live_tup)
```

| Parameter | Default | Meaning |
|---|---|---|
| `autovacuum_vacuum_threshold` | 50 | Base number of dead rows before considering a vacuum |
| `autovacuum_vacuum_scale_factor` | 0.2 | + 20% of the table's live row count |
| `autovacuum_analyze_threshold` / `_scale_factor` | 50 / 0.1 | Same formula, for triggering `ANALYZE` |
| `autovacuum_max_workers` | 3 | Max concurrent autovacuum worker processes cluster-wide |
| `autovacuum_naptime` | 1 min | How often the launcher wakes up to check tables |

```sql
SHOW autovacuum_vacuum_scale_factor;
SHOW autovacuum_vacuum_threshold;

-- See it in action: how close is each table to triggering?
SELECT relname, n_live_tup, n_dead_tup,
       (n_dead_tup::float / GREATEST(n_live_tup,1)) AS dead_ratio
FROM pg_stat_user_tables
ORDER BY dead_ratio DESC
LIMIT 10;
```

### F.3 The Default Doesn't Fit Every Table

**This is the single most important practical lesson in this module.** The default `scale_factor` of 0.2 means a table needs 20% dead rows before autovacuum bothers — fine for a small table, but on a **500-million-row table**, that's 100 million dead tuples accumulating before autovacuum even starts. This is the classic "why is my huge table so bloated / slow" root cause.

```sql
-- Override thresholds per table for large, high-churn tables
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.02,
    autovacuum_vacuum_threshold = 1000,
    autovacuum_analyze_scale_factor = 0.01
);
```

### F.4 Throttling — Not Letting Autovacuum Hurt Production I/O

Autovacuum work is deliberately rate-limited so it doesn't overwhelm production I/O:

```sql
SHOW autovacuum_vacuum_cost_delay;
SHOW autovacuum_vacuum_cost_limit;
```

```sql
-- A busy OLTP table might tolerate more aggressive vacuuming
ALTER TABLE orders SET (autovacuum_vacuum_cost_limit = 2000);
```

---

## Part G: Transaction ID Wraparound — The Danger Zone

### G.1 Why It Matters

XIDs are **32-bit** — roughly 4.2 billion possible values, treated as a circular space (~2.1 billion "in the future" vs. "in the past" relative to any given XID). If a database ever consumed transactions faster than autovacuum could **freeze** old row versions, XID comparisons would eventually become ambiguous — rows that should look old could appear to be "from the future," causing **silent, catastrophic data loss** (rows going invisible). PostgreSQL protects against this aggressively.

### G.2 Freezing

Autovacuum doesn't just remove dead tuples — as part of its work, it also **freezes** old-but-still-live tuples by marking them with a special "always visible" XID, so their original `xmin` no longer needs comparison against the wraparound boundary.

```sql
SHOW autovacuum_freeze_max_age;
SHOW vacuum_freeze_min_age;

-- How close is each database to the wraparound danger zone?
SELECT datname, age(datfrozenxid) AS xid_age
FROM pg_database
ORDER BY xid_age DESC;
```

### G.3 The Emergency Scenario

If autovacuum is somehow disabled, starved, or unable to keep up (long-running transactions blocking cleanup is a common cause) for long enough, PostgreSQL enters increasingly aggressive states: warnings in the log, then a forced **single-user, vacuum-only mode** where the database refuses new transactions until freezing catches up — a genuine outage.

```sql
-- Find the actual culprit blocking cleanup: long-running/idle-in-transaction sessions
SELECT pid, state, xact_start, now() - xact_start AS duration, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY xact_start
LIMIT 10;
```

**Trainer note:** This is worth a dedicated few minutes — "wraparound" sounds theoretical until trainees see it's almost always caused by something mundane: a forgotten `BEGIN` with no `COMMIT`, an abandoned application connection sitting idle-in-transaction for days, or autovacuum being manually disabled and forgotten.

---

## Part H: Monitoring Checklist

```sql
-- Dead tuple ratio per table
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Wraparound distance per database (alert well before autovacuum_freeze_max_age)
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC;

-- Currently running vacuum operations, live progress
SELECT * FROM pg_stat_progress_vacuum;

-- Long-running / idle-in-transaction sessions (common autovacuum blockers)
SELECT pid, state, now() - xact_start AS duration, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY duration DESC;
```

---

## Discussion Questions / Exercises for Trainees

1. Explain why PostgreSQL's `UPDATE` is really "insert a new version + mark the old one dead" rather than an in-place change, and how that connects back to MVCC's read/write non-blocking guarantee.
2. Run a loop of 5,000 `UPDATE`s against one row in a test table, then query `n_dead_tup` before and after a manual `VACUUM`. What changed, and what didn't (check the file size)?
3. Why is `VACUUM FULL` dangerous to run casually on a production table, and what would you do instead if a table genuinely needs disk space reclaimed?
4. A 300-million-row table is bloating badly despite autovacuum running regularly. Using the formula in Part F.2, explain why the default `scale_factor` is likely the problem, and what you'd change.
5. Find (or simulate) a session sitting `idle in transaction` and explain, in terms of Part G, why it's dangerous to leave running for days.

## Key Takeaways

- **MVCC** lets readers and writers avoid blocking each other by keeping multiple row versions and giving each transaction a consistent snapshot — the tradeoff is that old versions become **dead tuples** that must be cleaned up.
- `UPDATE`/`DELETE` never modify a row version in place; they create new versions and mark old ones via `xmax`.
- **VACUUM** reclaims dead-tuple space for reuse (not disk space back to the OS); **VACUUM FULL** physically rewrites the table but takes an exclusive lock — reserve it for rare, deliberate maintenance windows.
- **Autovacuum's default thresholds are tuned for small-to-medium tables** — large, high-churn tables almost always need per-table overrides (`autovacuum_vacuum_scale_factor`, `_threshold`) to avoid serious bloat.
- **Transaction ID wraparound** is the most severe consequence of neglected vacuuming — freezing is autovacuum's defense against it, and `age(datfrozenxid)` is a metric worth alerting on, not just checking reactively.
- Idle-in-transaction sessions and long-running transactions are the most common real-world cause of autovacuum falling behind — monitoring `pg_stat_activity` belongs in the same operational routine as monitoring bloat.
