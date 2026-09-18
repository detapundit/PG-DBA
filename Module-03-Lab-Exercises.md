# Module 3 — Hands-On Lab
## PostgreSQL Architecture: Processes | Memory | Storage | Query Lifecycle

*Datapundit — PostgreSQL DBA Training Series*
*Lab environment: Rocky Linux 9, PostgreSQL 17 (PGDG), consistent with Module 2*

---

This lab walks through every concept in Module 3 hands-on, in the same order: process architecture (Part A), memory architecture (Part B), physical storage layout (Part C), query lifecycle (Part D), and the write path/WAL preview (Part E). Each exercise is self-contained with exact commands and what you should expect to see.

**Estimated time:** 60–90 minutes for all five labs.

---

## Lab 0: Prerequisites — Environment & Sample Data Setup

### 0.1 Verify PostgreSQL Is Installed and Running

```bash
# Confirm the binary and check the server is up (from Module 2)
which psql || /usr/pgsql-17/bin/psql --version
/usr/pgsql-17/bin/pg_ctl -D /var/lib/pgsql/17/data status
```

If it's not running:

```bash
/usr/pgsql-17/bin/pg_ctl -D /var/lib/pgsql/17/data start
```

### 0.2 Connect and Create a Dedicated Lab Database

Keeping this lab in its own database means the cleanup at the end is a single `DROP DATABASE`, and nothing here touches any other work you've done.

```bash
psql -U postgres
```

```sql
CREATE DATABASE architecture_lab;
\c architecture_lab
```

### 0.3 Create Sample Tables

```sql
CREATE TABLE departments (
    department_id   SERIAL PRIMARY KEY,
    department_name VARCHAR(50) NOT NULL
);

CREATE TABLE employees (
    employee_id   SERIAL PRIMARY KEY,
    first_name    VARCHAR(50),
    last_name     VARCHAR(50),
    department_id INT REFERENCES departments(department_id)
);

CREATE TABLE orders (
    order_id    BIGSERIAL PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date  DATE NOT NULL,
    amount      NUMERIC(10,2) NOT NULL
);
```

### 0.4 Load Sample Data

`departments` and `employees` are small (for the process/session exercises); `orders` is deliberately large — **200,000 rows** — so the query-lifecycle exercises in Lab 4 have a table big enough for the planner to make a real seq-scan-vs-index-scan decision, and so Lab 2's `work_mem` demo has enough data to force an on-disk sort.

```sql
INSERT INTO departments (department_name) VALUES
    ('Sales'), ('Engineering'), ('Finance'), ('Support');

INSERT INTO employees (first_name, last_name, department_id)
SELECT 'FirstName' || i, 'LastName' || i, ((i - 1) % 4) + 1
FROM generate_series(1, 1000) AS i;

INSERT INTO orders (customer_id, order_date, amount)
SELECT
    (random() * 10000)::int,
    DATE '2024-01-01' + (random() * 700)::int,
    (random() * 500)::numeric(10,2)
FROM generate_series(1, 200000) AS i;
```

### 0.5 Verify the Load

```sql
SELECT (SELECT count(*) FROM departments) AS departments,
       (SELECT count(*) FROM employees)   AS employees,
       (SELECT count(*) FROM orders)      AS orders;
```

**Expected:** `4 | 1000 | 200000`

```sql
-- Refresh planner statistics now, so Lab 4's plans are based on accurate stats
ANALYZE;
```

Keep this `psql` session open — you'll use it throughout. Stay connected to `architecture_lab` for every SQL step below unless a step tells you to open a second session.

---

## Lab 1: Process Architecture (Module 3, Part A)

**Objective:** See the postmaster, backend processes, and background processes as real OS processes, and connect a SQL-level view (`pg_stat_activity`) to the OS-level view (`ps`).

### 1.1 Find the Postmaster

```bash
cat /var/lib/pgsql/17/data/postmaster.pid | head -1
ps -ef | grep "bin/postgres" | grep -v grep
```

**Observe:** the first line of `postmaster.pid` is the PID that should match the top-level `postgres` process in `ps` — the supervisor for the whole cluster.

### 1.2 Identify Your Own Backend Process

In your existing `psql` session (still connected to `architecture_lab`):

```sql
SELECT pg_backend_pid();
```

Note the number it returns, then confirm it from the OS side:

