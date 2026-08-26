# Module 6: Backup & Recovery
## Logical vs. Physical Backups | pg_dump/pg_basebackup | Point-in-Time Recovery | Verification | Strategy

*Datapundit — PostgreSQL DBA Training Series*

---

Module 4's crash recovery protects against a server crash — WAL replay gets a running cluster back to a consistent state automatically. It does **not** protect against a dropped table, a bad migration, ransomware, or a failed disk. That's what this module covers: deliberately keeping separate, restorable copies of your data, and knowing exactly how to get back to any point in time when something goes wrong.

---

## Part A: Why "WAL Already Protects Me" Isn't Enough

| Scenario | Does crash recovery (Module 4) help? |
|---|---|
| Server loses power mid-write | ✅ Yes — automatic WAL replay |
| `DROP TABLE orders;` run by mistake at 2 PM | ❌ No — that's now a durable, correctly-replicated fact |
| Disk hardware failure | ❌ No — the WAL and the data lived on the same failed disk |
| Ransomware encrypts `PGDATA` | ❌ No — every copy on that host is compromised |

A real backup strategy needs data that lives **somewhere else**, taken **on a schedule**, that can be **restored to a chosen point in time** — which is exactly what Part E (PITR) builds toward.

---

## Part B: Backup Types — The Big Picture

| | **Logical backup** (`pg_dump`) | **Physical backup** (`pg_basebackup`) |
|---|---|---|
| What it captures | SQL statements to recreate objects & data | Byte-for-byte copy of `PGDATA` |
| Portability | Cross-version, cross-platform, even cross-architecture | Same major version, same platform generally required |
| Granularity | Can back up a single table/schema/database | Whole cluster only |
| Restore speed | Slower (replays SQL, rebuilds indexes) | Faster (just copies files back) |
| PITR support | No, on its own | Yes, combined with WAL archiving |
| Typical use | Migrations, single-database/table recovery, version upgrades | Full-cluster disaster recovery, standby provisioning |

**Trainer note:** Most production strategies use **both** — physical backups + WAL archiving as the primary DR mechanism, and periodic logical dumps as an extra safety net that's easy to restore selectively (e.g., "just get me last Tuesday's `orders` table into a scratch database").

---

## Part C: Logical Backups — pg_dump / pg_dumpall

### C.1 pg_dump — Per-Database

```bash
# Custom format (recommended default): compressed, supports selective restore
pg_dump -U postgres -d salesdb -F c -f salesdb.dump

# Plain SQL text — human-readable, pipeable straight into psql
pg_dump -U postgres -d salesdb -F p -f salesdb.sql

# Directory format — supports parallel dump/restore, one file per table
pg_dump -U postgres -d salesdb -F d -j 4 -f salesdb_dir/
```

### C.2 Selective Backups

```bash
# Just one table
pg_dump -U postgres -d salesdb -t orders -F c -f orders_only.dump

# Just one schema
pg_dump -U postgres -d salesdb -n reporting -F c -f reporting_schema.dump

# Schema only, no data — useful for spinning up an empty structure
pg_dump -U postgres -d salesdb --schema-only -f schema_only.sql
```

### C.3 pg_dumpall — Cluster-Wide Objects

`pg_dump` operates **per database** and does not capture roles, tablespaces, or other cluster-level objects. `pg_dumpall` fills that gap.

```bash
# Just the globals (roles, tablespaces) -- commonly paired with per-DB pg_dump backups
pg_dumpall -U postgres --globals-only -f globals.sql

# Or a full cluster-wide plain-text dump (rarely used for large clusters -- prefer
# pg_dump per-database + globals-only for anything beyond a small cluster)
pg_dumpall -U postgres -f full_cluster.sql
```

### C.4 Restoring

```bash
# From custom/directory format -- pg_restore, supports parallel jobs and selective restore
pg_restore -U postgres -d salesdb_restored -j 4 salesdb.dump

# Restore just one table out of a full custom-format dump
pg_restore -U postgres -d salesdb_restored -t orders salesdb.dump

# Plain SQL format restores via psql directly
psql -U postgres -d salesdb_restored -f salesdb.sql
```

---

## Part D: Physical Backups — pg_basebackup

### D.1 A Full Base Backup

