# PostgreSQL DBA Training Curriculum & Hands-On Workbook: Backup and Restore

**Target Versions:** PostgreSQL 17–19 (with compatibility notes for PG 13–16)  
**Lab Environment:** Docker (`postgres:17` / `postgres:18` / `postgres:19` official images)

---

## 1. Core Theory & Architecture

### 1.1 Logical vs. Physical Backups
Backups in PostgreSQL fall into two architectural paradigms. Choosing between them depends on database size, Recovery Time Objective (RTO), cross-version portability, and granularity requirements.

| Dimension | Logical Backup (`pg_dump`, `pg_dumpall`) | Physical Backup (`pg_basebackup`, pgBackRest, Barman, WAL-G) |
| :--- | :--- | :--- |
| **Mechanism** | Connects via SQL (`libpq`), opens a `REPEATABLE READ` transaction, and exports DDL + DML/COPY data. | Copies raw binary files from `$PGDATA` (and tablespaces) at the OS/block level while recording WAL positions. |
| **Consistency Guarantee** | Transactionally consistent snapshot at the exact moment the transaction starts (MVCC). No WAL needed. | Fuzzy (crash-consistent) file copy during live writes; becomes consistent only after replaying WAL from `START WAL LOCATION` to `STOP WAL LOCATION` (defined in `backup_label`). |
| **Point-in-Time Recovery (PITR)** | **No.** Can only restore to the exact timestamp when the dump started. | **Yes.** Replays archived WAL segments to any timestamp, LSN, transaction ID, or named restore point. |
| **Restore Granularity** | High (single table, schema, function, or full database). | Cluster-level only (all databases in the instance must be restored together). |
| **Cross-Version / OS Portability** | **High.** Can migrate from PG 13 to PG 18 or Linux to Windows. | **Strict.** Must match exact major version, CPU architecture, and block size. |
| **Performance at Scale** | Slow on multi-TB databases (CPU-bound serialization and index rebuilds on restore). | Fast (raw I/O streaming, block-level incrementals, no index rebuild required on restore). |

### 1.2 Write-Ahead Logging (WAL) & Continuous Archiving
PostgreSQL guarantees durability using Write-Ahead Logging: every change to data pages in `shared_buffers` is written to WAL buffers and flushed to `pg_wal/` *before* the corresponding dirty data page is flushed to disk.

* **`wal_level`:**
  * `minimal`: Strips information needed for replication/archiving. Cannot be used with `archive_mode` or `pg_basebackup`.
  * `replica` (Default): Logs sufficient information for physical replication, base backups, and PITR.
  * `logical`: Adds tuple-level identity metadata for logical decoding/replication.
* **Checkpoints & `full_page_writes`:**
  * During a checkpoint, dirty buffers are flushed to disk.
  * Because an OS crash or physical backup copy might read an 8 KB PostgreSQL page mid-write (a "torn page"), `full_page_writes = on` (default) forces PostgreSQL to write the entire 8 KB page image into WAL the first time a page is modified after each checkpoint.
* **Log Sequence Numbers (LSNs):** A 64-bit integer printed as two 32-bit hex values (`X/YYYYYYYY`, e.g., `0/3000148`) representing a byte offset in the WAL stream. Every data page header stores the LSN of the last WAL record that modified it (`pd_lsn`). During recovery, PostgreSQL skips any WAL record whose LSN is less than or equal to the page's `pd_lsn` (idempotency).
* **Continuous Archiving (`archive_mode` & `archive_command` / `archive_library`):**
  * When a 16 MB WAL segment fills (or `archive_timeout` expires), PostgreSQL marks it `.ready` in `pg_wal/archive_status/` and invokes `archive_command` (or a C-based `archive_library` introduced in PG 15+ to avoid fork/exec overhead).
  * PostgreSQL will **not** delete or recycle a WAL segment from `pg_wal/` until `archive_command` exits with return code `0`.

> **CRITICAL DBA WARNING (`archive_mode` & Disk Full):**  
> If your `archive_command` fails continuously (e.g., backup storage is full or unreachable), PostgreSQL retains unarchived WAL files inside `pg_wal/`. Left unchecked, `pg_wal/` will fill the root/data filesystem and cause an immediate database PANIC/shutdown. Always monitor `pg_stat_archiver.failed_count` and alert on `pg_wal/` directory growth.

### 1.3 Point-in-Time Recovery (PITR) & Timelines
When PostgreSQL starts with a `recovery.signal` file present in `$PGDATA`, it enters **Archive Recovery** mode:
1. Reads `backup_label` to find the starting checkpoint LSN (`START WAL LOCATION`).
2. Fetches required WAL segments from the archive using `restore_command`.
3. Replays WAL records forward until it reaches the configured recovery target:
   * `recovery_target_time`: Stop at a specific timestamp (e.g., `'2026-11-25 08:30:00+05:30'`).
   * `recovery_target_lsn`: Stop at an exact LSN.
   * `recovery_target_xid`: Stop at a specific transaction ID.
   * `recovery_target_name`: Stop at a named restore point created via `SELECT pg_create_restore_point('pre_deploy');`.
   * `recovery_target_inclusive`: `true` (default, includes the target transaction) or `false` (stops right before the offending transaction).
   * `recovery_target_action`: What to do upon reaching the target—`pause` (default, allows read-only inspection before promoting), `promote` (immediately open for read/write), or `shutdown`.

