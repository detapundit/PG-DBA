# Module 2: Installation & Configuration
## Install Methods | initdb | Service Management | postgresql.conf | pg_hba.conf | Day-One Hardening

*Datapundit — PostgreSQL DBA Training Series*

---

Module 1 covered the "why PostgreSQL" story. Before going anywhere near internals, a DBA needs to actually **stand up a cluster the right way** — because several decisions made at install time (locale, checksums, encoding) are difficult or impossible to change later, and several config choices made on day one (authentication method, logging, connection limits) are what you'll be firefighting at 2 AM if you get them wrong. This module is deliberately hands-on and checklist-driven — it's the module trainees should be able to follow along with on a real VM.

---

## Part A: Installation Methods

### A.1 OS Package Manager — the PGDG Repository (Recommended)

Most Linux distributions ship PostgreSQL in their default repos, but usually an **older, lagging version**. For any serious DBA work, install from the official **PostgreSQL Global Development Group (PGDG)** repository instead, which tracks upstream releases directly and lets you pick your exact major version.

```bash
# Debian/Ubuntu — add the PGDG repo, then install a specific major version
sudo apt install -y curl ca-certificates
sudo install -d /usr/share/postgresql-common/pgdg
curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc \
  https://www.postgresql.org/media/keys/ACCC4CF8.asc
echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] \
  https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
  | sudo tee /etc/apt/sources.list.d/pgdg.list
sudo apt update
sudo apt install -y postgresql-17

# RHEL/Rocky/Alma — PGDG yum repo
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql17-server postgresql17-contrib
```

**Trainer note:** This is worth emphasizing strongly — a huge number of real-world PostgreSQL incidents trace back to someone running `apt install postgresql` and unknowingly getting whatever ancient version their distro happened to package, with no easy path to a newer major version later.

### A.2 Source Compilation

Rarely needed day-to-day, but a DBA should know when it applies: custom build flags, applying a specific patch before it's released, or building on an unsupported platform.

```bash
./configure --prefix=/usr/local/pgsql17 --with-openssl --with-libxml
make -j4
sudo make install
```

### A.3 Containers

Common for dev/test environments and increasingly for production with orchestration (Kubernetes/StatefulSets). A DBA managing containerized PostgreSQL still needs to understand everything in this module — a container just relocates where `PGDATA` physically lives.

```bash
docker run -d --name pg17 \
  -e POSTGRES_PASSWORD=change_me \
  -v pgdata17:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:17
```

### A.4 Managed Cloud Services (Awareness, Not Hands-On Here)

Amazon RDS/Aurora, Google Cloud SQL, and Azure Database for PostgreSQL abstract away `initdb`, OS patching, and some config access — a DBA should know these exist and roughly what control is *lost* (no filesystem/superuser access, restricted `postgresql.conf` parameters) versus a self-managed install like the rest of this module covers.

---

## Part B: Initializing a Cluster — initdb

### B.1 What initdb Actually Does

`initdb` creates a brand-new, empty **cluster**: the `PGDATA` directory structure (from Module 3 — `base/`, `global/`, `pg_wal/`, etc.), the `template0`/`template1`/`postgres` databases, and the default configuration files. Package installs usually run this for you automatically — but a DBA should know how to do it manually, since you'll need it for a second cluster on the same box, a restore scenario, or a from-scratch build.

```bash
sudo -u postgres /usr/lib/postgresql/17/bin/initdb \
  -D /var/lib/postgresql/17/main \
  --encoding=UTF8 \
  --locale=en_US.UTF-8 \
  --data-checksums \
  --auth-local=peer \
  --auth-host=scram-sha-256
```

### B.2 Decisions You Cannot Easily Change Later

| Decision | Why it matters | DBA guidance |
|---|---|---|
| **Encoding** (`--encoding`) | Set per-cluster at initdb time | `UTF8` unless you have a specific legacy reason not to |
| **Locale / Collation** (`--locale`, or `--locale-provider=icu`) | Affects sort order, `LIKE`, indexes on text columns | Pick deliberately; changing collation later can silently invalidate existing indexes |
| **Data checksums** (`--data-checksums`) | Detects silent storage-level corruption | **As of PostgreSQL 18, `initdb` enables checksums by default** — a genuinely significant change, since for years DBAs had to remember to pass `--data-checksums` explicitly. On PG 17 and earlier, always pass it explicitly; there is essentially never a good reason to skip it on a production cluster |

```sql
-- Verify whether the running cluster has checksums on
SHOW data_checksums;

-- If you're on an older cluster without them, check pg_checksums (offline operation
-- on PG12+; PostgreSQL 19 is introducing an online enable/disable capability)
```

```bash
pg_checksums --pgdata=/var/lib/postgresql/17/main --enable
```

### B.3 Multiple Clusters on One Server