```bash
ps -ef | grep <the_pid_you_noted> | grep -v grep
```

**Observe:** a dedicated `postgres` process exists for your session alone, distinct from the postmaster.

### 1.3 Two Sessions Side by Side

Open a **second terminal** and start a second `psql` session:

```bash
psql -U postgres -d architecture_lab
```

In this second session:

```sql
SELECT pg_backend_pid();
```

Back in your **first** session, list every connected backend:

```sql
SELECT pid, usename, datname, state, query
FROM pg_stat_activity
WHERE datname = 'architecture_lab';
```

**Observe:** two distinct rows, two distinct PIDs — one per connection — matching what `ps -ef` shows as two separate OS processes. This is the "multi-process, not multi-threaded" architecture from Part A.1, made concrete.

Leave the second session open — you'll use it again in Lab 2.

### 1.4 Identify Background (Utility) Processes

```bash
ps -ef | grep postgres | grep -v grep
```

**Observe:** alongside your two backend processes, you should see entries resembling:

```
postgres: checkpointer
postgres: background writer
postgres: walwriter
postgres: autovacuum launcher
postgres: logical replication launcher
```

Confirm two of them from the SQL side too:

```sql
-- Checkpointer / background writer activity counters
SELECT * FROM pg_stat_bgwriter;

-- WAL writer activity
SELECT * FROM pg_stat_wal;
```

**Checkpoint for this lab:** you should now be able to point at a live `ps -ef` listing and correctly label at least four different process roles without guessing.

---

## Lab 2: Memory Architecture (Module 3, Part B)

**Objective:** Observe shared vs. per-backend memory settings, and directly witness `work_mem` change a query's execution behavior — not just its speed, but *where the sort happens* (memory vs. disk).

### 2.1 Inspect Shared Memory Settings

```sql
SHOW shared_buffers;
SHOW wal_buffers;

SELECT name, setting, unit, context
FROM pg_settings
WHERE name IN ('shared_buffers', 'wal_buffers', 'max_connections');
```

**Observe:** note the `context` column — `postmaster` means changing it needs a full restart (tie this back to Module 2, Part C.3).

### 2.2 Inspect Local (Per-Backend) Memory Settings

```sql
SHOW work_mem;
SHOW maintenance_work_mem;
```

### 2.3 The work_mem Demo — Memory Sort vs. Disk Sort

This is the single most convincing hands-on demo in this lab. First, force a **small** `work_mem` and run a query that has to sort a lot of data:

```sql
SET work_mem = '64kB';

EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, order_date, amount
FROM orders
ORDER BY amount DESC;
```

**Observe:** in the plan's `Sort` node, look for a line like:

```
Sort Method: external merge  Disk: xxxxkB
```

This means PostgreSQL couldn't fit the sort in memory and spilled it to temporary files on disk — slower, and generating extra I/O.

Now raise `work_mem` generously and run the **exact same query**:

```sql
SET work_mem = '64MB';

EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, order_date, amount
FROM orders
ORDER BY amount DESC;
```

**Observe:** the `Sort Method` line should now read:

```
Sort Method: quicksort  Memory: xxxxkB
```

Same query, same data — a different `work_mem` produced a fundamentally different execution strategy. Compare the `Execution Time` between the two runs as well.

```sql
-- Return to the session/system default when you're done experimenting
RESET work_mem;
```

**Trainer note for self-study:** this is exactly the mechanism Module 3's trainer note warned about — imagine 200 concurrent connections each running a query like this with `work_mem` set too generously; multiply it out and you can see how a well-intentioned setting change becomes a memory exhaustion incident.

### 2.4 Connection-Scoped vs. Cluster-Wide Confirmation

In your **second** `psql` session (still open from Lab 1):

```sql
SHOW work_mem;
```

**Observe:** it shows the original/default value, not `64MB` — confirming `SET work_mem` in Lab 2.3 only affected that one session's local memory, not the whole cluster.

---

## Lab 3: Physical Storage Layout (Module 3, Part C)

**Objective:** Walk from a SQL table name down to the actual bytes on disk — directory structure, tablespaces, and relation forks.

### 3.1 Explore the PGDATA Directory

```sql
SHOW data_directory;
```

```bash
ls -la /var/lib/pgsql/17/data
```

