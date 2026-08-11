# Module 2: Installation & Configuration
## Rocky Linux 9 Install | initdb | Service Management | postgresql.conf | pg_hba.conf | Day-One Hardening

*Datapundit — PostgreSQL DBA Training Series*

---

Module 1 covered the "why PostgreSQL" story. Before going anywhere near internals, a DBA needs to actually **stand up a cluster the right way** — because several decisions made at install time (locale, checksums, encoding) are difficult or impossible to change later, and several config choices made on day one (authentication method, logging, connection limits) are what you'll be firefighting at 2 AM if you get them wrong. This module is deliberately hands-on and checklist-driven, targeting **Rocky Linux 9** as the lab platform — it's the module trainees should be able to follow along with on a real VM or container.

---

## Part A: Installation on Rocky Linux 9

### A.1 Why Not the Default AppStream Package

Rocky Linux 9 ships a PostgreSQL module in its default AppStream repo, but — like most distros — it lags behind upstream releases. For real DBA work, install from the official **PGDG (PostgreSQL Global Development Group)** repository instead, which lets you pick your exact major version and tracks upstream directly.

```bash
# 1. Install the PGDG repo RPM for Enterprise Linux 9
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# 2. Disable the built-in AppStream postgresql module so it doesn't conflict
dnf -qy module disable postgresql

# 3. Install PostgreSQL 17 server + contrib utilities from PGDG
dnf install -y postgresql17-server postgresql17-contrib
```

**Trainer note:** This two-step "add PGDG repo, then disable the AppStream module" pattern is Rocky/RHEL-specific and easy to forget — skipping the `module disable` step is a common cause of `dnf` pulling in the wrong (older) package or throwing a conflict error.

### A.2 Where Things Land

RPM-based PGDG packages use a different layout than Debian's `apt` packages — worth memorizing since most online tutorials are written for one or the other:

| | Rocky/RHEL (PGDG rpm) | Debian/Ubuntu (PGDG apt) |
|---|---|---|
| Binaries | `/usr/pgsql-17/bin/` | `/usr/lib/postgresql/17/bin/` |
| Data directory | `/var/lib/pgsql/17/data/` | `/var/lib/postgresql/17/main/` |
| Service name | `postgresql-17` | `postgresql` |

```bash
# Confirm the binary path directly rather than assuming
which initdb || find / -name initdb 2>/dev/null
```

### A.3 Source Compilation

Rarely needed day-to-day, but a DBA should know when it applies: custom build flags, applying a specific patch before it's released, or building on an unsupported platform.

```bash
dnf install -y gcc make readline-devel zlib-devel openssl-devel
./configure --prefix=/usr/local/pgsql17 --with-openssl
make -j4
make install
```

### A.4 Containers

If you're running Rocky Linux 9 as a container image for this lab (common for a repeatable training environment), be aware of two things that differ from a full VM:

- **No `sudo` by default** on minimal images — if you're already `root` or already the `postgres` user (check your prompt), just run commands directly rather than prefixing `sudo`.
- **Locale packages are often missing** — covered in Part B.3 below, since it's the single most common first error trainees hit.

```bash
docker run -it --name pg17-lab rockylinux:9 bash
```

### A.5 Managed Cloud Services (Awareness, Not Hands-On Here)

Amazon RDS/Aurora, Google Cloud SQL, and Azure Database for PostgreSQL abstract away `initdb`, OS patching, and some config access — a DBA should know these exist and roughly what control is *lost* (no filesystem/superuser access, restricted `postgresql.conf` parameters) versus a self-managed install like the rest of this module covers.

---

## Part B: Initializing a Cluster — initdb

### B.1 What initdb Actually Does

`initdb` creates a brand-new, empty **cluster**: the `PGDATA` directory structure (from Module 3 — `base/`, `global/`, `pg_wal/`, etc.), the `template0`/`template1`/`postgres` databases, and the default configuration files.

### B.2 Two Ways to Run It on Rocky Linux 9

**Option 1 — the PGDG-provided setup wrapper** (recommended for a standard, single-cluster install — it handles directory ownership and permissions for you):

```bash
# Pass extra initdb flags (like data checksums or a specific locale) via this env var
PGSETUP_INITDB_OPTIONS="--data-checksums --locale=C.UTF-8" \
  /usr/pgsql-17/bin/postgresql-17-setup initdb
```