Debian/Ubuntu's PGDG packages provide convenience tooling for running several clusters (e.g., PG16 and PG17 side by side during a major-version upgrade window):

```bash
pg_lsclusters
pg_createcluster 17 secondary -- --data-checksums
pg_ctlcluster 17 secondary start
```

---

## Part C: Starting, Stopping, and Managing the Service

### C.1 pg_ctl vs systemctl

`pg_ctl` operates directly on a `PGDATA` directory; `systemctl` is the standard service-manager wrapper most DBAs use day-to-day on a package install.

```bash
# Direct
pg_ctl -D /var/lib/postgresql/17/main start
pg_ctl -D /var/lib/postgresql/17/main status

# Via systemd (typical production usage)
sudo systemctl start postgresql
sudo systemctl enable postgresql   # start on boot
sudo systemctl status postgresql
```

### C.2 Shutdown Modes — a Frequently Misunderstood DBA Topic

| Mode | Behavior |
|---|---|
| `smart` | Waits for all clients to disconnect on their own before shutting down — can hang indefinitely if a session never closes |
| `fast` *(default for `pg_ctl stop`)* | Disconnects clients immediately, rolls back in-flight transactions cleanly, shuts down — the normal choice for planned maintenance |
| `immediate` | Aborts everything without a clean shutdown checkpoint — next startup **requires crash recovery** (Module 4). Emergency use only |

```bash
pg_ctl -D /var/lib/postgresql/17/main stop -m fast
```

### C.3 Reload vs. Restart

Not every config change needs downtime — this trips up a lot of new DBAs.

```sql
-- Ask PostgreSQL, per-parameter, whether a change needs a restart
SELECT name, context
FROM pg_settings
WHERE name IN ('shared_buffers', 'work_mem', 'listen_addresses', 'max_connections');
```

| `context` value | Meaning |
|---|---|
| `postmaster` | Requires a full **restart** (e.g., `shared_buffers`, `max_connections`, `listen_addresses`) |
| `sighup` | Just needs a **reload** — no downtime (e.g., `work_mem`, most logging settings) |
| `user`/`superuser` | Can be set per-session with `SET`, no config file edit needed at all |

```bash
# Reload without dropping connections
sudo systemctl reload postgresql
# or, from inside psql:
```
```sql
SELECT pg_reload_conf();
```

---

## Part D: postgresql.conf — Core Configuration

### D.1 File Locations

```sql
SHOW config_file;
SHOW hba_file;
SHOW data_directory;
```

### D.2 Parameter Categories Every DBA Must Know on Day One

| Category | Key parameters |
|---|---|
| Connections | `listen_addresses`, `port`, `max_connections` |
| Memory | `shared_buffers`, `work_mem`, `maintenance_work_mem`, `effective_cache_size` |
| WAL/Checkpoints | `wal_level`, `checkpoint_timeout`, `max_wal_size` *(full depth in Module 4)* |
| Logging | `logging_collector`, `log_destination`, `log_directory`, `log_line_prefix`, `log_min_duration_statement` |
| Autovacuum | `autovacuum`, `autovacuum_max_workers` *(full depth in Module 5)* |

```ini
# A representative starting point in postgresql.conf — not exhaustive
listen_addresses = 'localhost'        # tighten before opening to the network — see Part F
max_connections = 200
shared_buffers = 4GB                  # commonly ~25% of RAM as a starting point
effective_cache_size = 12GB           # commonly ~50-75% of RAM
work_mem = 32MB
maintenance_work_mem = 512MB
logging_collector = on
log_directory = 'log'
log_line_prefix = '%m [%p] %u@%d '
log_min_duration_statement = 500      # log any query slower than 500ms
```

### D.3 postgresql.auto.conf and ALTER SYSTEM

Anything set via `ALTER SYSTEM` is written to a separate file, `postgresql.auto.conf`, which is loaded *after* — and overrides — `postgresql.conf`. This matters because editing `postgresql.conf` by hand won't override a value someone previously set with `ALTER SYSTEM`.

```sql
ALTER SYSTEM SET work_mem = '64MB';
SELECT pg_reload_conf();

-- To undo an ALTER SYSTEM change:
ALTER SYSTEM RESET work_mem;
```

```bash
# See exactly what ALTER SYSTEM has written
cat /var/lib/postgresql/17/main/postgresql.auto.conf
```

### D.4 Modular Configuration with include Directives

For any real environment, splitting config into logical files is far easier to manage and version-control than one giant file.

```ini
# In postgresql.conf
include_dir 'conf.d'
```

```bash
# Then drop version-controlled snippets in, e.g.:
/etc/postgresql/17/main/conf.d/10-memory.conf
/etc/postgresql/17/main/conf.d/20-logging.conf
/etc/postgresql/17/main/conf.d/30-replication.conf
```

---

## Part E: pg_hba.conf — Client Authentication

### E.1 File Format