**Why Timelines Exist:**  
Suppose your database is on **Timeline 1** (`00000001...`). At 10:00 AM, someone drops a table, and WAL continues up to 10:15 AM on Timeline 1. You restore a backup and perform PITR to 09:59 AM, then promote the server. If the newly promoted server continued writing to Timeline 1, its new WAL segments would collide with and overwrite the original 10:00–10:15 AM WAL files in your archive.  
To prevent history collisions, PostgreSQL increments the timeline ID to **Timeline 2** (`00000002...`) upon promotion and writes a `.history` file (`00000002.history`) recording the exact LSN and parent timeline where the branch occurred.

### 1.4 Backup Types & Verification
* **Full Backup:** Complete copy of `$PGDATA` at a point in time.
* **Differential Backup:** Copies all blocks/files changed since the last **Full** backup (supported by pgBackRest and Barman).
* **Incremental Backup:**
  * *Tool-level (pgBackRest / WAL-G):* Copies blocks/files changed since the last backup of **any type** (full, diff, or prior incremental).
  * *Native PostgreSQL (PG 17+):* Uses the background **WAL Summarizer** process (`summarize_wal = on`), which writes compact block-modification summaries into `pg_wal/summaries/`. `pg_basebackup --incremental=/path/to/previous/backup_manifest` queries the server for only the modified blocks since that manifest's LSN. Before starting PostgreSQL, `pg_combinebackup` reconstructs a synthetic full backup from the base + incremental chain.
* **Page Checksums & `pg_verifybackup`:**
  * Initializing a cluster with `initdb --data-checksums` (or enabling offline via `pg_checksums --enable`) adds a 16-bit FNV-1a checksum to every 8 KB data page header.
  * Every physical backup generated by `pg_basebackup` or `pg_combinebackup` includes a `backup_manifest` JSON file listing every file, its size, and its cryptographic checksum (default `CRC32C`, configurable to `SHA256`/`SHA512`). `pg_verifybackup` validates files on disk against this manifest and parses required WAL to guarantee the backup is restorable before an emergency occurs.

### 1.5 RPO & RTO Planning
* **RPO (Recovery Point Objective):** Maximum tolerable data loss measured in time.
  * *Daily `pg_dump` only:* RPO = 24 hours.
  * *Physical backup + `archive_command` (`archive_timeout = 60s`):* RPO at most 60 seconds.
  * *Physical backup + synchronous `pg_receivewal` / Barman streaming:* RPO near zero (zero committed transaction loss).
* **RTO (Recovery Time Objective):** Maximum tolerable downtime from incident declaration to full service restoration. Driven by backup restore throughput, network bandwidth from object storage, and volume of WAL replay since the last base/incremental backup.

---

## 2. Native PostgreSQL Tools (Theory + Practical Steps)

### 2.1 `pg_dump` & `pg_dumpall`
`pg_dump` backs up a single database; `pg_dumpall` dumps cluster-wide globals (roles, tablespaces) and iterates through all databases.

**Format Comparison:**
* **Plain (`-Fp`, default):** Plain SQL script. Restored with `psql`. Cannot be parallelized or selectively restored via `pg_restore`.
* **Custom (`-Fc`):** Compressed binary archive supporting `pg_restore` parallel jobs (`-j`) and selective object restoration. Best default for single databases.
* **Directory (`-Fd`):** Creates a directory with one compressed file per table plus `toc.dat`. **Required** if you want parallel *dumping* (`pg_dump -Fd -j 4`).
* **Tar (`-Ft`):** Uncompressed tar archive; lacks parallel restore support and cannot modify compression dynamically.

**Practical Commands:**
```bash
# 1. Dump cluster-wide globals ONLY (roles, passwords, tablespaces)
pg_dumpall -h localhost -U postgres --globals-only -f /backups/globals.sql

# 2. Custom format dump of a single database (zstd compression supported in PG 16+)
pg_dump -h localhost -U postgres -d appdb -Fc --compress=zstd:3 -f /backups/appdb.dump

# 3. Parallel directory dump (4 worker connections; requires repeatable-read snapshot export)
pg_dump -h localhost -U postgres -d appdb -Fd -j 4 -f /backups/appdb_dir/
```

**DBA Gotchas:**
* **Locking:** `pg_dump` acquires an `ACCESS SHARE` lock on every table it dumps. This blocks `ALTER TABLE`, `DROP TABLE`, `TRUNCATE`, and `VACUUM FULL` while the dump runs.
* **Large Objects & Extensions:** Extensions are dumped as `CREATE EXTENSION ...` statements—the underlying shared libraries (`.so`) must already be installed on the target server.