**Option 2 — calling `initdb` directly** (more control — needed for a second cluster on the same box, or a custom `PGDATA` location):

```bash
# As the postgres OS user (or directly, if you're already logged in as postgres --
# no "sudo" needed inside most minimal containers):
/usr/pgsql-17/bin/initdb \
  -D /var/lib/pgsql/17/data \
  --encoding=UTF8 \
  --locale=C.UTF-8 \
  --data-checksums \
  --auth-local=peer \
  --auth-host=scram-sha-256
```

**Trainer note:** If you're logged in as `root`, switch user properly first (`su - postgres`) rather than trying `sudo -u postgres`, since minimal container images frequently don't have `sudo` installed at all. If you're *already* the `postgres` user (check your shell prompt), skip `su`/`sudo` entirely and just run the binary directly — this is exactly the situation most trainees hit first in a container-based lab.

### B.3 The Locale Trap — A Very Common First Error on Rocky Linux 9

Minimal Rocky Linux 9 images frequently do **not** have `en_US.UTF-8` generated, which produces this exact error:

```
initdb: error: invalid locale name "en_US.UTF-8"
initdb: hint: If the locale name is specific to ICU, use --icu-locale.
```

Check what's actually available before assuming:

```bash
locale -a
```

Two fixes:

```bash
# Fix 1: install the missing locale package, then re-run initdb with --locale=en_US.UTF-8
dnf install -y glibc-langpack-en
locale -a | grep en_US

# Fix 2 (simpler, works on virtually every minimal image with no extra package):
# just use C.UTF-8 instead -- gives UTF-8 encoding with predictable, simple
# collation, which is often preferable for a lab/training environment anyway
/usr/pgsql-17/bin/initdb -D /var/lib/pgsql/17/data --encoding=UTF8 \
  --locale=C.UTF-8 --data-checksums --auth-local=peer --auth-host=scram-sha-256
```

### B.4 Decisions You Cannot Easily Change Later

| Decision | Why it matters | DBA guidance |
|---|---|---|
| **Encoding** (`--encoding`) | Set per-cluster at initdb time | `UTF8` unless you have a specific legacy reason not to |
| **Locale / Collation** (`--locale`, or `--locale-provider=icu`) | Affects sort order, `LIKE`, indexes on text columns | Pick deliberately; changing collation later can silently invalidate existing indexes |
| **Data checksums** (`--data-checksums`) | Detects silent storage-level corruption | **As of PostgreSQL 18, `initdb` enables checksums by default** — a genuinely significant change, since for years DBAs had to remember to pass `--data-checksums` explicitly. On PG 17 and earlier (including this lab), always pass it explicitly; there is essentially never a good reason to skip it on a production cluster |

```sql
-- Verify whether the running cluster has checksums on
SHOW data_checksums;
```

```bash
pg_checksums --pgdata=/var/lib/pgsql/17/data --enable
```

### B.5 Multiple Clusters on One Server

```bash
# Initialize a second cluster in a custom location on the same host
PGSETUP_INITDB_OPTIONS="--data-checksums" \
  /usr/pgsql-17/bin/initdb -D /var/lib/pgsql/17/secondary --data-checksums

# Run it on a different port to avoid colliding with the primary cluster
/usr/pgsql-17/bin/pg_ctl -D /var/lib/pgsql/17/secondary -o "-p 5433" start
```

---

## Part C: Starting, Stopping, and Managing the Service

### C.1 pg_ctl vs systemctl

`pg_ctl` operates directly on a `PGDATA` directory — the only option inside a bare container with no systemd running. `systemctl` is the standard service-manager wrapper on a full Rocky Linux 9 VM.

```bash
# Direct -- always works, including inside minimal containers
/usr/pgsql-17/bin/pg_ctl -D /var/lib/pgsql/17/data start
/usr/pgsql-17/bin/pg_ctl -D /var/lib/pgsql/17/data status

# Via systemd on a full VM -- note the version-specific service name on Rocky/RHEL
systemctl start postgresql-17
systemctl enable postgresql-17   # start on boot
systemctl status postgresql-17
```

### C.2 Shutdown Modes — a Frequently Misunderstood DBA Topic