`pg_basebackup` takes a consistent, full copy of the running cluster's data directory over a replication connection — no downtime required.

```bash
pg_basebackup -U replicator -h primary_host \
  -D /backups/full_$(date +%Y%m%d) \
  -F tar -z \
  -X stream \
  -c fast \
  -P
```

| Flag | Meaning |
|---|---|
| `-F tar -z` | Tar format, gzip-compressed (vs. `plain`, an actual directory copy) |
| `-X stream` | Stream WAL alongside the backup so it's self-contained and consistent |
| `-c fast` | Force an immediate checkpoint to start (vs. `spread`, which is gentler on I/O but slower to start) |
| `-P` | Show progress |

### D.2 Incremental Backups (Native Since PostgreSQL 17)

Prior to PG17, every `pg_basebackup` was a full copy. PostgreSQL 17 introduced **native incremental backup**: given a previous backup's manifest, it copies only the blocks that changed since then.

```ini
# Required in postgresql.conf on the source server
summarize_wal = on
```

```bash
# Step 1: a full backup (as above), which produces a backup_manifest file
pg_basebackup -D /backups/full/ -F plain -c fast -X stream -P

# Step 2: an incremental backup, referencing the previous manifest
pg_basebackup --incremental=/backups/full/backup_manifest \
  -D /backups/incr1/ -F plain -c fast -X stream -P

# Step 3 (restore time): reconstruct a complete, restorable directory
pg_combinebackup /backups/full/ /backups/incr1/ -o /restore/combined/
```

**Important constraints:** incremental backups must be taken **from the primary**, not a standby, and the backup chain must be applied in strict order — `pg_combinebackup` will error out if a backup is missing or out of sequence.

---

## Part E: Point-in-Time Recovery (PITR)

### E.1 The Ingredients

PITR combines a physical base backup with the WAL archive built in Module 4, Part F — letting you restore not just to "when the backup was taken" but to **any moment covered by the archived WAL** (e.g., "roll forward to 2:59 PM, one minute before the accidental `DROP TABLE`").

Prerequisites, tying directly back to Module 4:

```ini
# On the source server, already covered in Module 4
archive_mode = on
archive_command = 'cp %p /mnt/wal_archive/%f'
wal_level = replica
```

### E.2 The Restore Procedure

```bash
# 1. Stop the target server (if running) and clear/replace its data directory
pg_ctl -D /var/lib/postgresql/17/main stop -m fast
rm -rf /var/lib/postgresql/17/main/*

# 2. Restore the base backup into place
tar -xzf /backups/full_20260801.tar.gz -C /var/lib/postgresql/17/main/

# 3. Create a recovery signal file -- this tells PostgreSQL to enter recovery mode
touch /var/lib/postgresql/17/main/recovery.signal
```

```ini
# 4. In postgresql.conf (or postgresql.auto.conf), point at the WAL archive
# and specify how far to recover
restore_command = 'cp /mnt/wal_archive/%f %p'
recovery_target_time = '2026-08-01 14:59:00'
```

```bash
# 5. Start the server -- it will replay archived WAL up to the target, then
# stop being "in recovery" and open for connections
pg_ctl -D /var/lib/postgresql/17/main start
```

### E.3 Recovery Target Options

| Parameter | Recovers to... |
|---|---|
| `recovery_target_time` | A specific timestamp |
| `recovery_target_xid` | Immediately after a specific transaction ID commits |
| `recovery_target_lsn` | A specific WAL LSN |
| `recovery_target_name` | A named restore point (set in advance with `pg_create_restore_point()`) |

```sql
-- Create a named, easy-to-target checkpoint before a risky deployment
SELECT pg_create_restore_point('before_migration_v42');
```

**Trainer note:** `recovery.conf` was **removed as a separate file in PostgreSQL 12** — recovery settings now live in `postgresql.conf`/`postgresql.auto.conf`, and the presence of `recovery.signal` (or `standby.signal` for a standby server) is what tells PostgreSQL to enter recovery mode at all. This trips up anyone following an old pre-PG12 tutorial.

---

## Part F: Backup Verification

**An unverified backup is not a backup — it's an assumption.** This deserves to be said explicitly to trainees, because it's the step most often skipped under time pressure.