### 2.2 `pg_restore`
```bash
# 1. Parallel restore into a newly created database (disable triggers/constraints handled automatically)
createdb -h localhost -U postgres appdb_restored
pg_restore -h localhost -U postgres -d appdb_restored -j 4 --verbose /backups/appdb.dump

# 2. Selective restore of a single table (data + indexes/constraints for that table)
pg_restore -h localhost -U postgres -d appdb_restored -t orders /backups/appdb.dump

# 3. Two-step TOC (Table of Contents) filtered restore
pg_restore -l /backups/appdb.dump > /tmp/toc.list
# Edit /tmp/toc.list and comment out unwanted items with ';'
pg_restore -h localhost -U postgres -d appdb_restored -L /tmp/toc.list /backups/appdb.dump
```

### 2.3 `pg_basebackup` & Continuous WAL Archiving (PITR)

**Step 1: Configure PostgreSQL (`postgresql.conf`)**
```ini
wal_level = replica
archive_mode = on
# Secure, atomic copy: only succeeds if destination file doesn't already exist
archive_command = 'test ! -f /backups/wal_archive/%f && cp %p /backups/wal_archive/%f'
archive_timeout = 300   # Force WAL switch every 5 mins even on low-traffic systems
```
*(Note: Changing `archive_mode` from `off` to `on` requires a PostgreSQL restart; changing `archive_command` only requires `pg_ctl reload` / `SELECT pg_reload_conf();`.)*

**Step 2: Take a Base Backup**
```bash
# -X stream: opens a second replication connection to stream WAL generated during the backup
# -c fast: forces an immediate checkpoint instead of waiting for checkpoint_timeout
pg_basebackup -h localhost -U postgres -D /backups/base_full_01 \
  -Fp -X stream -c fast -P -v

# Verify backup integrity immediately
pg_verifybackup -w /backups/base_full_01
```

**Step 3: Perform a PITR Restore**
```bash
# 1. Stop PostgreSQL and preserve/clear the corrupted $PGDATA
pg_ctl -D /var/lib/postgresql/data stop

# 2. Copy base backup into $PGDATA (preserving ownership 0700)
rm -rf /var/lib/postgresql/data/*
cp -a /backups/base_full_01/. /var/lib/postgresql/data/
chmod 0700 /var/lib/postgresql/data

# 3. Configure recovery parameters in postgresql.auto.conf (or postgresql.conf)
cat << 'EOF' >> /var/lib/postgresql/data/postgresql.auto.conf
restore_command = 'cp /backups/wal_archive/%f %p'
recovery_target_time = '2026-11-25 08:15:00+05:30'
recovery_target_inclusive = false
recovery_target_action = 'pause'
EOF

# 4. Signal PostgreSQL to enter archive recovery
touch /var/lib/postgresql/data/recovery.signal

# 5. Start PostgreSQL and watch logs until recovery pauses at target
pg_ctl -D /var/lib/postgresql/data start
```

### 2.4 Native Incremental Backups (PG 17–19)
PostgreSQL 17 introduced native block-level incremental backups via WAL summarization and `pg_combinebackup` (also standard in PG 18 and PG 19).

**Step 1: Enable WAL Summarization**
```sql
ALTER SYSTEM SET summarize_wal = on;
SELECT pg_reload_conf();

-- Verify summarizer is running and advancing
SELECT * FROM pg_stat_get_wal_summarizer();
```

**Step 2: Take Full + Incremental Chain**
```bash
# Day 0: Full Backup
pg_basebackup -U postgres -D /backups/inc_chain/full_day0 -c fast -P

# Day 1: Incremental Backup referencing Day 0's manifest
pg_basebackup -U postgres -D /backups/inc_chain/incr_day1 \
  --incremental=/backups/inc_chain/full_day0/backup_manifest -c fast -P

# Day 2: Incremental Backup referencing Day 1's manifest
pg_basebackup -U postgres -D /backups/inc_chain/incr_day2 \
  --incremental=/backups/inc_chain/incr_day1/backup_manifest -c fast -P
```

**Step 3: Reconstruct & Verify Before Restore**
```bash
# Combine backups in strict chronological order (Full -> Incr1 -> Incr2)
# Use --clone (XFS/Btrfs reflinks) or --copy-file-range on modern Linux for near-instant merging
pg_combinebackup --copy-file-range \
  /backups/inc_chain/full_day0 \
  /backups/inc_chain/incr_day1 \
  /backups/inc_chain/incr_day2 \
  -o /backups/inc_chain/reconstructed_full

# Validate the reconstructed synthetic full backup
pg_verifybackup /backups/inc_chain/reconstructed_full
```

---

## 3. Open-Source Enterprise Backup & Restore Tools

While native tools work well for smaller setups, production enterprise environments rely on dedicated orchestrators for retention management, parallel compression, encryption, and S3/GCS/Azure object storage.

### 3.1 Tool Architecture Comparison