**Observe:** match what you see against the Module 3, Part C.1 table — `base/`, `global/`, `pg_wal/`, `pg_xact/`, `postgresql.conf`, `pg_hba.conf`, `PG_VERSION`.

```sql
-- The same base/ listing, from inside SQL
SELECT pg_ls_dir('base');
```

### 3.2 Find Your Database's Own Subdirectory

```sql
SELECT oid, datname FROM pg_database WHERE datname = 'architecture_lab';
```

```bash
# Substitute the oid value returned above
ls -la /var/lib/pgsql/17/data/base/<database_oid>/ | head -20
```

**Observe:** a long list of numeric filenames — every table and index in `architecture_lab` lives here, named by OID, not by name.

### 3.3 Trace the employees Table to Its Actual File

```sql
SELECT pg_relation_filepath('employees');
SELECT relfilenode FROM pg_class WHERE relname = 'employees';
SELECT pg_size_pretty(pg_total_relation_size('employees'));
```

```bash
# Use the filepath returned above (relative to PGDATA)
ls -la /var/lib/pgsql/17/data/base/<database_oid>/<relfilenode>*
```

**Observe:** you should see the main file, and — since `employees` has had inserts and now `ANALYZE` has run — possibly `_fsm` (Free Space Map) and `_vm` (Visibility Map) fork files alongside it, matching Part C.3's table.

### 3.4 Create a Tablespace (SELinux-Aware, Rocky Linux 9)

```bash
# Create and correctly own a directory for the new tablespace
mkdir -p /var/lib/pgsql/17/lab_tablespace
chown postgres:postgres /var/lib/pgsql/17/lab_tablespace
chmod 700 /var/lib/pgsql/17/lab_tablespace

# This path is already under /var/lib/pgsql, which normally already carries
# the correct SELinux context -- confirm, and fix it explicitly if not
# (see Module 2, Part F.1 for the full explanation)
ls -Zd /var/lib/pgsql/17/lab_tablespace
# If the context shown is NOT postgresql_db_t, run:
# semanage fcontext -a -t postgresql_db_t "/var/lib/pgsql/17/lab_tablespace(/.*)?"
# restorecon -Rv /var/lib/pgsql/17/lab_tablespace
```

```sql
CREATE TABLESPACE lab_ssd LOCATION '/var/lib/pgsql/17/lab_tablespace';

CREATE TABLE audit_log (
    id        BIGSERIAL PRIMARY KEY,
    event     TEXT,
    logged_at TIMESTAMPTZ DEFAULT now()
) TABLESPACE lab_ssd;

INSERT INTO audit_log (event) VALUES ('lab tablespace test row');
```

```bash
# Confirm the new table's file physically landed in the new location, not base/
find /var/lib/pgsql/17/lab_tablespace -type f
```

**Observe:** PostgreSQL's `pg_tblspc/` directory inside `PGDATA` holds a symlink pointing at this external location — worth a quick look:

```bash
ls -la /var/lib/pgsql/17/data/pg_tblspc/
```

---

## Lab 4: Query Lifecycle — Parse, Rewrite, Plan, Execute (Module 3, Part D)

**Objective:** See each of the four stages produce an observable, distinct effect.

### 4.1 The Parser — Catching a Syntax Error

```sql
SELCT * FROM orders;
```

**Observe:** `ERROR: syntax error at or near "SELCT"` — this fails in the **parser**, before PostgreSQL has any concept of whether the `orders` table even exists. The parser only understands grammar, not object names yet.

### 4.2 The Rewriter — Expanding a View

```sql
CREATE VIEW high_value_orders AS
SELECT order_id, customer_id, amount
FROM orders
WHERE amount > 400;

EXPLAIN SELECT * FROM high_value_orders WHERE customer_id = 42;
```

**Observe:** the plan shows a filter on `orders` directly (`amount > 400 AND customer_id = 42`), even though your query only mentioned `high_value_orders` and `customer_id`. The **rewriter** substituted the view's definition into your query before the planner ever saw it — exactly Part D's description of rule expansion.

### 4.3 The Planner — Before and After an Index