| Mode | Behavior |
|---|---|
| `smart` | Waits for all clients to disconnect on their own before shutting down — can hang indefinitely if a session never closes |
| `fast` *(default for `pg_ctl stop`)* | Disconnects clients immediately, rolls back in-flight transactions cleanly, shuts down — the normal choice for planned maintenance |
| `immediate` | Aborts everything without a clean shutdown checkpoint — next startup **requires crash recovery** (Module 4). Emergency use only |

```bash
/usr/pgsql-17/bin/pg_ctl -D /var/lib/pgsql/17/data stop -m fast
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
systemctl reload postgresql-17
# or, directly:
/usr/pgsql-17/bin/pg_ctl -D /var/lib/pgsql/17/data reload
```
```sql
-- or, from inside psql:
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

On this lab setup, that's typically `/var/lib/pgsql/17/data/postgresql.conf` and `/var/lib/pgsql/17/data/pg_hba.conf` — both live directly inside `PGDATA` on a PGDG RPM install (unlike Debian's packages, which relocate config files to `/etc/postgresql/17/main/`).

### D.2 Parameter Categories Every DBA Must Know on Day One

| Category | Key parameters |
|---|---|
| Connections | `listen_addresses`, `port`, `max_connections` |
| Memory | `shared_buffers`, `work_mem`, `maintenance_work_mem`, `effective_cache_size` |
| WAL/Checkpoints | `wal_level`, `checkpoint_timeout`, `max_wal_size` *(full depth in Module 4)* |
| Logging | `logging_collector`, `log_destination`, `log_directory`, `log_line_prefix`, `log_min_duration_statement` |
| Autovacuum | `autovacuum`, `autovacuum_max_workers` *(full depth in Module 5)* |

```ini
# A representative starting point in postgresql.conf -- not exhaustive
listen_addresses = 'localhost'        # tighten before opening to the network -- see Part G
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
cat /var/lib/pgsql/17/data/postgresql.auto.conf
```

### D.4 Modular Configuration with include Directives

For any real environment, splitting config into logical files is far easier to manage and version-control than one giant file.

```ini
# In postgresql.conf
include_dir 'conf.d'
```

```bash
mkdir -p /var/lib/pgsql/17/data/conf.d
# Then drop version-controlled snippets in, e.g.:
# /var/lib/pgsql/17/data/conf.d/10-memory.conf
# /var/lib/pgsql/17/data/conf.d/20-logging.conf
# /var/lib/pgsql/17/data/conf.d/30-replication.conf
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

## Part F: Rocky Linux 9-Specific Gotchas — SELinux & firewalld

Two RHEL-family subsystems that don't exist on Debian/Ubuntu, and that catch DBAs off guard the first time they hit them:

### F.1 SELinux

Rocky Linux 9 ships with SELinux **enforcing by default**. Standard PGDG package paths (`/var/lib/pgsql/17/data`) already have the correct context — but a **custom `PGDATA` location** or non-default port will be silently blocked unless you fix the SELinux context.

```bash
# Check current enforcement mode
getenforce

# If you use a non-standard data directory, label it correctly instead of
# disabling SELinux outright:
semanage fcontext -a -t postgresql_db_t "/data/pgsql17(/.*)?"
restorecon -Rv /data/pgsql17

# If you change the listening port, tell SELinux about it too:
semanage port -a -t postgresql_port_t -p tcp 5433
```

**Trainer note:** "Just turn off SELinux" is a common shortcut in tutorials and should be explicitly discouraged in this training — it's a real security control in production RHEL-family environments, and `semanage`/`restorecon` solve the actual problem without disabling protection cluster-wide.

### F.2 firewalld

If PostgreSQL needs to accept connections from other hosts (not just `localhost`), the firewall needs an explicit rule — this is separate from, and in addition to, `listen_addresses` and `pg_hba.conf`.

```bash
firewall-cmd --permanent --add-service=postgresql
firewall-cmd --reload
firewall-cmd --list-services   # confirm it's active
```

---

## Part G: Day-One DBA Hardening Checklist

This is the section trainees should walk away treating as a literal checklist for any new cluster they stand up:

1. **Set a strong `postgres` superuser password immediately** — don't leave install defaults.
   ```sql
   ALTER ROLE postgres WITH PASSWORD 'a_strong_generated_password';
   ```
2. **Don't leave `listen_addresses = '*'`** unless you actually need remote connections — and if you do, pair it with tight `pg_hba.conf` CIDR ranges, never `0.0.0.0/0`, **and** an explicit `firewalld` rule (Part F.2).
3. **Use `scram-sha-256`, not `md5` or `trust`**, for any password-based line in `pg_hba.conf`.
4. **Confirm data checksums are on** (`SHOW data_checksums;`) — automatic on PG18+, but verify explicitly on PG17 and earlier.
5. **Turn on meaningful logging from day one**, not after the first incident: `log_min_duration_statement`, `log_connections`, `log_disconnections`, and a `log_line_prefix` that includes PID, user, and database.
6. **Decide your connection strategy before you have a problem** — plan for a pooler (PgBouncer) if the application will open many short-lived connections, rather than just raising `max_connections` indefinitely.
7. **Enable `pg_stat_statements` immediately**, not after a performance issue forces the question:
   ```sql
   CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
   ```
8. **Review `pg_hba.conf` for leftover permissive lines** — many install defaults ship a broad local `trust`/`peer` entry that's fine for a single-admin dev box but should be tightened before anything resembling production use.
9. **Handle SELinux and firewalld deliberately, don't disable them** — label custom paths/ports correctly (Part F.1) and open only the specific service/port needed (Part F.2).
10. **Record your `initdb` choices** (encoding, locale, checksums) somewhere durable — they're effectively permanent for that cluster's lifetime.

---

## Discussion Questions / Exercises for Trainees

1. Install PostgreSQL from the PGDG repo (not the AppStream default) on a Rocky Linux 9 VM or container, and confirm the version installed matches what you expected with `SELECT version();`.
2. Deliberately trigger the locale error from Part B.3 (or reproduce it if you already hit it), then fix it both ways — installing `glibc-langpack-en`, and separately using `C.UTF-8` — and discuss which you'd choose for a real production cluster and why.
3. Run `pg_ctl stop` with no `-m` flag, then look up which shutdown mode is actually the default in your installed version's docs — don't assume.
4. Change `work_mem` two ways — once via `SET` in a session, once via `ALTER SYSTEM` — and explain the scope and persistence difference between the two.
5. Deliberately misorder two lines in a test `pg_hba.conf` so a broad rule shadows a specific one, reload, and observe the effect on a connection attempt.
6. Move a cluster's data directory to a custom path and use `semanage`/`restorecon` to get it running under SELinux enforcing mode, rather than switching to permissive mode.
7. Walk through the Part G checklist against a cluster you set up yourself and note anything you'd change.

## Key Takeaways

- Install PostgreSQL on Rocky Linux 9 from the **PGDG repo** (`dnf install pgdg-redhat-repo...`, then `dnf module disable postgresql` before installing) — not the lagging AppStream default.
- RPM installs use **different paths** than Debian: binaries under `/usr/pgsql-17/bin/`, data under `/var/lib/pgsql/17/data/`, service name `postgresql-17`. Always verify with `which initdb` rather than assuming.
- The **locale trap** (`invalid locale name "en_US.UTF-8"`) is one of the most common first errors on minimal Rocky Linux 9 images — fix it with `glibc-langpack-en` or fall back to `C.UTF-8`.
- `initdb` decisions — **encoding, locale, and data checksums** — are effectively permanent for a cluster's lifetime; get them right up front. PostgreSQL 18+ finally defaults `--data-checksums` to on.
- Know your **shutdown modes** (`smart`/`fast`/`immediate`) — using `immediate` casually forces an unnecessary crash-recovery cycle.
- Check a parameter's `context` in `pg_settings` before assuming you need downtime — many changes only need a **reload**, not a restart.
- `pg_hba.conf` is evaluated **top to bottom, first match wins** — ordering mistakes are one of the most common real-world "why can't I connect" issues.
- **SELinux and firewalld** are Rocky/RHEL-specific layers on top of PostgreSQL's own auth — handle both deliberately (`semanage`/`restorecon`, `firewall-cmd`) rather than disabling them.
- Treat Part G as a literal pre-production checklist, not optional advice.