| Feature | pgBackRest | Barman | WAL-G |
| :--- | :--- | :--- | :--- |
| **Architecture** | Push/Pull agent model via SSH or TLS (`pgbackrest server`); single binary in C. | Centralized backup server (pull-based via SSH + streaming replication protocol). | Cloud-native CLI binary written in Go; stateless execution directly against object storage. |
| **WAL Shipping** | `archive-push` / `archive-get` with asynchronous parallel queues (`archive-async=y`). | `barman receive-wal` (zero-RPO streaming) and/or `archive_command`. | `wal-push` / `wal-fetch` with prefetching for fast parallel restore. |
| **Backup Types** | Full, Differential, Incremental (file & block-level). | Full, Incremental (file-level hardlinks or block-level). | Full, Delta backups (page-level deltas via WAL or file scanning). |
| **Compression & Encryption** | `lz4`, `zstd`, `gzip`, `bzip2`; native AES-256-CBC repo encryption. | `gzip`, `bzip2`, `lz4`, `zstd`, `pigz`; GPG encryption support. | `lz4`, `lzma`, `zstd`, `brotli`; libsodium / PGP / AWS KMS encryption. |
| **Storage Backends** | POSIX local/NFS, S3, GCS, Azure Blob (multi-repo simultaneous support). | POSIX local/NFS, S3/GCS/Azure (via `barman-cloud-*` utilities). | S3, GCS, Azure, Swift, SSH, Local FS. |

### 3.2 pgBackRest (Theory + Practical Steps)
pgBackRest uses **stanzas** (configuration profiles mapping a PostgreSQL cluster to a backup repository). It checks page checksums during backup and automatically expires old backups and unneeded WAL archives.

**Configuration (`/etc/pgbackrest/pgbackrest.conf`):**
```ini
[global]
repo1-type=s3
repo1-s3-bucket=prod-pg-backups
repo1-s3-endpoint=s3.ap-south-1.amazonaws.com
repo1-s3-region=ap-south-1
repo1-s3-key=AKIA...
repo1-s3-key-secret=...
repo1-path=/pgbackrest
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass=StrongRepoPassphraseHere
repo1-retention-full=2
repo1-retention-diff=7
compress-type=zst
compress-level=3
process-max=4
archive-async=y
spool-path=/var/spool/pgbackrest

[main_cluster]
pg1-path=/var/lib/postgresql/data
pg1-port=5432
```

**PostgreSQL `postgresql.conf`:**
```ini
archive_mode = on
archive_command = 'pgbackrest --stanza=main_cluster archive-push %p'
```

**Operations:**
```bash
# Initialize stanza and validate end-to-end connectivity & WAL archiving
pgbackrest --stanza=main_cluster stanza-create
pgbackrest --stanza=main_cluster check

# Take Full, Differential, and Incremental backups
pgbackrest --stanza=main_cluster --type=full backup
pgbackrest --stanza=main_cluster --type=diff backup
pgbackrest --stanza=main_cluster --type=incr backup

# Inspect repository inventory, sizes, and LSN ranges
pgbackrest --stanza=main_cluster info

# PITR Restore (stop Postgres first; --delta overwrites only changed files in $PGDATA)
pgbackrest --stanza=main_cluster --delta \
  --type=time "--target=2026-11-25 08:15:00+05:30" \
  --target-action=promote restore
```

### 3.3 Barman (Backup and Recovery Manager)
Barman runs on a dedicated backup server and supports **streaming WAL reception** (`pg_receivewal` under the hood) so WAL records arrive at the backup server milliseconds after commit, achieving near-zero RPO without waiting for a 16 MB segment to fill.

**Configuration (`/etc/barman.d/pg-prod.conf`):**
```ini
[pg-prod]
description = "Production PostgreSQL Cluster"
conninfo = host=10.0.1.10 user=barman dbname=postgres
streaming_conninfo = host=10.0.1.10 user=streaming_barman
backup_method = postgres
streaming_archiver = on
slot_name = barman_slot
create_slot = auto
archiver = on
retention_policy = RECOVERY WINDOW OF 14 DAYS
```

**Operations:**
```bash
# Start background WAL streamer and run maintenance cron
barman receive-wal --create-slot pg-prod
barman cron
barman check pg-prod

# Execute full backup and list backups
barman backup pg-prod
barman list-backup pg-prod

# Remote PITR restore over SSH to a target host
barman recover --remote-ssh-command "ssh postgres@10.0.1.11" \
  --target-time "2026-11-25 08:15:00+05:30" \
  pg-prod latest /var/lib/postgresql/data
```

### 3.4 WAL-G
WAL-G excels in containerized and cloud-native environments where there is no persistent sidecar backup host—only environment variables pointing to S3/GCS/Azure.

**Environment & `postgresql.conf`:**
```bash
export WALG_S3_PREFIX="s3://prod-pg-walg/cluster01"
export AWS_REGION="ap-south-1"
export WALG_COMPRESSION_METHOD="zstd"
export WALG_LIBSODIUM_KEY="0123456789abcdef0123456789abcdef"
export WALG_DELTA_MAX_STEPS=6
export PGDATA="/var/lib/postgresql/data"
```
```ini
archive_mode = on
archive_command = '/usr/local/bin/wal-g wal-push %p'
```