First, force a baseline with no index on `order_date`:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE order_date = '2024-06-15';
```

**Observe:** a `Seq Scan` on `orders`, scanning all 200,000 rows to find matches.

Now create an index and repeat the identical query:

```sql
CREATE INDEX idx_orders_order_date ON orders (order_date);
ANALYZE orders;

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE order_date = '2024-06-15';
```

**Observe:** the plan should now show an `Index Scan` (or `Bitmap Heap Scan` + `Bitmap Index Scan`, depending on how many rows match), with a much lower `Execution Time`. This is the **planner** making a genuinely different decision, based on statistics from `ANALYZE`, about the same logical query.

### 4.4 The Executor — Real vs. Estimated Rows

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, SUM(amount)
FROM orders
WHERE order_date >= '2024-06-01'
GROUP BY customer_id
ORDER BY SUM(amount) DESC
LIMIT 10;
```

**Observe:** compare the planner's estimated `rows=` value on each node against the `actual rows=` value `ANALYZE` reports. This gap is exactly how a DBA later diagnoses bad query plans (full depth in the performance-tuning module) — for now, the goal is simply recognizing that `EXPLAIN ANALYZE` shows you both numbers side by side.

---

## Lab 5: The Write Path & WAL Preview (Module 3, Part E)

**Objective:** Watch the WAL position (LSN) advance in real time as you commit, connecting this module directly to Module 4.

### 5.1 Observe the LSN Move

```sql
SELECT pg_current_wal_lsn();

INSERT INTO employees (first_name, last_name, department_id)
VALUES ('Lab', 'Trainee', 1);

SELECT pg_current_wal_lsn();
```

**Observe:** the second LSN is higher than the first — your `INSERT` generated a WAL record, and the position advanced, exactly as Part E describes, before the change was ever necessarily written to the `employees` heap file itself.

### 5.2 Which WAL File Is That?

```sql
SELECT pg_walfile_name(pg_current_wal_lsn());
```

```bash
ls -la /var/lib/pgsql/17/data/pg_wal/ | grep <the_filename_returned_above>
```

**Observe:** the actual 16 MB WAL segment file physically exists in `pg_wal/`, matching what Module 4 will cover in full depth.

---

## Lab Wrap-Up Checklist

By the end of this lab, you should be able to, without notes:

- [ ] Point to the postmaster process and at least three background processes in a live `ps -ef` listing
- [ ] Explain why two `psql` sessions show up as two separate OS processes
- [ ] Reproduce the `work_mem` memory-sort-vs-disk-sort demo and explain what changed
- [ ] Trace a table name to its actual on-disk file via `pg_relation_filepath`
- [ ] Create a tablespace on Rocky Linux 9, including the SELinux context check
- [ ] Demonstrate all four query lifecycle stages with a distinct, observable example for each
- [ ] Show that a single `INSERT` advances the current WAL LSN

## Cleanup

```sql
\c postgres
DROP DATABASE architecture_lab;
DROP TABLESPACE lab_ssd;
```

```bash
# Only needed if the tablespace drop above succeeds and leaves the directory empty
rmdir /var/lib/pgsql/17/lab_tablespace 2>/dev/null || echo "Directory not empty or already removed -- check manually"
```

**Trainer note:** `DROP TABLESPACE` will fail with "tablespace is not empty" if any table (like `audit_log`) still lives in it — since `audit_log` lived inside `architecture_lab`, dropping the database first removes that table automatically, then the tablespace drop succeeds cleanly. If you created objects in `lab_ssd` from a *different* database during the lab, drop those first.

## Troubleshooting Reference (Common Issues Seen in This Lab Series)

| Symptom | Likely Cause | Fix |
|---|---|---|
| `sudo: command not found` | Minimal container image | Use `su - postgres` or run directly if already that user (Module 2) |
| `initdb: error: invalid locale name "en_US.UTF-8"` | Locale not generated on this image | `dnf install -y glibc-langpack-en`, or use `--locale=C.UTF-8` (Module 2) |
| `pg_ctl: directory "..." does not exist` | Debian-style path used against an RPM install | Use `/var/lib/pgsql/17/data`, not `/var/lib/postgresql/17/main` (Module 2) |
| `CREATE TABLESPACE` fails silently or errors on access | SELinux context missing on a custom path | `semanage fcontext` + `restorecon` (Module 2, Part F.1; Lab 3.4 above) |
| `Sort Method: external merge Disk` never appears in Lab 2.3 | `work_mem` set too high, or dataset too small | Confirm `SET work_mem = '64kB'` actually took effect with `SHOW work_mem;` before running the query |