```bash
# pg_verifybackup checks a backup's completeness and checksum integrity
# against its manifest, without needing a full restore
pg_verifybackup /backups/full_20260801/
```

The only *fully* trustworthy verification, though, is a **periodic real restore drill** — spin up a scratch instance from the backup, confirm the application can actually query it, and time how long the process takes (this feeds directly into RTO planning in Part H).

---

## Part G: Production-Grade Tooling (Awareness)

Hand-rolled `pg_basebackup`/`pg_dump` scripts work, but most production environments lean on purpose-built backup managers that add retention policies, parallelism, cloud storage offload, and automated PITR orchestration:

| Tool | Notable strengths |
|---|---|
| **pgBackRest** | Widely used, incremental/differential backups, parallel compression, direct S3/Azure/GCS support |
| **Barman** | Mature, strong PITR tooling, popular in EDB/2ndQuadrant-influenced shops |
| **WAL-G** | Lightweight, cloud-storage-native, popular in containerized/Kubernetes environments |

A DBA doesn't need to master all three, but should know they exist and roughly why a team might choose one over hand-rolled scripts — mainly retention/rotation management and built-in restore orchestration.

---

## Part H: Backup Strategy — RPO, RTO, and Retention

### H.1 Two Numbers That Should Drive Every Decision Above

- **RPO (Recovery Point Objective):** how much data can you afford to lose? (*"we can tolerate losing up to 5 minutes"* → you need continuous WAL archiving, not just nightly dumps)
- **RTO (Recovery Time Objective):** how fast must you be back up? (*"under 30 minutes"* → favor physical backups + PITR over a slow logical restore on a huge database)

### H.2 The 3-2-1 Rule

- **3** copies of your data (production + 2 backups)
- **2** different storage media/locations
- **1** copy off-site (a different region/data center than production)

### H.3 A Representative Retention Policy

```
Daily incremental physical backups  -> retain 7 days
Weekly full physical backup          -> retain 4 weeks
Monthly full physical backup         -> retain 12 months
WAL archive                          -> retain at least as long as your longest RPO window
Logical pg_dump (per-database)       -> retain 7-14 days, for quick selective restores
```

### H.4 Automate, but Monitor the Automation

```bash
# A cron entry is not a backup strategy by itself -- alert on failure
0 1 * * * pg_basebackup ... || curl -X POST https://alerts.example.com/backup-failed
```

```sql
-- Track backup-related activity from inside the database too
SELECT * FROM pg_stat_progress_basebackup;
```

---

## Discussion Questions / Exercises for Trainees

1. A colleague says "we don't need backups, WAL gives us crash recovery." Using Part A, explain exactly what that claim gets wrong.
2. Take a `pg_dump -F c` backup of a small test database, drop a table, and restore just that one table using `pg_restore -t`.
3. Walk through a full PITR exercise on a test VM: base backup → simulate a bad `DELETE` → restore to a `recovery_target_time` just before the mistake.
4. Explain, in your own words, why incremental backups (Part D.2) must always be taken from the primary and applied in strict chain order.
5. For a database your team manages (or a hypothetical one), write out its RPO and RTO, then check whether the current backup approach actually satisfies both.

## Key Takeaways

- Crash recovery (Module 4) and backups solve **different problems** — WAL protects against a crash; backups protect against data loss, corruption, and disasters that WAL replication would faithfully preserve.
- **Logical backups** (`pg_dump`/`pg_dumpall`) are portable and selective but slower to restore; **physical backups** (`pg_basebackup`) are fast to restore and enable PITR but are less flexible. Most production strategies use both.
- **PostgreSQL 17's native incremental backup** (`pg_basebackup --incremental` + `pg_combinebackup`) meaningfully reduces backup time/storage for large databases — but only from the primary, in strict chain order.
- **PITR** combines a base backup with the WAL archive to restore to any covered point in time; recovery is driven by `recovery.signal` + `restore_command` + a `recovery_target_*` setting, not a separate `recovery.conf` (removed since PG12).
- **An unverified backup isn't a backup** — `pg_verifybackup` checks integrity, but a real periodic restore drill is the only test that actually proves you can recover.
- Let **RPO and RTO** drive every backup decision, and build retention/automation around the 3-2-1 rule rather than an ad hoc cron job nobody's watching.