**Operations:**
```bash
# Push base/delta backup
wal-g backup-push $PGDATA
wal-g backup-list

# Verify WAL continuity and backup integrity
wal-g wal-verify integrity timeline

# Fetch latest backup and configure PITR
wal-g backup-fetch $PGDATA LATEST
echo "restore_command = '/usr/local/bin/wal-g wal-fetch %f %p'" >> $PGDATA/postgresql.auto.conf
echo "recovery_target_time = '2026-11-25 08:15:00+05:30'" >> $PGDATA/postgresql.auto.conf
touch $PGDATA/recovery.signal
```

---

## 4. Common Backup & Restore Scenarios (Playbooks)

### Scenario 1: Accidental `DROP TABLE` / `DELETE` / Bad Application Deployment
* **Diagnosis:** Application alerts on missing table or wiped rows. Production database is still online and serving writes to other tables.
* **Decision Criteria:** Do **not** restore over the live primary. Doing so destroys valid transactions committed on unrelated tables after the incident. Instead, spin up an **isolated clone instance** (e.g., on a temporary host or container), perform PITR to 1 second before the bad transaction, export the missing table with `pg_dump`, and import it back into production.
* **Playbook:**
  1. Pinpoint the exact timestamp or transaction ID (`xid`) using application logs or `pg_waldump` on recent WAL segments:
     ```bash
     pg_waldump -p /backups/wal_archive 000000010000000000000005 | grep -i "DROP\|DELETE"
     ```
  2. Restore your latest backup onto a **separate recovery instance/container** with `archive_mode = off` (critical so the clone never pushes WAL to your prod archive).
  3. Set `recovery_target_time = '<timestamp just before mistake>'` (or `recovery_target_xid = '<xid>'` with `recovery_target_inclusive = false`) and `recovery_target_action = 'promote'`.
  4. Verify row counts on the recovered clone, then surgically migrate the data back:
     ```bash
     pg_dump -h clone-host -U postgres -d appdb -t dropped_table -Fc -f /tmp/recovered_table.dump
     pg_restore -h prod-host -U postgres -d appdb /tmp/recovered_table.dump
     ```

### Scenario 2: Data File Corruption or Missing/Corrupt Pages (`data_checksums` alerts)
* **Diagnosis:** Logs report `ERROR: invalid page in block 412 of relation base/16384/24590` or `WARNING: page verification failed, calculated checksum ...`.
* **Decision Criteria:**
  * If a physical streaming standby exists and its page is clean (checksum error caused by local disk bit-rot on primary), **fail over to the standby immediately** and rebuild the old primary.
  * If corruption propagated or no standby exists, map the `relfilenode` (`24590`) to its database object:
    ```sql
    SELECT oid::regclass, relkind FROM pg_class WHERE pg_relation_filenode(oid) = 24590;
    ```
  * If `relkind = 'i'` (an index), fix it online with zero data loss: `REINDEX INDEX CONCURRENTLY <index_name>;`.
  * If `relkind = 'r'` (a heap table), perform a full cluster PITR from your last verified clean backup + WAL archive onto clean storage.

### Scenario 3: Complete Primary Host / Storage Loss
* **Diagnosis:** Underlying VM, bare-metal server, or EBS/persistent volume is unrecoverable.
* **Playbook:**
  1. Provision new host/container with identical PostgreSQL major version, extensions, and filesystem permissions (`0700` on `$PGDATA`).
  2. Restore the latest base backup (`pg_basebackup` + `pg_combinebackup`, `pgbackrest restore`, or `wal-g backup-fetch`).
  3. Configure `restore_command` and `recovery_target = 'latest'` with `recovery_target_action = 'promote'`, create `recovery.signal`, and start PostgreSQL.
  4. Once promoted to primary, verify `pg_is_in_recovery()` returns `false`, take a fresh full backup on the new timeline, and rebuild downstream read replicas.

### Scenario 4: Multi-Terabyte Database with Tight Backup Windows
* **Diagnosis:** A 6 TB database takes 9 hours for a full `pg_basebackup`, exceeding the nightly maintenance window and saturating disk I/O.
* **Mitigation Playbook:**
  * **Move backups off the primary:** Run `pg_basebackup` or `pgbackrest` (`--backup-standby=y`) against a physical standby replica so primary CPU/disk I/O is untouched.
  * **Use parallel block-level incrementals:** Switch to weekly full + daily incremental backups (`pgbackrest --process-max=8` or PG 17+ `pg_basebackup --incremental` with `summarize_wal = on`).
  * **Use storage-level atomic snapshots (ZFS / LVM / AWS EBS / GCP PD):** Call `SELECT pg_backup_start('snap_01', true);`, trigger instantaneous copy-on-write volume snapshots across `$PGDATA` and tablespaces simultaneously, then run `SELECT * FROM pg_backup_stop(true);` and save the returned `backup_label` content alongside the snapshot.