Each line is checked **top to bottom, first match wins** — a very common source of "why can't I connect" confusion when a broader rule earlier in the file shadows a more specific one added later.

```
# TYPE  DATABASE  USER      ADDRESS          METHOD
local   all       postgres                   peer
local   all       all                        scram-sha-256
host    all       all       127.0.0.1/32     scram-sha-256
host    all       all       ::1/128          scram-sha-256
host    salesdb   app_user  10.0.1.0/24      scram-sha-256
```

### E.2 Authentication Methods a DBA Will Actually Use

| Method | Use case |
|---|---|
| `peer` | Local Unix-socket connections matched to OS username — common for the `postgres` superuser locally |
| `scram-sha-256` | **Recommended** password authentication — strong, salted challenge-response |
| `md5` | Legacy password method; weaker than SCRAM, kept mainly for backward compatibility with old clients |
| `trust` | No authentication at all — never use outside a fully isolated local dev sandbox |
| `cert` | Client SSL certificate-based auth |
| `ldap` / `gss` | Enterprise directory integration |

```sql
-- Check what encryption new passwords will use
SHOW password_encryption;
```

```sql
-- Applying a change to pg_hba.conf only takes effect after a reload
SELECT pg_reload_conf();
```

### E.3 pg_ident.conf — Mapping OS Users to DB Roles

Used alongside `peer`/`ident` authentication when the OS username and the PostgreSQL role name don't match one-to-one.

```
# MAPNAME  SYSTEM-USERNAME  PG-USERNAME
etl_map    etl_service      etl_readonly
```

```
# referenced from pg_hba.conf like this:
local   salesdb   etl_readonly   peer map=etl_map
```

---

## Part F: Day-One DBA Hardening Checklist

This is the section trainees should walk away treating as a literal checklist for any new cluster they stand up:

1. **Set a strong `postgres` superuser password immediately** — don't leave install defaults.
   ```sql
   ALTER ROLE postgres WITH PASSWORD 'a_strong_generated_password';
   ```
2. **Don't leave `listen_addresses = '*'`** unless you actually need remote connections — and if you do, pair it with tight `pg_hba.conf` CIDR ranges, never `0.0.0.0/0`.
3. **Use `scram-sha-256`, not `md5` or `trust`**, for any password-based line in `pg_hba.conf`.
4. **Confirm data checksums are on** (`SHOW data_checksums;`) — automatic on PG18+, but verify explicitly on PG17 and earlier.
5. **Turn on meaningful logging from day one**, not after the first incident: `log_min_duration_statement`, `log_connections`, `log_disconnections`, and a `log_line_prefix` that includes PID, user, and database.
6. **Decide your connection strategy before you have a problem** — plan for a pooler (PgBouncer) if the application will open many short-lived connections, rather than just raising `max_connections` indefinitely.
7. **Enable `pg_stat_statements` immediately**, not after a performance issue forces the question:
   ```sql
   CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
   ```
8. **Review `pg_hba.conf` for leftover permissive lines** — many install defaults ship a broad local `trust`/`peer` entry that's fine for a single-admin dev box but should be tightened before anything resembling production use.
9. **Record your `initdb` choices** (encoding, locale, checksums) somewhere durable — they're effectively permanent for that cluster's lifetime.

---

## Discussion Questions / Exercises for Trainees

1. Install PostgreSQL from the PGDG repo (not the OS default repo) on a test VM, and confirm the version installed matches what you expected with `SELECT version();`.
2. Run `pg_ctl stop` with no `-m` flag, then look up which shutdown mode is actually the default in your installed version's docs — don't assume.
3. Change `work_mem` two ways — once via `SET` in a session, once via `ALTER SYSTEM` — and explain the scope and persistence difference between the two.
4. Deliberately misorder two lines in a test `pg_hba.conf` so a broad rule shadows a specific one, reload, and observe the effect on a connection attempt.
5. Walk through the Part F checklist against a cluster you set up yourself and note anything you'd change.

## Key Takeaways

- Install from the **PGDG repository**, not a distro's default packages, to get a current, DBA-controlled version.
- `initdb` decisions — **encoding, locale, and data checksums** — are effectively permanent for a cluster's lifetime; get them right up front. PostgreSQL 18+ finally defaults `--data-checksums` to on.
- Know your **shutdown modes** (`smart`/`fast`/`immediate`) — using `immediate` casually forces an unnecessary crash-recovery cycle.
- Check a parameter's `context` in `pg_settings` before assuming you need downtime — many changes only need a **reload**, not a restart.
- `pg_hba.conf` is evaluated **top to bottom, first match wins** — ordering mistakes are one of the most common real-world "why can't I connect" issues.
- Treat Part F as a literal pre-production checklist, not optional advice — password, listen address, auth method, checksums, logging, and `pg_stat_statements` should all be handled before a cluster goes anywhere near real traffic.