### Scenario 5: Cross-Major-Version Upgrade or Schema/Environment Migration
* **Diagnosis:** Upgrading from PostgreSQL 15 to PostgreSQL 18, or migrating from on-prem Linux to managed cloud PostgreSQL.
* **Decision Criteria:**
  * **Sub-100 GB or Cloud Managed Target:** Dump globals (`pg_dumpall --globals-only`) + parallel directory dump (`pg_dump -Fd -j 8`) using the **target (newer)** version's `pg_dump` binary, then `pg_restore -j 8` and run `ANALYZE` immediately afterward (optimizer statistics are not included in logical dumps).
  * **Multi-TB On-Prem / Self-Hosted:** Physical backups cannot cross major versions. Use `pg_upgrade --link` (near-instant hardlink upgrade) or Logical Replication (`CREATE PUBLICATION` / `CREATE SUBSCRIPTION`) for near-zero-downtime migration. Always take a verified physical backup of the old cluster before cutting over.

### Scenario 6: Recovery Across Timelines & Failed PITR Attempts
* **Diagnosis:** You performed a PITR to `10:05:00`, promoted the server (creating Timeline 2), and realized the bad transaction actually happened at `10:02:00`. Now you need to rewind back to `10:01:00`.
* **Playbook:**
  1. Stop PostgreSQL, wipe `$PGDATA`, and restore the base backup again.
  2. Set `recovery_target_timeline = '1'` (or `'latest'` if branching off Timeline 2 at an earlier point) and `recovery_target_time = '10:01:00'`.
  3. Re-run recovery; upon promotion, PostgreSQL will create **Timeline 3** (`00000003.history`).

> **CRITICAL DBA WARNING (`archive_mode` During Drills & PITR):**  
> Never start a restored test cluster with `archive_mode = on` pointing to your **production** WAL archive path or S3 bucket. If a test cluster promotes and starts archiving new WAL segments or `.history` files into production storage, it can corrupt your production recovery chain. Set `archive_mode = off` in `postgresql.auto.conf` before starting any drill or forensic clone.

---

## 5. Hands-On Lab Exercises & Assessment (Docker Environment)

All labs below run on a single host using Docker. Create a clean workspace directory first:
```bash
mkdir -p ~/pg_backup_labs/{pgdata,pgdata_restore,wal_archive,backups}
docker network create pg_lab_net
```

### Lab 1: Logical Backup, Selective Table Restore & Globals
**Objective:** Configure a PostgreSQL 18 container, populate data, dump globals + custom archive, simulate a dropped table, and restore only the lost table.

1. **Setup:**
   ```bash
   docker run -d --name pg_primary --network pg_lab_net \
     -e POSTGRES_PASSWORD=postgres \
     -v ~/pg_backup_labs/backups:/backups \
     postgres:18

   # Create a custom role, database, and two tables
   docker exec -it pg_primary psql -U postgres -c "CREATE ROLE app_user LOGIN PASSWORD 'secret';"
   docker exec -it pg_primary psql -U postgres -c "CREATE DATABASE shopdb OWNER app_user;"
   docker exec -it pg_primary psql -U postgres -d shopdb -c "
     CREATE TABLE customers (id serial PRIMARY KEY, name text);
     INSERT INTO customers (name) VALUES ('Alice'), ('Bob');
     CREATE TABLE audit_log (id serial PRIMARY KEY, event text);
     INSERT INTO audit_log (event) VALUES ('login_ok');"
   ```
2. **Backup:**
   ```bash
   docker exec -it pg_primary pg_dumpall -U postgres --globals-only -f /backups/globals.sql
   docker exec -it pg_primary pg_dump -U postgres -d shopdb -Fc -f /backups/shopdb.dump
   ```
3. **Simulate Failure:**
   ```bash
   docker exec -it pg_primary psql -U postgres -d shopdb -c "DROP TABLE customers;"
   ```
4. **Recover & Verify:**
   ```bash
   docker exec -it pg_primary pg_restore -U postgres -d shopdb -t customers /backups/shopdb.dump
   docker exec -it pg_primary psql -U postgres -d shopdb -c "SELECT * FROM customers;"
   ```

---

### Lab 2: Physical Base Backup & Point-in-Time Recovery (PITR)
**Objective:** Enable continuous WAL archiving, capture a base backup, record a restore timestamp, execute a destructive `DELETE`, and recover to the exact moment prior to data loss.

1. **Setup with WAL Archiving:**
   ```bash
   docker rm -f pg_primary
   rm -rf ~/pg_backup_labs/pgdata/* ~/pg_backup_labs/wal_archive/*

   docker run -d --name pg_primary --network pg_lab_net \
     -e POSTGRES_PASSWORD=postgres \
     -v ~/pg_backup_labs/pgdata:/var/lib/postgresql/data \
     -v ~/pg_backup_labs/wal_archive:/wal_archive \
     -v ~/pg_backup_labs/backups:/backups \
     postgres:18 \
     -c wal_level=replica \
     -c archive_mode=on \
     -c archive_command='test ! -f /wal_archive/%f && cp %p /wal_archive/%f'
   ```
2. **Create Baseline Data & Take Base Backup:**
   ```bash
   docker exec -it pg_primary psql -U postgres -c "
     CREATE TABLE ledger (id serial PRIMARY KEY, amount numeric, created_at timestamptz DEFAULT now());
     INSERT INTO ledger (amount) SELECT generate_series(1, 100) * 10;"

   docker exec -it pg_primary pg_basebackup -U postgres -D /backups/base_pitr -Fp -X stream -c fast -P
   docker exec -it pg_primary pg_verifybackup /backups/base_pitr
   ```
3. **Record Target Timestamp & Simulate Accidental `DELETE`:**
   ```bash
   # Add valid post-backup transactions
   docker exec -it pg_primary psql -U postgres -c "INSERT INTO ledger (amount) VALUES (9999);"
   # Capture the exact timestamp right BEFORE the mistake
   docker exec -it pg_primary psql -U postgres -t -c "SELECT now();"
   # -> Copy this timestamp (e.g., 2026-11-25 03:20:15.123456+00)

   sleep 2
   # Oops! Accidental DELETE without WHERE clause
   docker exec -it pg_primary psql -U postgres -c "DELETE FROM ledger;"
   # Force WAL switch so the DELETE and prior commit are archived
   docker exec -it pg_primary psql -U postgres -c "SELECT pg_switch_wal();"
   ```
4. **Execute PITR into a Recovery Container:**
   ```bash
   rm -rf ~/pg_backup_labs/pgdata_restore/*
   cp -a ~/pg_backup_labs/backups/base_pitr/. ~/pg_backup_labs/pgdata_restore/

   # Configure recovery target and disable archiving on the recovery instance
   cat << 'EOF' >> ~/pg_backup_labs/pgdata_restore/postgresql.auto.conf
   archive_mode = off
   restore_command = 'cp /wal_archive/%f %p'
   recovery_target_time = 'PASTE_YOUR_TIMESTAMP_HERE'
   recovery_target_action = 'promote'
   EOF
   touch ~/pg_backup_labs/pgdata_restore/recovery.signal

   docker run -d --name pg_recovered --network pg_lab_net \
     -v ~/pg_backup_labs/pgdata_restore:/var/lib/postgresql/data \
     -v ~/pg_backup_labs/wal_archive:/wal_archive \
     postgres:18
   ```
5. **Post-Mortem Verification:**
   ```bash
   # Verify 101 rows exist (including 9999) and timeline incremented to 2
   docker exec -it pg_recovered psql -U postgres -c "SELECT count(*), max(amount) FROM ledger;"
   docker exec -it pg_recovered psql -U postgres -c "SELECT timeline_id FROM pg_control_checkpoint();"
   ```

---

### Lab 3: Native PG 17–19 Incremental Backups (`summarize_wal` + `pg_combinebackup`)
**Objective:** Use PostgreSQL's native WAL summarizer to take a full backup followed by two block-level incremental backups, merge them with `pg_combinebackup`, and validate the synthetic full backup.

1. **Enable WAL Summarization on `pg_primary`:**
   ```bash
   docker exec -it pg_primary psql -U postgres -c "ALTER SYSTEM SET summarize_wal = on;"
   docker exec -it pg_primary psql -U postgres -c "SELECT pg_reload_conf();"
   ```
2. **Execute Full -> Incr1 -> Incr2 Chain:**
   ```bash
   mkdir -p ~/pg_backup_labs/backups/native_inc

   # Base Full Backup
   docker exec -it pg_primary pg_basebackup -U postgres -D /backups/native_inc/full -c fast -P

   # Modify data & take Incremental 1
   docker exec -it pg_primary psql -U postgres -c "CREATE TABLE inc_test AS SELECT generate_series(1,50000) AS id;"
   docker exec -it pg_primary pg_basebackup -U postgres \
     --incremental=/backups/native_inc/full/backup_manifest \
     -D /backups/native_inc/inc1 -c fast -P

   # Modify more data & take Incremental 2
   docker exec -it pg_primary psql -U postgres -c "INSERT INTO inc_test SELECT generate_series(50001,100000);"
   docker exec -it pg_primary pg_basebackup -U postgres \
     --incremental=/backups/native_inc/inc1/backup_manifest \
     -D /backups/native_inc/inc2 -c fast -P
   ```
3. **Compare Sizes, Reconstruct & Verify:**
   ```bash
   du -sh ~/pg_backup_labs/backups/native_inc/*

   docker exec -it pg_primary pg_combinebackup \
     /backups/native_inc/full \
     /backups/native_inc/inc1 \
     /backups/native_inc/inc2 \
     -o /backups/native_inc/merged_full

   docker exec -it pg_primary pg_verifybackup /backups/native_inc/merged_full
   ```

---

### Lab 4: Enterprise Backup & PITR with pgBackRest in Docker
**Objective:** Configure `pgbackrest` inside a custom container image, initialize a stanza, run full + incremental backups, and execute a `--delta` PITR restore.

1. **Build a Quick PG 18 + pgBackRest Image:**
   ```bash
   cat << 'EOF' > ~/pg_backup_labs/Dockerfile.pgbackrest
   FROM postgres:18
   RUN apt-get update && apt-get install -y pgbackrest && rm -rf /var/lib/apt/lists/*
   RUN mkdir -p /var/lib/pgbackrest /var/log/pgbackrest /etc/pgbackrest && \
       chown -R postgres:postgres /var/lib/pgbackrest /var/log/pgbackrest /etc/pgbackrest
   EOF

   docker build -t pg18-backrest -f ~/pg_backup_labs/Dockerfile.pgbackrest ~/pg_backup_labs
   ```
2. **Configure `pgbackrest.conf` & Start Container:**
   ```bash
   cat << 'EOF' > ~/pg_backup_labs/pgbackrest.conf
   [global]
   repo1-path=/var/lib/pgbackrest
   repo1-retention-full=2
   compress-type=zst
   process-max=2

   [demo]
   pg1-path=/var/lib/postgresql/data
   EOF

   docker run -d --name pg_backrest_lab \
     -e POSTGRES_PASSWORD=postgres \
     -v ~/pg_backup_labs/pgbackrest.conf:/etc/pgbackrest/pgbackrest.conf \
     pg18-backrest \
     -c wal_level=replica \
     -c archive_mode=on \
     -c archive_command='pgbackrest --stanza=demo archive-push %p'
   ```
3. **Initialize Stanza, Backup, Drop Data & Restore:**
   ```bash
   docker exec -u postgres pg_backrest_lab pgbackrest --stanza=demo stanza-create
   docker exec -u postgres pg_backrest_lab pgbackrest --stanza=demo check
   docker exec -u postgres pg_backrest_lab pgbackrest --stanza=demo --type=full backup

   # Create data and take incremental backup
   docker exec -u postgres pg_backrest_lab psql -c "CREATE TABLE critical_data (id int); INSERT INTO critical_data VALUES (1), (2), (3);"
   docker exec -u postgres pg_backrest_lab pgbackrest --stanza=demo --type=incr backup
   docker exec -u postgres pg_backrest_lab pgbackrest --stanza=demo info

   # Simulate catastrophic DROP TABLE
   docker exec -u postgres pg_backrest_lab psql -c "DROP TABLE critical_data;"

   # Stop PostgreSQL inside container, restore with --delta, and restart
   docker exec -u postgres pg_backrest_lab pg_ctl -D /var/lib/postgresql/data stop
   docker exec -u postgres pg_backrest_lab pgbackrest --stanza=demo --delta restore
   docker exec -u postgres pg_backrest_lab pg_ctl -D /var/lib/postgresql/data start

   # Verify recovery
   docker exec -u postgres pg_backrest_lab psql -c "SELECT * FROM critical_data;"
   ```

*(Cleanup when finished: `docker rm -f pg_primary pg_recovered pg_backrest_lab && docker network rm pg_lab_net`)*

---

### DBA Knowledge Check & Troubleshooting Quiz

**1. Why does `pg_basebackup` require WAL replay even if no queries ran while the backup was executing?**  
**2. Your `pg_basebackup --incremental` command fails with `ERROR: manifest requires WAL from ... which has already been removed` (or missing WAL summaries). What happened, and how do you fix it?**  
**3. During a PITR drill on a staging server, PostgreSQL stops at `recovery_target_time` and accepts read-only queries, but all `INSERT` statements fail with `cannot execute INSERT in a read-only transaction`. Why?**  
**4. You have a 2 TB database and want to use `pg_dump` with `-j 8` for parallel dumping. Which dump format flag (`-Fp`, `-Fc`, `-Fd`, `-Ft`) must you use, and why?**  
**5. Why does `pg_restore` not restore user roles or tablespaces even when you pass `--create`?**

---

#### Answers & Explanations
1. **Crash Consistency & `backup_label`:** A physical backup is an online file-system copy taken while PostgreSQL's background writer/checkpointer may be modifying files. `backup_label` records the exact starting checkpoint LSN; PostgreSQL must replay all WAL generated between the start and end of the backup to bring all data files to a single consistent point in time.
2. **WAL Summaries Purged (`wal_summary_keep_time`):** The WAL summarizer only retains summary files for `wal_summary_keep_time` (default `10d`). If an incremental backup references a manifest older than the retained summaries (or if `summarize_wal` was toggled off), PostgreSQL cannot determine changed blocks. Fix: Take a fresh full base backup and adjust `wal_summary_keep_time` if needed.
3. **`recovery_target_action = 'pause'`:** By default, `recovery_target_action` is set to `pause` so the DBA can inspect the database before committing to a timeline branch. Run `SELECT pg_wal_replay_resume();` (or set `recovery_target_action = 'promote'`) to end recovery and promote the node to read-write.
4. **Directory Format (`-Fd`):** Parallel `pg_dump` requires multiple worker processes writing separate table files concurrently into a directory structure (`toc.dat` + per-table compressed files). Single-file formats (`-Fc`, `-Fp`, `-Ft`) cannot be written concurrently by multiple dump workers.
5. **Database-Scoped vs. Cluster-Scoped Objects:** `pg_dump` and `pg_restore` operate on a single database (`pg_database`), whereas roles (`pg_authid`) and tablespaces (`pg_tablespace`) are shared across the entire PostgreSQL cluster. You must always back those up separately using `pg_dumpall --globals-only`.