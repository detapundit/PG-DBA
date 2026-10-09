# PostgreSQL 17 — User & Role Management: A DBA's Complete Guide

**Target version:** PostgreSQL 17.x
**Audience:** Database administrators (new to intermediate-advanced)
**Scope:** Roles, authentication, privileges, ownership, hardening, delegation, auditing, runbooks, troubleshooting.

> **Conventions used in this guide**
> - `-- as: postgres` above a block means "run as that role" (usually a superuser).
> - Passwords shown (`'ChangeMe_#1'`) are **placeholders**. In real life use `\password` (see §4.2) or a secrets manager.
> - Lab objects are prefixed so they're easy to clean up (§22).
> - Commands were written against the PostgreSQL 17 documentation; always test in a non-production lab first.

---

## Table of Contents

1. [Prerequisites and Lab Setup](#1-prerequisites-and-lab-setup)
2. [Core Concepts](#2-core-concepts)
3. [Creating Roles (Users and Groups)](#3-creating-roles-users-and-groups)
4. [Passwords and Authentication](#4-passwords-and-authentication)
5. [Altering, Renaming and Dropping Roles](#5-altering-renaming-and-dropping-roles)
6. [Role Membership: INHERIT, SET, ADMIN](#6-role-membership-inherit-set-admin)
7. [Predefined Roles](#7-predefined-roles)
8. [Privileges: GRANT and REVOKE](#8-privileges-grant-and-revoke)
9. [Default Privileges](#9-default-privileges)
10. [Ownership Management](#10-ownership-management)
11. [Database, Schema and Function Hardening](#11-database-schema-and-function-hardening)
12. [Row-Level Security (RLS)](#12-row-level-security-rls)
13. [Per-Role Settings, Limits and Login Controls](#13-per-role-settings-limits-and-login-controls)
14. [Real-World Role Blueprints (RBAC Design)](#14-real-world-role-blueprints-rbac-design)
15. [Session Management](#15-session-management)
16. [Auditing and Inventory Queries](#16-auditing-and-inventory-queries)
17. [Backup, Restore and Migration of Roles](#17-backup-restore-and-migration-of-roles)
18. [Operational Runbooks](#18-operational-runbooks)
19. [Troubleshooting Common Errors](#19-troubleshooting-common-errors)
20. [Best-Practice Checklist](#20-best-practice-checklist)
21. [Cheat Sheet](#21-cheat-sheet)
22. [Lab Cleanup](#22-lab-cleanup)

---

## 1. Prerequisites and Lab Setup

### 1.1 What you need

| Item | Requirement |
|---|---|
| PostgreSQL | 17.x server (a throwaway lab instance) |
| OS access | Shell on the DB host (for `pg_hba.conf`, `pg_ctl`, `psql`) or Docker |
| Superuser | A superuser login (`postgres` by default) |
| Client | `psql` 17 (older clients work but `\drg` needs psql 16+) |
| Optional | `openssl` (certs), `pgaudit` extension, `pg_stat_statements` |

### 1.2 Installing PostgreSQL 17

**Option A — Docker (fastest lab)**

```bash
docker run -d --name pg17 \
  -e POSTGRES_PASSWORD='ChangeMe_#1' \
  -p 5432:5432 \
  postgres:17

docker exec -it pg17 psql -U postgres
```

**Option B — Ubuntu / Debian (PGDG repository)**

```bash
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh   # adds PGDG repo
sudo apt install -y postgresql-17
sudo systemctl enable --now postgresql@17-main
sudo -u postgres psql
```

**Option C — RHEL / Rocky / Alma (PGDG repository)**

```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql17-server postgresql17-contrib
sudo /usr/pgsql-17/bin/postgresql-17-setup initdb
sudo systemctl enable --now postgresql-17
sudo -u postgres psql
```

### 1.3 Verify version and superuser access

```sql
-- as: postgres
SELECT version();
SHOW server_version;
SELECT current_user, session_user, usesuper FROM pg_user WHERE usename = current_user;
```

### 1.4 Locate the configuration files

```sql
-- as: postgres
SHOW config_file;
SHOW hba_file;
SHOW ident_file;
SHOW data_directory;
SHOW password_encryption;
SHOW log_directory;
```

Typical paths:

| Platform | `pg_hba.conf` |
|---|---|
| Debian/Ubuntu | `/etc/postgresql/17/main/pg_hba.conf` |
| RHEL family | `/var/lib/pgsql/17/data/pg_hba.conf` |
| Docker image | `/var/lib/postgresql/data/pg_hba.conf` |

Reload configuration after editing (does **not** drop connections):

```sql
SELECT pg_reload_conf();
```
```bash
# or from the shell
sudo -u postgres pg_ctl reload -D /var/lib/pgsql/17/data
sudo systemctl reload postgresql-17
```

### 1.5 psql meta-commands you will use constantly

| Command | Purpose |
|---|---|
| `\du[+]` / `\dg[+]` | List roles and attributes |
| `\drg` | List role **memberships** with grant options (psql 16+) |
| `\l[+]` | List databases (owner, ACL) |
| `\dn[+]` | List schemas (owner, ACL) |
| `\dp` / `\z` | Table/view/sequence privileges (ACL) |
| `\ddp` | Default privileges (`ALTER DEFAULT PRIVILEGES`) |
| `\df+` | Functions with owner/ACL |
| `\drds` | Per-role / per-database settings |
| `\dconfig` | Server parameters (psql 15+) |
| `\password [role]` | Change a password safely (hashed client-side) |
| `\c db role` | Reconnect as another role/database |
| `\conninfo` | Show current connection details |
| `\x` | Toggle expanded output |

### 1.6 Sample data used throughout this guide

This builds a database `appdb` owned by a **non-login owner role**, three schemas, and sample rows. Every later example assumes it.

```sql
-- as: postgres
CREATE ROLE appdb_owner NOLOGIN;
CREATE DATABASE appdb OWNER appdb_owner ENCODING 'UTF8';

\c appdb
```

```sql
-- as: postgres, connected to appdb
-- We create objects *as* the owner role so that ownership is clean.
SET ROLE appdb_owner;

CREATE SCHEMA sales;
CREATE SCHEMA hr;
CREATE SCHEMA reporting;

CREATE TABLE sales.customers (
    customer_id  serial PRIMARY KEY,
    name         text        NOT NULL,
    email        text        NOT NULL UNIQUE,
    region       text        NOT NULL,
    created_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE sales.orders (
    order_id     bigserial PRIMARY KEY,
    customer_id  int           NOT NULL REFERENCES sales.customers,
    order_date   date          NOT NULL DEFAULT current_date,
    amount       numeric(12,2) NOT NULL,
    status       text          NOT NULL DEFAULT 'NEW'
);

CREATE TABLE hr.employees (
    employee_id  serial PRIMARY KEY,
    full_name    text          NOT NULL,
    department   text          NOT NULL,
    salary       numeric(12,2) NOT NULL,   -- sensitive
    ssn_last4    char(4),                  -- sensitive
    db_user      name                      -- used in RLS demo
);

INSERT INTO sales.customers (name, email, region)
SELECT 'Customer ' || g,
       'cust' || g || '@example.com',
       (ARRAY['SOUTH','NORTH','EAST','WEST'])[1 + g % 4]
FROM generate_series(1, 20) AS g;

INSERT INTO sales.orders (customer_id, amount, status)
SELECT 1 + (g % 20),
       round((random() * 5000)::numeric, 2),
       (ARRAY['NEW','PAID','SHIPPED'])[1 + g % 3]
FROM generate_series(1, 200) AS g;

INSERT INTO hr.employees (full_name, department, salary, ssn_last4, db_user) VALUES
 ('Alice Rao',   'Analytics',   95000, '1111', 'alice'),
 ('Bob Shetty',  'Engineering', 120000,'2222', 'bob'),
 ('Carol Nayak', 'IT-Ops',      130000,'3333', 'carol'),
 ('Dev Kamath',  'Finance',     88000, '4444', 'dev');

CREATE VIEW reporting.order_summary AS
SELECT c.region, count(*) AS orders, sum(o.amount) AS revenue
FROM sales.orders o JOIN sales.customers c USING (customer_id)
GROUP BY c.region;

CREATE FUNCTION reporting.revenue_for(p_region text) RETURNS numeric
LANGUAGE sql STABLE AS $$
    SELECT coalesce(sum(o.amount), 0)
    FROM sales.orders o JOIN sales.customers c USING (customer_id)
    WHERE c.region = p_region
$$;

RESET ROLE;
```

Sanity check:

```sql
SELECT count(*) FROM sales.customers;   -- 20
SELECT count(*) FROM sales.orders;      -- 200
\dt sales.*
\dn+
```

---

## 2. Core Concepts

### 2.1 Roles, users and groups are the same thing

- A **role** is the only identity object in PostgreSQL. A "user" is a role that has `LOGIN`; a "group" is a role (usually `NOLOGIN`) that other roles are members of.
- `CREATE USER` ≡ `CREATE ROLE ... LOGIN`. `CREATE GROUP` ≡ `CREATE ROLE` (legacy alias).
- Roles are **cluster-wide** (shared across all databases in the instance). Their *privileges* are per-object and therefore per-database.

### 2.2 The access-check layers (in order)

```
 Client ──► 1. pg_hba.conf        (can this host/user/db even attempt to connect, and how to authenticate?)
        ──► 2. Role attributes    (LOGIN? connection limit? VALID UNTIL? password correct?)
        ──► 3. Database CONNECT   (GRANT CONNECT ON DATABASE)
        ──► 4. Schema USAGE       (GRANT USAGE ON SCHEMA)
        ──► 5. Object privileges  (SELECT/INSERT/EXECUTE/...)
        ──► 6. Column privileges  (optional narrowing)
        ──► 7. Row-Level Security (policies filter rows)
```

A failure at any layer denies access. Superusers skip layers 3–7 entirely (and RLS).

### 2.3 System catalogs for roles

| Catalog / view | What it holds | Who can read |
|---|---|---|
| `pg_roles` | Role attributes (password masked as `********`) | Everyone |
| `pg_authid` | Same + `rolpassword` (hash) | Superuser only |
| `pg_user` | Roles with `LOGIN`, legacy view | Everyone |
| `pg_shadow` | Login roles + password hash | Superuser only |
| `pg_group` | Legacy group view | Everyone |
| `pg_auth_members` | Role membership + `admin_option`, `inherit_option`, `set_option` (16+) | Everyone |
| `pg_db_role_setting` | Per-role / per-db GUC settings | Everyone |
| `pg_default_acl` | Default privileges | Everyone |
| `pg_parameter_acl` | Privileges on server parameters (15+) | Everyone |
| `pg_shdepend` | Shared dependencies (what objects a role owns/has privileges on) | Everyone |
| `pg_hba_file_rules` | Parsed `pg_hba.conf` (with errors) | Superuser / `pg_read_all_settings` |
| `pg_ident_file_mappings` | Parsed `pg_ident.conf` (15+) | Superuser / `pg_read_all_settings` |
| `information_schema.role_table_grants` etc. | SQL-standard views of privileges | Limited to roles visible to you |

### 2.4 What changed in versions 14 → 17 (things DBAs must know)

| Version | Change |
|---|---|
| **14** | `scram-sha-256` becomes default `password_encryption`; new predefined roles `pg_read_all_data`, `pg_write_all_data`; `pg_database_owner` |
| **15** | `CREATE` on schema `public` **revoked from PUBLIC** (new clusters); `pg_checkpoint` role; **privileges on server parameters** (`GRANT SET/ALTER SYSTEM ON PARAMETER`); `pg_ident_file_mappings` view |
| **16** | **Big CREATEROLE rework**: a CREATEROLE role can only manage roles it has `ADMIN` on; membership grants gain `INHERIT`/`SET`/`ADMIN` options per grant; `GRANTED BY`; `createrole_self_grant`; `pg_hba.conf` regex + `include` directives; `\drg`; `pg_create_subscription`, `pg_use_reserved_connections` + `reserved_connections`; `scram_iterations`; libpq `require_auth` |
| **17** | **`MAINTAIN` privilege** and **`pg_maintain`** role (VACUUM/ANALYZE/CLUSTER/REFRESH MATERIALIZED VIEW/REINDEX/LOCK TABLE without ownership); **login event triggers** + `event_triggers` GUC; `allow_alter_system` GUC; maintenance operations run with a safe `search_path`; `db_user_namespace` removed |
| 18 (preview) | MD5 password auth deprecated — **migrate to SCRAM now** |

---

## 3. Creating Roles (Users and Groups)

### 3.1 Syntax (PostgreSQL 17)

```sql
CREATE ROLE name [ [ WITH ] option [ ... ] ]

where option can be:
      SUPERUSER   | NOSUPERUSER
    | CREATEDB    | NOCREATEDB
    | CREATEROLE  | NOCREATEROLE
    | INHERIT     | NOINHERIT
    | LOGIN       | NOLOGIN
    | REPLICATION | NOREPLICATION
    | BYPASSRLS   | NOBYPASSRLS
    | CONNECTION LIMIT connlimit
    | [ ENCRYPTED ] PASSWORD 'password' | PASSWORD NULL
    | VALID UNTIL 'timestamp'
    | IN ROLE role_name [, ...]      -- new role becomes a member of these roles
    | ROLE role_name [, ...]         -- these roles become members of the new role
    | ADMIN role_name [, ...]        -- like ROLE, but with ADMIN OPTION
    | SYSID uid                      -- ignored, legacy
```

### 3.2 Attribute reference

| Attribute | Default | Meaning | DBA notes |
|---|---|---|---|
| `LOGIN` | off (`CREATE ROLE`), on (`CREATE USER`) | Role may start a session | Groups should be `NOLOGIN` |
| `SUPERUSER` | off | Bypasses **all** permission checks (incl. RLS) | Only a superuser can grant it. Keep to a minimum of named DBA accounts |
| `CREATEDB` | off | May `CREATE DATABASE` (owns what it creates) | Fine for dev/CI roles |
| `CREATEROLE` | off | May create roles and manage roles it has ADMIN on (16+) | Safer than before, still powerful — see §6.4 |
| `INHERIT` | on | Default for membership grants where this role is the **member**: whether it automatically uses the granted role's privileges | Per-grant `INHERIT` option overrides (16+) |
| `REPLICATION` | off | May start streaming replication / `pg_basebackup` / logical slots | Dedicated roles only |
| `BYPASSRLS` | off | Bypasses row-level security policies | Needed for `pg_dump` of RLS tables when not superuser |
| `CONNECTION LIMIT n` | -1 (unlimited) | Max concurrent sessions for the role | Not enforced for superusers |
| `PASSWORD 'x'` | none | Password (stored hashed per `password_encryption`) | `PASSWORD NULL` = no password |
| `VALID UNTIL 'ts'` | infinity | Password expiry (affects password auth only) | Does **not** block cert/peer/LDAP auth |

> Privilege-escalation rule: you can only grant/set an attribute you yourself hold (and `SUPERUSER`, `REPLICATION`, `BYPASSRLS` can only be set by a superuser, or — for the latter two — by a CREATEROLE role that has them, in 16+).

### 3.3 Examples

**A. Simple login user (human)**

```sql
-- as: postgres
CREATE ROLE alice WITH LOGIN PASSWORD 'ChangeMe_#1';
-- equivalent:
-- CREATE USER alice PASSWORD 'ChangeMe_#1';
```

**B. Group role (no login)**

```sql
CREATE ROLE app_readonly  NOLOGIN;
CREATE ROLE app_readwrite NOLOGIN;
```

**C. User created directly as a member of a group**

```sql
CREATE ROLE bob LOGIN PASSWORD 'ChangeMe_#1' IN ROLE app_readonly;
```

**D. Group created with existing members**

```sql
CREATE ROLE data_team NOLOGIN ROLE alice, bob;            -- alice, bob become members
CREATE ROLE data_leads NOLOGIN ADMIN alice;               -- alice is member WITH ADMIN OPTION
```

**E. Service account with limits and expiry**

```sql
CREATE ROLE svc_orders
    LOGIN
    PASSWORD 'ChangeMe_#1'
    CONNECTION LIMIT 50
    VALID UNTIL '2027-03-31 23:59:59+05:30';
```

**F. Developer who may create databases**

```sql
CREATE ROLE dev_ci LOGIN CREATEDB PASSWORD 'ChangeMe_#1' CONNECTION LIMIT 10;
```

**G. Named DBA superuser (never share `postgres`)**

```sql
CREATE ROLE dba_carol LOGIN SUPERUSER PASSWORD 'ChangeMe_#1' VALID UNTIL '2027-01-31';
```

**H. Role used only for certificate / peer auth (no password)**

```sql
CREATE ROLE svc_etl LOGIN PASSWORD NULL;
```

**I. Replication and monitoring users**

```sql
CREATE ROLE replicator LOGIN REPLICATION PASSWORD 'ChangeMe_#1' CONNECTION LIMIT 5;
CREATE ROLE pmm_monitor LOGIN PASSWORD 'ChangeMe_#1' CONNECTION LIMIT 10;
GRANT pg_monitor TO pmm_monitor;
```

**J. From the shell (wrappers around the same SQL)**

```bash
createuser --login --pwprompt --connection-limit=20 alice
createuser --no-login app_readonly
createuser --createdb --no-superuser --no-createrole dev_ci
dropuser alice
```

### 3.4 Verify what you created

```sql
\du+
```
```sql
SELECT rolname, rolsuper, rolinherit, rolcreaterole, rolcreatedb,
       rolcanlogin, rolreplication, rolbypassrls, rolconnlimit, rolvaliduntil
FROM   pg_roles
WHERE  rolname !~ '^pg_'
ORDER  BY rolname;
```

> Names beginning with `pg_` are **reserved** for predefined roles; PostgreSQL refuses to create them.

---

## 4. Passwords and Authentication

### 4.1 Password storage: use SCRAM

```sql
SHOW password_encryption;            -- should be scram-sha-256 (default since PG14)
SHOW scram_iterations;               -- default 4096 (PG16+); raise for stronger hashes
```

Check how each login role's password is stored (needs superuser):

```sql
-- as: postgres
SELECT rolname,
       CASE
         WHEN rolpassword IS NULL                THEN 'no password'
         WHEN rolpassword LIKE 'SCRAM-SHA-256$%' THEN 'scram-sha-256'
         WHEN rolpassword LIKE 'md5%'            THEN 'md5 (legacy!)'
         ELSE 'unknown'
       END AS storage,
       rolvaliduntil
FROM   pg_authid
WHERE  rolcanlogin
ORDER  BY 2, 1;
```

### 4.2 Setting and changing passwords

**Preferred — interactive, never exposes the cleartext in logs:**

```
\password alice
```
`psql` hashes the password **client-side** (using the server's `password_encryption`) and sends only the verifier.

**SQL form (cleartext travels to server; may appear in logs/`pg_stat_statements`/shell history):**

```sql
ALTER ROLE alice PASSWORD 'NewStrong#Pass9';
```

**Remove a password:**

```sql
ALTER ROLE alice PASSWORD NULL;
```

**Safer scripted rotation:** generate a SCRAM verifier outside the DB (e.g., with a client library) and supply the pre-hashed string; PostgreSQL stores it as-is:

```sql
ALTER ROLE alice PASSWORD 'SCRAM-SHA-256$4096:salt$StoredKey:ServerKey';
```

**Prevent passwords leaking into logs** — avoid cleartext `ALTER ROLE` when:

```
log_statement = 'ddl'          # logs ALTER ROLE ... PASSWORD
log_min_error_statement = error
```

### 4.3 Password expiry

```sql
ALTER ROLE alice VALID UNTIL '2026-12-31 23:59:59+05:30';
ALTER ROLE alice VALID UNTIL 'infinity';             -- never expires
```

Roles expiring in the next 14 days:

```sql
SELECT rolname, rolvaliduntil
FROM   pg_roles
WHERE  rolcanlogin
  AND  rolvaliduntil IS NOT NULL
  AND  rolvaliduntil < now() + interval '14 days'
ORDER  BY rolvaliduntil;
```

> `VALID UNTIL` only affects **password** authentication. A role authenticated via `cert`, `peer`, `ldap`, `gss` is unaffected — use `NOLOGIN` to block those.

### 4.4 `pg_hba.conf` — host-based authentication

Format:

```
# TYPE      DATABASE     USER            ADDRESS            METHOD   [OPTIONS]
local       all          postgres                           peer
local       all          all                                scram-sha-256
host        all          all             127.0.0.1/32       scram-sha-256
host        all          all             ::1/128            scram-sha-256

# application tiers (+group = any member of that role, direct or indirect)
hostssl     appdb        +app_readwrite  10.10.0.0/24       scram-sha-256
hostssl     appdb        +app_readonly   10.20.0.0/24       scram-sha-256

# DBAs by regex (PG16+) — user field starting with '/' is a regex
hostssl     all          /^dba_.*$       10.0.0.0/8         scram-sha-256

# replication and monitoring
hostssl     replication  replicator      10.10.0.0/24       scram-sha-256
hostssl     all          pmm_monitor     10.10.5.0/28       scram-sha-256

# reject all unencrypted remote connections, then everything else
hostnossl   all          all             0.0.0.0/0          reject
host        all          all             0.0.0.0/0          reject
```

Key rules:

- **First matching line wins** — order matters (specific before general).
- `TYPE`: `local` (Unix socket), `host` (TCP, SSL or not), `hostssl`, `hostnossl`, `hostgssenc`, `hostnogssenc`.
- `DATABASE`: name, `all`, `sameuser`, `samerole`, `replication` (physical replication, not a database), `@file`, regex (PG16+).
- `USER`: name, `all`, `+role` (membership), `@file`, regex.
- `METHOD`: `scram-sha-256`, `md5` (legacy), `password` (cleartext, **never**), `peer`, `ident`, `cert`, `ldap`, `gss`, `sspi`, `pam`, `radius`, `trust` (**never in prod**), `reject`.
- PG16+: `include`, `include_if_exists`, `include_dir` directives for modular files.

**Validate before reload** (look at the `error` column):

```sql
-- as: postgres
SELECT line_number, type, database, user_name, address, auth_method, error
FROM   pg_hba_file_rules
ORDER  BY line_number;
```

```sql
SELECT pg_reload_conf();
```

> An `md5`-method line accepts SCRAM-stored passwords too (it negotiates SCRAM). That makes the md5 → scram migration painless.

### 4.5 Mapping OS/cert identities with `pg_ident.conf`

`pg_ident.conf`:

```
# MAPNAME   SYSTEM-USERNAME          DATABASE-USERNAME
ops         shri                     postgres
ops         /^(.*)@corp\.example$    dba_\1
certmap     /^(.*)\.apps\.example$   svc_\1
```

`pg_hba.conf`:

```
local   all   postgres                     peer   map=ops
hostssl all   /^svc_.*$   10.10.0.0/24      cert   map=certmap
```

Check parsed maps (PG15+):

```sql
SELECT * FROM pg_ident_file_mappings;
```

### 4.6 Migrating from MD5 to SCRAM (no downtime)

```sql
-- Step 1: ensure new passwords are hashed with SCRAM
ALTER SYSTEM SET password_encryption = 'scram-sha-256';
SELECT pg_reload_conf();

-- Step 2: find MD5 holders
SELECT rolname FROM pg_authid WHERE rolpassword LIKE 'md5%';

-- Step 3: have each re-set their password (re-hashes as SCRAM)
--   psql> \password username

-- Step 4: once the Step 2 query returns zero rows, switch hba lines
--   from `md5` to `scram-sha-256` and reload.
```

Remember: client libraries must support SCRAM (libpq ≥ 10; check JDBC ≥ 42.2.0, psycopg2 built on libpq ≥ 10, Npgsql ≥ 4.0.3...).

### 4.7 TLS client-certificate authentication

```bash
# generate a client certificate whose CN equals the role name
openssl req -new -newkey rsa:4096 -nodes -keyout svc_etl.key -out svc_etl.csr -subj "/CN=svc_etl"
openssl x509 -req -in svc_etl.csr -CA root.crt -CAkey root.key -CAcreateserial -out svc_etl.crt -days 365
```

`postgresql.conf`:

```
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file  = 'server.key'
ssl_ca_file   = 'root.crt'
```

`pg_hba.conf`:

```
hostssl appdb svc_etl 10.10.0.0/24 cert
```

Client:

```bash
psql "host=db1 dbname=appdb user=svc_etl sslmode=verify-full sslrootcert=root.crt sslcert=svc_etl.crt sslkey=svc_etl.key"
```

### 4.8 LDAP authentication (the role must still exist in PostgreSQL)

```
host all +corp_users 10.0.0.0/8 ldap ldapserver=ldap.example.com ldapscheme=ldaps ldapprefix="uid=" ldapsuffix=",ou=people,dc=example,dc=com"
```

```sql
CREATE ROLE corp_users NOLOGIN;
CREATE ROLE jdoe LOGIN IN ROLE corp_users PASSWORD NULL;   -- authenticated by LDAP, not PG password
```

### 4.9 Client-side credential handling

**`~/.pgpass`** (must be `chmod 0600`):

```
# hostname:port:database:username:password
db1.example.com:5432:appdb:svc_orders:ChangeMe_#1
*:5432:*:alice:ChangeMe_#1
```

**`~/.pg_service.conf`:**

```
[appdb-ro]
host=db1.example.com
port=5432
dbname=appdb
user=alice
sslmode=verify-full
```
```bash
psql service=appdb-ro
```

**libpq `require_auth` (PG16+)** — client refuses weak methods (protects against a rogue server downgrading auth):

```bash
psql "host=db1 dbname=appdb user=alice require_auth=scram-sha-256 sslmode=verify-full"
```

### 4.10 Password complexity

PostgreSQL has **no built-in** password policy. Options:

| Tool | Notes |
|---|---|
| `passwordcheck` (contrib) | `shared_preload_libraries = 'passwordcheck'`; checks length/username/dictionary but only on **cleartext** passwords (it cannot inspect pre-hashed ones) |
| `credcheck` (third-party) | Policies for length, complexity, reuse, expiry, failed-attempt lockout |
| External IdP (LDAP/Kerberos/OAuth proxy) | Delegates policy to the identity provider |

---

## 5. Altering, Renaming and Dropping Roles

### 5.1 `ALTER ROLE`

```sql
-- attributes
ALTER ROLE alice CREATEDB;
ALTER ROLE alice NOCREATEDB;
ALTER ROLE alice CONNECTION LIMIT 5;
ALTER ROLE alice NOLOGIN;                -- disable login (existing sessions keep running!)
ALTER ROLE alice LOGIN;
ALTER ROLE alice NOINHERIT;
ALTER ROLE alice BYPASSRLS;
ALTER ROLE alice REPLICATION;

-- several at once
ALTER ROLE svc_orders WITH LOGIN CONNECTION LIMIT 100 VALID UNTIL '2027-06-30';

-- ALTER USER is identical
ALTER USER alice PASSWORD 'x';
```

Promote / demote a superuser:

```sql
-- as: postgres
ALTER ROLE dba_carol SUPERUSER;
ALTER ROLE dba_carol NOSUPERUSER;
```

### 5.2 Renaming

```sql
ALTER ROLE bob RENAME TO bob_s;
```

> Renaming **clears MD5 passwords** (the role name was the salt). SCRAM hashes are unaffected. You cannot rename the current session user, and the bootstrap superuser rename is allowed but strongly discouraged (scripts and extensions assume `postgres`).

### 5.3 Dropping roles

```sql
DROP ROLE alice;
DROP ROLE IF EXISTS alice;
DROP ROLE alice, bob;
```

A drop **fails** if the role still:

- owns objects (tables, schemas, databases, ...), or
- holds privileges on objects, or
- is referenced by default privileges, or
- is the grantor of other grants.

Safe, complete removal procedure (**repeat in every database of the cluster**):

```sql
-- 1) Decide the new owner of anything the old role owns
REASSIGN OWNED BY alice TO appdb_owner;      -- transfers ownership in the CURRENT database (+ shared objects)

-- 2) Remove all remaining privileges and default ACLs of that role in the CURRENT database
DROP OWNED BY alice;

-- 3) After doing 1 & 2 in each database, drop the role (once)
DROP ROLE alice;
```

Why diagnose first? See exactly what blocks the drop:

```sql
-- in each database
SELECT classid::regclass AS catalog, objid, deptype
FROM   pg_shdepend
WHERE  refobjid = 'alice'::regrole
  AND  dbid IN (0, (SELECT oid FROM pg_database WHERE datname = current_database()));
```
(`deptype`: `o` owner, `a` ACL entry, `r` policy role, `t` tablespace.)

> `DROP OWNED BY` **deletes** objects owned by the role if you skip `REASSIGN OWNED` first. Always reassign first.

---

## 6. Role Membership: INHERIT, SET, ADMIN

### 6.1 The three independent membership options (PG16+)

Every membership grant `member → role` has three booleans, stored in `pg_auth_members`:

| Option | Column | Default | Meaning |
|---|---|---|---|
| `INHERIT` | `inherit_option` | member's `rolinherit` attribute | Member automatically has the role's privileges |
| `SET` | `set_option` | `TRUE` | Member may `SET ROLE` to the role |
| `ADMIN` | `admin_option` | `FALSE` | Member may grant/revoke this role to/from others |

Also each grant records its **grantor**.

### 6.2 Granting and revoking membership

```sql
-- standard grant (inherit + set)
GRANT app_readonly TO alice;

-- several roles / several members
GRANT app_readonly, app_readwrite TO data_team;
GRANT app_readonly TO alice, bob;

-- explicit options
GRANT app_readwrite TO bob WITH INHERIT FALSE, SET TRUE;     -- "break-glass": must SET ROLE to use it
GRANT app_readonly  TO team_lead WITH ADMIN TRUE, INHERIT FALSE, SET FALSE;   -- can delegate, can't use

-- re-issuing GRANT with options updates the existing grant
GRANT app_readwrite TO bob WITH INHERIT TRUE;

-- record a particular grantor
GRANT app_readonly TO alice GRANTED BY dba_carol;

-- remove only an option
REVOKE ADMIN OPTION   FOR app_readonly FROM team_lead;
REVOKE INHERIT OPTION FOR app_readwrite FROM bob;
REVOKE SET OPTION     FOR app_readwrite FROM bob;

-- remove the membership completely
REVOKE app_readonly FROM alice;
REVOKE app_readonly FROM alice CASCADE;   -- also remove grants that alice made using her admin option
```

### 6.3 Demonstration: INHERIT vs SET

```sql
-- as: postgres
CREATE ROLE bob_demo LOGIN PASSWORD 'ChangeMe_#1';
-- connect rights come via app_readonly (inherit); write rights only via explicit SET ROLE
GRANT CONNECT ON DATABASE appdb TO app_readonly, app_readwrite;
GRANT USAGE ON SCHEMA sales TO app_readonly, app_readwrite;
GRANT SELECT ON ALL TABLES IN SCHEMA sales TO app_readonly;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA sales TO app_readwrite;
GRANT app_readonly  TO bob_demo;                                   -- inherited
GRANT app_readwrite TO bob_demo WITH INHERIT FALSE, SET TRUE;      -- set-only
```
```sql
-- as: bob_demo (psql -U bob_demo -d appdb)
SELECT count(*) FROM sales.orders;                     -- OK (inherits read)
DELETE FROM sales.orders WHERE order_id = 1;           -- ERROR: permission denied
SET ROLE app_readwrite;                                -- becomes app_readwrite
DELETE FROM sales.orders WHERE order_id = 1;           -- OK
SELECT current_user, session_user;                     -- app_readwrite | bob_demo
RESET ROLE;
```

`current_user` = effective role for permission checks; `session_user` = the authenticated login. Also available:

```sql
SET ROLE NONE;                          -- same as RESET ROLE
SET SESSION AUTHORIZATION alice;        -- superuser only; changes session_user itself
RESET SESSION AUTHORIZATION;
SET LOCAL ROLE app_readwrite;           -- only inside the current transaction
```

### 6.4 Delegated administration with `CREATEROLE` (PG16+ model)

Before PG16 a `CREATEROLE` user was nearly a superuser (could grant any role, alter any non-superuser). Now:

- It can create roles but **cannot** give attributes it lacks (`SUPERUSER`, `REPLICATION`, `BYPASSRLS`, and `CREATEDB`/`LOGIN` only if it has them).
- The creator automatically gets `ADMIN OPTION` on each role it creates → it may alter/drop/grant **only roles it administers**.
- It cannot modify superusers or roles it has no `ADMIN` on.
- To hand out an existing role (e.g., `app_readonly`), it needs `ADMIN OPTION` on that role.

```sql
-- as: postgres
CREATE ROLE team_lead LOGIN CREATEROLE PASSWORD 'ChangeMe_#1';

-- allow team_lead to delegate app_readonly, without being able to use it
GRANT app_readonly TO team_lead WITH ADMIN TRUE, INHERIT FALSE, SET FALSE;
```
```sql
-- as: team_lead
CREATE ROLE intern1 LOGIN PASSWORD 'ChangeMe_#1';      -- OK, team_lead gets ADMIN on intern1
GRANT app_readonly TO intern1;                          -- OK (has ADMIN on app_readonly)
GRANT app_readwrite TO intern1;                         -- ERROR: no ADMIN on app_readwrite
CREATE ROLE evil SUPERUSER;                             -- ERROR: must be superuser
ALTER ROLE alice PASSWORD 'x';                          -- ERROR: no ADMIN on alice
DROP ROLE intern1;                                      -- OK
```

Optionally let the creator **also** use the roles it creates:

```sql
-- values: '' (default), 'set', 'inherit', 'set, inherit'
ALTER ROLE team_lead SET createrole_self_grant = 'set, inherit';
```

### 6.5 Inspecting membership

```
\drg
\drg alice
```
```sql
SELECT r.rolname  AS role,
       m.rolname  AS member,
       g.rolname  AS grantor,
       am.admin_option, am.inherit_option, am.set_option
FROM   pg_auth_members am
JOIN   pg_roles r ON r.oid = am.roleid
JOIN   pg_roles m ON m.oid = am.member
JOIN   pg_roles g ON g.oid = am.grantor
ORDER  BY 1, 2;
```

Test effective membership:

```sql
SELECT pg_has_role('alice', 'app_readonly', 'MEMBER');   -- member directly or indirectly (ignores inherit)
SELECT pg_has_role('alice', 'app_readonly', 'USAGE');    -- privileges are inherited
SELECT pg_has_role('alice', 'app_readonly', 'SET');      -- can SET ROLE (PG16+)
```

Full transitive tree for a role:

```sql
WITH RECURSIVE tree AS (
    SELECT roleid, member, 1 AS depth, ARRAY[roleid] AS path
    FROM   pg_auth_members
    WHERE  member = 'alice'::regrole
  UNION ALL
    SELECT am.roleid, am.member, t.depth + 1, t.path || am.roleid
    FROM   pg_auth_members am
    JOIN   tree t ON am.member = t.roleid
    WHERE  NOT am.roleid = ANY (t.path)
)
SELECT depth, pg_get_userbyid(roleid) AS granted_role
FROM   tree
ORDER  BY depth, 2;
```

> **Circular membership** (A in B in A) is rejected by PostgreSQL.

---

## 7. Predefined Roles

Cluster-provided roles (names start with `pg_`). Grant them with `GRANT pg_xxx TO role;` (superuser, or someone with `ADMIN OPTION` on that predefined role).

| Role | Capability | Typical use |
|---|---|---|
| `pg_read_all_data` | `SELECT` on all tables/views/sequences + `USAGE` on all schemas (does **not** bypass RLS) | BI/reporting, logical `pg_dump` |
| `pg_write_all_data` | `INSERT/UPDATE/DELETE` on all tables, `USAGE/UPDATE` on sequences, `USAGE` on schemas | Data loaders (use sparingly) |
| `pg_read_all_settings` | Read all GUCs, incl. superuser-only ones | Monitoring/audit |
| `pg_read_all_stats` | Read all `pg_stat_*` views and extension stats, see other users' queries in `pg_stat_activity` | Monitoring |
| `pg_stat_scan_tables` | Run monitoring functions that take `ACCESS SHARE` locks on tables (e.g. `pgstattuple`) | Monitoring |
| `pg_monitor` | Member of `pg_read_all_settings`, `pg_read_all_stats`, `pg_stat_scan_tables` | PMM, Prometheus exporter, Datadog |
| `pg_database_owner` | Implicit member = current database owner | Ownership of `public` schema |
| `pg_signal_backend` | `pg_cancel_backend`/`pg_terminate_backend` on **non-superuser** sessions | Junior DBA / helpdesk |
| `pg_read_server_files` | `COPY ... FROM file`, file_fdw, read server files | **Dangerous** — OS file read |
| `pg_write_server_files` | `COPY ... TO file` on server | **Dangerous** |
| `pg_execute_server_program` | `COPY ... PROGRAM` — run OS commands as the postgres OS user | **Extremely dangerous** — equals shell access |
| `pg_checkpoint` | Run `CHECKPOINT` (15+) | Backup tooling |
| `pg_use_reserved_connections` | Use slots set by `reserved_connections` (16+) | Emergency access/monitoring |
| `pg_create_subscription` | `CREATE SUBSCRIPTION` without superuser (16+) | Logical replication admins |
| `pg_maintain` | `VACUUM`, `ANALYZE`, `CLUSTER`, `REFRESH MATERIALIZED VIEW`, `REINDEX`, `LOCK TABLE` on **all** relations (**17**) | Junior DBA / maintenance jobs |

Examples:

```sql
-- as: postgres
GRANT pg_monitor          TO pmm_monitor;
GRANT pg_read_all_data    TO alice;                       -- read-everything BI user
GRANT pg_signal_backend   TO helpdesk;
GRANT pg_maintain         TO dba_junior;                  -- PG17: maintenance without ownership
GRANT pg_checkpoint       TO backup_agent;
GRANT pg_create_subscription TO repl_admin;
```

Test `pg_maintain`:

```sql
-- as: dba_junior (member of pg_maintain)
VACUUM (ANALYZE, VERBOSE) sales.orders;
REINDEX TABLE sales.orders;
REFRESH MATERIALIZED VIEW reporting.some_mv;
```

> **Audit often:** anyone with `pg_execute_server_program`, `pg_read_server_files`, or `pg_write_server_files` can effectively reach the OS. Treat them like superusers.

List all current holders of predefined roles:

```sql
SELECT r.rolname AS predefined_role, m.rolname AS member, am.admin_option, am.inherit_option, am.set_option
FROM   pg_auth_members am
JOIN   pg_roles r ON r.oid = am.roleid AND r.rolname LIKE 'pg\_%'
JOIN   pg_roles m ON m.oid = am.member
ORDER  BY 1, 2;
```

---

## 8. Privileges: GRANT and REVOKE

### 8.1 Privilege matrix (what exists on which object)

| Object | Privileges | ACL letter |
|---|---|---|
| Table / view / mat. view | `SELECT` `INSERT` `UPDATE` `DELETE` `TRUNCATE` `REFERENCES` `TRIGGER` **`MAINTAIN` (17)** | `r a w d D x t m` |
| Column | `SELECT` `INSERT` `UPDATE` `REFERENCES` | `r a w x` |
| Sequence | `USAGE` `SELECT` `UPDATE` | `U r w` |
| Database | `CREATE` `CONNECT` `TEMPORARY` | `C c T` |
| Schema | `CREATE` `USAGE` | `C U` |
| Function / procedure / routine | `EXECUTE` | `X` |
| Type / domain | `USAGE` | `U` |
| Language | `USAGE` | `U` |
| Foreign data wrapper / foreign server | `USAGE` | `U` |
| Tablespace | `CREATE` | `C` |
| Large object | `SELECT` `UPDATE` | `r w` |
| Server parameter (15+) | `SET` `ALTER SYSTEM` | `s A` |

**Defaults granted to `PUBLIC` (everyone):** `CONNECT` and `TEMPORARY` on databases, `EXECUTE` on functions/procedures, `USAGE` on languages and types/domains. (Since PG15, `CREATE` on schema `public` is *not* granted on new clusters.) — This is why §11 hardening matters.

Reading ACL strings, e.g. `alice=r/appdb_owner`: *grantee `=` privileges `/` grantor*. An empty grantee (`=r/owner`) means `PUBLIC`. `*` after a letter (`r*`) means "with grant option".

### 8.2 `GRANT` syntax by object

```sql
-- TABLES (also views, materialized views, foreign tables)
GRANT SELECT ON sales.customers TO alice;
GRANT SELECT, INSERT, UPDATE, DELETE ON sales.orders TO app_readwrite;
GRANT ALL PRIVILEGES ON sales.orders TO appdb_owner;      -- all applicable (incl. MAINTAIN in 17)
GRANT TRUNCATE, REFERENCES, TRIGGER ON sales.orders TO dev_ci;
GRANT MAINTAIN ON sales.orders TO maint_job;               -- PG17

-- bulk by schema (existing objects only!)
GRANT SELECT ON ALL TABLES IN SCHEMA sales, reporting TO app_readonly;
GRANT MAINTAIN ON ALL TABLES IN SCHEMA sales TO maint_job;

-- COLUMNS
GRANT SELECT (employee_id, full_name, department) ON hr.employees TO app_readonly;
GRANT UPDATE (department) ON hr.employees TO hr_clerk;
GRANT INSERT (full_name, department, salary) ON hr.employees TO hr_clerk;

-- SEQUENCES (needed for INSERT into serial/identity columns)
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA sales TO app_readwrite;
GRANT USAGE ON SEQUENCE sales.orders_order_id_seq TO app_readwrite;

-- SCHEMAS
GRANT USAGE  ON SCHEMA sales TO app_readonly, app_readwrite;
GRANT CREATE ON SCHEMA sales TO dev_ci;                   -- may create objects in the schema

-- DATABASES
GRANT CONNECT ON DATABASE appdb TO app_readonly, app_readwrite;
GRANT TEMPORARY ON DATABASE appdb TO app_readwrite;
GRANT CREATE ON DATABASE appdb TO dev_ci;                 -- may create schemas

-- FUNCTIONS / PROCEDURES
GRANT EXECUTE ON FUNCTION reporting.revenue_for(text) TO app_readonly;
GRANT EXECUTE ON ALL FUNCTIONS  IN SCHEMA reporting TO app_readonly;
GRANT EXECUTE ON ALL PROCEDURES IN SCHEMA reporting TO app_readwrite;
GRANT EXECUTE ON ALL ROUTINES   IN SCHEMA reporting TO app_readwrite;

-- TYPES / DOMAINS / LANGUAGES
GRANT USAGE ON TYPE   reporting.some_enum TO app_readonly;
GRANT USAGE ON DOMAIN reporting.some_domain TO app_readonly;
GRANT USAGE ON LANGUAGE plpgsql TO app_readwrite;

-- FOREIGN DATA
GRANT USAGE ON FOREIGN DATA WRAPPER postgres_fdw TO dev_ci;
GRANT USAGE ON FOREIGN SERVER remote_srv TO dev_ci;

-- TABLESPACES
GRANT CREATE ON TABLESPACE fast_ssd TO dev_ci;

-- LARGE OBJECTS
GRANT SELECT, UPDATE ON LARGE OBJECT 16408 TO app_readwrite;

-- SERVER PARAMETERS (PG15+) — delegate tuning without superuser
GRANT SET          ON PARAMETER log_min_duration_statement TO dba_junior;
GRANT ALTER SYSTEM ON PARAMETER work_mem                   TO dba_junior;
GRANT SET, ALTER SYSTEM ON PARAMETER maintenance_work_mem, statement_timeout TO dba_junior;
```

### 8.3 Grant option, grantor and propagation

```sql
GRANT SELECT ON sales.orders TO team_lead WITH GRANT OPTION;       -- team_lead may re-grant SELECT
GRANT SELECT ON sales.orders TO alice GRANTED BY team_lead;        -- record the grantor (must be you or a role you're in)

REVOKE GRANT OPTION FOR SELECT ON sales.orders FROM team_lead;     -- keep SELECT, drop re-grant right
REVOKE SELECT ON sales.orders FROM team_lead CASCADE;              -- also revoke what team_lead granted onward
```

Rules worth remembering:

- Only the **owner** (or a role with grant option, or a member of the owning role) can grant on an object.
- The **owner's own** privileges can be revoked, but the owner can always re-grant them to themselves.
- `REVOKE` only removes grants made **by the revoker** (or by roles they act as). A privilege granted by role X must be revoked by X (or by a superuser acting as X, via `GRANTED BY`).

```sql
REVOKE SELECT ON sales.orders FROM alice GRANTED BY team_lead;
```

### 8.4 `REVOKE`

```sql
REVOKE INSERT, UPDATE, DELETE ON sales.orders FROM app_readwrite;
REVOKE ALL ON ALL TABLES IN SCHEMA sales FROM app_readonly;
REVOKE ALL ON DATABASE appdb FROM PUBLIC;
REVOKE EXECUTE ON FUNCTION reporting.revenue_for(text) FROM PUBLIC;
REVOKE USAGE ON LANGUAGE plpgsql FROM PUBLIC;
REVOKE SET ON PARAMETER work_mem FROM dba_junior;
```

### 8.5 Inspecting privileges

**psql:**

```
\dp sales.*
\z  sales.orders
\l+  appdb
\dn+
\df+ reporting.*
```

**SQL — "who can do what on this table":**

```sql
SELECT n.nspname AS schema, c.relname AS object,
       CASE a.grantee WHEN 0 THEN 'PUBLIC' ELSE pg_get_userbyid(a.grantee) END AS grantee,
       pg_get_userbyid(a.grantor) AS grantor,
       a.privilege_type, a.is_grantable
FROM   pg_class c
JOIN   pg_namespace n ON n.oid = c.relnamespace
CROSS  JOIN LATERAL aclexplode(c.relacl) a
WHERE  n.nspname IN ('sales','hr','reporting')
ORDER  BY 1, 2, 3, 5;
```

**Test a specific permission (honors membership/inheritance):**

```sql
SELECT has_table_privilege   ('alice', 'sales.orders', 'SELECT');
SELECT has_table_privilege   ('alice', 'sales.orders', 'SELECT, INSERT');       -- any of
SELECT has_column_privilege  ('alice', 'hr.employees', 'salary', 'SELECT');
SELECT has_schema_privilege  ('alice', 'sales', 'USAGE');
SELECT has_database_privilege('alice', 'appdb', 'CONNECT');
SELECT has_function_privilege('alice', 'reporting.revenue_for(text)', 'EXECUTE');
SELECT has_sequence_privilege('alice', 'sales.orders_order_id_seq', 'USAGE');
SELECT has_any_column_privilege('alice', 'hr.employees', 'SELECT');
SELECT has_parameter_privilege('dba_junior', 'work_mem', 'SET');                -- PG15+
SELECT has_table_privilege   ('alice', 'sales.orders', 'MAINTAIN');             -- PG17
```

**Effective privileges for one role across all tables:**

```sql
SELECT n.nspname, c.relname,
       has_table_privilege('alice', c.oid, 'SELECT') AS can_select,
       has_table_privilege('alice', c.oid, 'INSERT') AS can_insert,
       has_table_privilege('alice', c.oid, 'UPDATE') AS can_update,
       has_table_privilege('alice', c.oid, 'DELETE') AS can_delete
FROM   pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE  c.relkind IN ('r','p','v','m','f')
  AND  n.nspname NOT IN ('pg_catalog','information_schema','pg_toast')
ORDER  BY 1, 2;
```

### 8.6 Worked example: build the read-only / read-write tiers

```sql
-- as: postgres, connected to appdb
-- 1. Lock down defaults inherited by PUBLIC
REVOKE ALL ON DATABASE appdb FROM PUBLIC;
REVOKE ALL ON SCHEMA public FROM PUBLIC;

-- 2. Gate: connect + schema visibility
GRANT CONNECT ON DATABASE appdb TO app_readonly, app_readwrite;
GRANT USAGE ON SCHEMA sales, reporting TO app_readonly, app_readwrite;

-- 3. Existing objects
GRANT SELECT ON ALL TABLES    IN SCHEMA sales, reporting TO app_readonly;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA sales TO app_readwrite;
GRANT SELECT ON ALL TABLES    IN SCHEMA reporting TO app_readwrite;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA sales TO app_readwrite;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA reporting TO app_readonly, app_readwrite;

-- 4. Column-level: expose HR directory but hide salary / SSN
GRANT USAGE ON SCHEMA hr TO app_readonly;
GRANT SELECT (employee_id, full_name, department) ON hr.employees TO app_readonly;

-- 5. Future objects: see §9
```

Verify as a member:

```sql
GRANT app_readonly TO alice;
```
```sql
-- as: alice
SELECT * FROM sales.orders LIMIT 3;                         -- OK
SELECT full_name, department FROM hr.employees;             -- OK
SELECT salary FROM hr.employees;                            -- ERROR: permission denied for table employees
INSERT INTO sales.orders (customer_id, amount) VALUES (1,1);-- ERROR: permission denied
```

---

## 9. Default Privileges

`GRANT ... ON ALL TABLES IN SCHEMA` only affects objects that **exist now**. For future objects use `ALTER DEFAULT PRIVILEGES`.

Critical rule: defaults apply only to objects created by the role named in `FOR ROLE` (default: **the role running the command**).

```sql
-- as: postgres (or appdb_owner)
ALTER DEFAULT PRIVILEGES FOR ROLE appdb_owner IN SCHEMA sales
    GRANT SELECT ON TABLES TO app_readonly;

ALTER DEFAULT PRIVILEGES FOR ROLE appdb_owner IN SCHEMA sales
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_readwrite;

ALTER DEFAULT PRIVILEGES FOR ROLE appdb_owner IN SCHEMA sales
    GRANT USAGE, SELECT ON SEQUENCES TO app_readwrite;

ALTER DEFAULT PRIVILEGES FOR ROLE appdb_owner IN SCHEMA reporting
    GRANT EXECUTE ON FUNCTIONS TO app_readonly, app_readwrite;

-- not tied to a schema = applies to the whole database
ALTER DEFAULT PRIVILEGES FOR ROLE appdb_owner
    GRANT USAGE ON TYPES TO app_readonly;

-- stop the built-in "EXECUTE to PUBLIC" for functions created by appdb_owner
ALTER DEFAULT PRIVILEGES FOR ROLE appdb_owner
    REVOKE EXECUTE ON FUNCTIONS FROM PUBLIC;

-- remove a default
ALTER DEFAULT PRIVILEGES FOR ROLE appdb_owner IN SCHEMA sales
    REVOKE SELECT ON TABLES FROM app_readonly;

-- PG17: default MAINTAIN
ALTER DEFAULT PRIVILEGES FOR ROLE appdb_owner IN SCHEMA sales
    GRANT MAINTAIN ON TABLES TO maint_job;
```

Inspect them:

```
\ddp
```
```sql
SELECT pg_get_userbyid(defaclrole) AS for_role,
       coalesce(n.nspname,'(all schemas)') AS in_schema,
       defaclobjtype AS objtype,      -- r tables, S sequences, f functions, T types, n schemas
       defaclacl
FROM   pg_default_acl d
LEFT   JOIN pg_namespace n ON n.oid = d.defaclnamespace;
```

Test it:

```sql
SET ROLE appdb_owner;
CREATE TABLE sales.returns (return_id serial PRIMARY KEY, order_id bigint, reason text);
RESET ROLE;
SELECT has_table_privilege('alice', 'sales.returns', 'SELECT');     -- true
```

> **Common gotcha:** you configured defaults `FOR ROLE appdb_owner`, but developers create tables as `bob`. Those tables get **no** auto-grants. Standardize: DDL is executed by the owner role (`SET ROLE appdb_owner` or a migration tool logging in as the owner), or define defaults for each creator role.

---

## 10. Ownership Management

### 10.1 Why ownership matters

The owner of an object implicitly holds **all** privileges on it, can `ALTER`/`DROP` it, can `GRANT`/`REVOKE` on it, and — for tables — bypasses RLS unless `FORCE ROW LEVEL SECURITY` is set. Therefore:

- **Never let application login roles own tables.** Use a separate `NOLOGIN` owner role plus app roles with DML privileges only.
- Owner roles should be non-superusers.

### 10.2 Changing ownership

```sql
ALTER DATABASE appdb       OWNER TO appdb_owner;
ALTER SCHEMA   sales       OWNER TO appdb_owner;
ALTER TABLE    sales.orders OWNER TO appdb_owner;
ALTER SEQUENCE sales.orders_order_id_seq OWNER TO appdb_owner;
ALTER VIEW     reporting.order_summary OWNER TO appdb_owner;
ALTER FUNCTION reporting.revenue_for(text) OWNER TO appdb_owner;
ALTER TYPE     reporting.some_enum OWNER TO appdb_owner;
ALTER TABLESPACE fast_ssd  OWNER TO appdb_owner;
```

> Changing a table's owner does **not** change the owner of its **owned sequences** if they were created independently; for `serial`/identity columns the sequence ownership follows the table (via `ALTER TABLE ... OWNER`).

You must be a member of the new owner role (or a superuser), and the new owner needs `CREATE` on the schema/database as applicable.

Bulk-fix ownership for an entire schema:

```sql
DO $$
DECLARE r record;
BEGIN
  FOR r IN
    SELECT format('ALTER TABLE %I.%I OWNER TO appdb_owner', schemaname, tablename) AS cmd
    FROM   pg_tables WHERE schemaname IN ('sales','hr','reporting')
    UNION ALL
    SELECT format('ALTER SEQUENCE %I.%I OWNER TO appdb_owner', schemaname, sequencename)
    FROM   pg_sequences WHERE schemaname IN ('sales','hr','reporting')
    UNION ALL
    SELECT format('ALTER VIEW %I.%I OWNER TO appdb_owner', schemaname, viewname)
    FROM   pg_views WHERE schemaname IN ('sales','hr','reporting')
  LOOP
    EXECUTE r.cmd;
  END LOOP;
END $$;
```

### 10.3 Who owns what

```sql
-- all relations owned by one role in the current db
SELECT n.nspname, c.relname, c.relkind
FROM   pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE  c.relowner = 'bob'::regrole
ORDER  BY 1, 2;

-- relations NOT owned by the expected owner
SELECT n.nspname, c.relname, pg_get_userbyid(c.relowner) AS owner
FROM   pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE  n.nspname IN ('sales','hr','reporting')
  AND  c.relkind IN ('r','p','v','m','S','f')
  AND  c.relowner <> 'appdb_owner'::regrole;

-- databases and schemas
SELECT datname, pg_get_userbyid(datdba) AS owner FROM pg_database;
SELECT nspname, pg_get_userbyid(nspowner) AS owner FROM pg_namespace WHERE nspname !~ '^pg_';
```

### 10.4 Offboarding a person: full procedure

Assume `bob` leaves, objects go to `appdb_owner`.

```sql
-- 0. Block new logins and end existing sessions
ALTER ROLE bob NOLOGIN;
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE usename = 'bob';

-- 1. In EVERY database in the cluster (use \c or a loop, see below):
REASSIGN OWNED BY bob TO appdb_owner;
DROP OWNED BY bob;

-- 2. Cluster-level removal
DROP ROLE bob;
```

Loop over all databases from the shell:

```bash
for db in $(psql -Atc "SELECT datname FROM pg_database WHERE datallowconn AND NOT datistemplate;"); do
  psql -d "$db" -v ON_ERROR_STOP=1 -c "REASSIGN OWNED BY bob TO appdb_owner;" -c "DROP OWNED BY bob;"
done
psql -c "DROP ROLE bob;"
```

If `DROP ROLE` still errors with *"role cannot be dropped because some objects depend on it"*, the `DETAIL` lists the leftover (database, privileges on other databases, default ACLs, policies...). Repeat step 1 in the database named there. The loop above skips template databases; if you ever granted anything in `template1`, run the two commands there as well (`template0` normally holds nothing and refuses connections).

---

## 11. Database, Schema and Function Hardening

### 11.1 Lock down `PUBLIC`

```sql
-- as: postgres
-- Per database: nobody connects unless explicitly allowed
REVOKE ALL ON DATABASE appdb FROM PUBLIC;
REVOKE ALL ON DATABASE postgres FROM PUBLIC;
REVOKE ALL ON DATABASE template1 FROM PUBLIC;       -- optional, prevents inheritance into new DBs

-- In each database: schema public
\c appdb
REVOKE ALL ON SCHEMA public FROM PUBLIC;            -- already CREATE-less on PG15+ new clusters; USAGE still granted
GRANT  USAGE ON SCHEMA public TO app_readonly, app_readwrite;   -- only if you intend to use it
```

> Upgraded clusters (from ≤ 14) **keep** `CREATE` on `public` for `PUBLIC` — check:
> `SELECT nspacl FROM pg_namespace WHERE nspname = 'public';` — look for `=UC/...` (bad) vs `=U/...` (acceptable).

Make every **new** database start locked down by changing `template1` once:

```sql
\c template1
REVOKE ALL ON SCHEMA public FROM PUBLIC;
GRANT USAGE ON SCHEMA public TO PUBLIC;             -- or omit for a stricter baseline
```

### 11.2 Control who can create objects

```sql
-- Only the owner role creates objects in business schemas
REVOKE CREATE ON SCHEMA sales, hr, reporting FROM PUBLIC;
-- Users may not create temp tables unless allowed
REVOKE TEMPORARY ON DATABASE appdb FROM PUBLIC;
```

### 11.3 Safe `search_path`

Attackers who can create objects in a schema earlier in a victim's `search_path` can hijack unqualified function/operator calls.

```sql
ALTER ROLE svc_orders SET search_path = sales, reporting;           -- no "$user", no public
ALTER DATABASE appdb  SET search_path = "$user", sales;             -- sensible default
```

PG17 note: maintenance operations (autovacuum, `ANALYZE`, `REINDEX`, `CLUSTER`, `REFRESH MATERIALIZED VIEW`) now run with a **safe `search_path`**. Functions used by expression indexes / generated columns / mat. views must be **schema-qualified** or they can fail after upgrading.

### 11.4 `SECURITY DEFINER` functions

They run with the **owner's** privileges — powerful and risky.

```sql
CREATE FUNCTION reporting.mask_ssn(p_id int) RETURNS text
LANGUAGE sql
SECURITY DEFINER
SET search_path = pg_catalog, pg_temp           -- mandatory hardening
AS $$ SELECT 'XXX-XX-' || ssn_last4 FROM hr.employees WHERE employee_id = p_id $$;

ALTER FUNCTION reporting.mask_ssn(int) OWNER TO appdb_owner;
REVOKE ALL ON FUNCTION reporting.mask_ssn(int) FROM PUBLIC;     -- default EXECUTE-to-PUBLIC would expose it
GRANT  EXECUTE ON FUNCTION reporting.mask_ssn(int) TO app_readonly;
```

Checklist: fixed `search_path` incl. `pg_temp` last, qualified object names, `REVOKE ... FROM PUBLIC`, owner is a non-superuser with minimum privileges.

### 11.5 Superuser safety

```sql
-- list superusers and anyone who can behave like one
SELECT rolname FROM pg_roles WHERE rolsuper;
SELECT m.rolname AS member, r.rolname AS dangerous_role
FROM   pg_auth_members am
JOIN   pg_roles r ON r.oid = am.roleid
JOIN   pg_roles m ON m.oid = am.member
WHERE  r.rolname IN ('pg_execute_server_program','pg_write_server_files','pg_read_server_files');
```

Block remote superuser logins in `pg_hba.conf`:

```
host  all  postgres  0.0.0.0/0  reject
local all  postgres             peer
```
(Place the `reject` line **before** broader allow-rules.)

PG17 extras:

```
allow_alter_system = off      # blocks ALTER SYSTEM (hygiene for config-managed servers; not a security boundary — superusers can still do other things)
```

---

## 12. Row-Level Security (RLS)

RLS filters/blocks individual rows per role. Owners and superusers bypass unless `FORCE` is set; `BYPASSRLS` roles bypass too.

### 12.1 Per-user rows (employee sees only own record)

```sql
-- as: postgres  (roles and grants first)
CREATE ROLE hr_staff NOLOGIN;      -- everyone allowed to query hr.employees (rows then filtered by policy)
CREATE ROLE hr_admin NOLOGIN;      -- sees and edits all rows
GRANT hr_staff TO alice;
GRANT USAGE  ON SCHEMA hr TO hr_staff, hr_admin;
GRANT SELECT ON hr.employees TO hr_staff;
GRANT SELECT, INSERT, UPDATE, DELETE ON hr.employees TO hr_admin;
```
```sql
-- as: postgres, then switch to the table owner so the policies are owned cleanly
SET ROLE appdb_owner;

ALTER TABLE hr.employees ENABLE ROW LEVEL SECURITY;
ALTER TABLE hr.employees FORCE  ROW LEVEL SECURITY;      -- apply to the owner too

CREATE POLICY emp_self_select ON hr.employees
    FOR SELECT TO hr_staff
    USING (db_user = current_user);

CREATE POLICY hr_admin_all ON hr.employees
    FOR ALL TO hr_admin
    USING (true) WITH CHECK (true);

RESET ROLE;
```
```sql
-- as: alice (member of hr_staff)  -> sees only her own row
SELECT employee_id, full_name, salary FROM hr.employees;
```

> A table with RLS enabled and **no** applicable policy is **default-deny** for non-owners/non-bypass roles.

### 12.2 Multi-tenant isolation using a session variable

```sql
-- as: appdb_owner
CREATE TABLE sales.regional_targets (
    target_id serial PRIMARY KEY,
    region    text NOT NULL,
    target    numeric(12,2) NOT NULL
);
INSERT INTO sales.regional_targets (region, target)
VALUES ('SOUTH', 100000), ('NORTH', 90000), ('EAST', 80000), ('WEST', 70000);

ALTER TABLE sales.regional_targets ENABLE ROW LEVEL SECURITY;

CREATE POLICY region_isolation ON sales.regional_targets
    FOR ALL TO app_readwrite
    USING      (region = current_setting('app.region', true))
    WITH CHECK (region = current_setting('app.region', true));

GRANT SELECT, INSERT, UPDATE, DELETE ON sales.regional_targets TO app_readwrite;
```
```sql
-- app connection pool sets this per request/transaction:
BEGIN;
SET LOCAL app.region = 'SOUTH';
SELECT * FROM sales.regional_targets;     -- only SOUTH
COMMIT;
```

> The session variable is only as trustworthy as the code setting it — for hard multi-tenancy, bind the tenant to the **login role** (policy on `current_user`) or use a role per tenant with `SET ROLE`.

### 12.3 Managing and inspecting policies

```sql
ALTER POLICY region_isolation ON sales.regional_targets TO app_readwrite, app_readonly;
DROP  POLICY region_isolation ON sales.regional_targets;
ALTER TABLE sales.regional_targets DISABLE ROW LEVEL SECURITY;

SELECT schemaname, tablename, policyname, permissive, roles, cmd, qual, with_check
FROM   pg_policies;

SELECT relname, relrowsecurity, relforcerowsecurity
FROM   pg_class WHERE relrowsecurity;
```

`pg_dump` and RLS: dumping a table with RLS as a non-bypass role returns **only visible rows** (or errors with `row_security = off`). Give backup roles `BYPASSRLS`:

```sql
ALTER ROLE backup_agent BYPASSRLS;
```

---

## 13. Per-Role Settings, Limits and Login Controls

### 13.1 Per-role / per-database GUC defaults

```sql
-- apply on next login of that role
ALTER ROLE svc_orders SET statement_timeout = '30s';
ALTER ROLE svc_orders SET idle_in_transaction_session_timeout = '60s';
ALTER ROLE svc_orders SET idle_session_timeout = '15min';
ALTER ROLE svc_orders SET lock_timeout = '5s';
ALTER ROLE svc_orders SET search_path = sales, reporting;
ALTER ROLE alice      SET work_mem = '64MB';
ALTER ROLE alice      SET default_transaction_read_only = on;     -- enforce read-only sessions
ALTER ROLE alice      SET log_statement = 'all';                  -- audit one user (superuser needed)

-- role AND database
ALTER ROLE svc_orders IN DATABASE appdb SET work_mem = '16MB';

-- all roles in one database / all databases
ALTER ROLE ALL IN DATABASE appdb SET idle_in_transaction_session_timeout = '5min';
ALTER DATABASE appdb SET temp_file_limit = '2GB';

-- undo
ALTER ROLE svc_orders RESET statement_timeout;
ALTER ROLE svc_orders RESET ALL;
```

Inspect:

```
\drds
```
```sql
SELECT coalesce(pg_get_userbyid(setrole),'(all roles)') AS role,
       coalesce((SELECT datname FROM pg_database WHERE oid = setdatabase),'(all dbs)') AS db,
       setconfig
FROM   pg_db_role_setting;
```

Precedence (highest wins): **role+database** > **role** > **database** > global `ALL`.
(Settings from `postgresql.conf` are the base; command-line `-c` and `SET` in-session override.)

### 13.2 Connection limits

```sql
ALTER ROLE svc_orders CONNECTION LIMIT 50;
ALTER DATABASE appdb  CONNECTION LIMIT 200;
```

Capacity picture:

```sql
SHOW max_connections;
SHOW superuser_reserved_connections;       -- default 3: slots only superusers can use
SHOW reserved_connections;                 -- PG16+: slots for pg_use_reserved_connections members
```
```sql
GRANT pg_use_reserved_connections TO pmm_monitor;     -- monitoring keeps working during a connection storm
```

Per-role usage vs limit:

```sql
SELECT r.rolname, r.rolconnlimit AS limit, count(a.pid) AS in_use
FROM   pg_roles r
LEFT   JOIN pg_stat_activity a ON a.usename = r.rolname
WHERE  r.rolcanlogin
GROUP  BY 1, 2
ORDER  BY in_use DESC;
```

### 13.3 Login event triggers (PG17)

Run logic at every login — e.g., enforce an access window or restrict a role. A login trigger must be a `RETURNS event_trigger` function; raising an exception **blocks the login**.

```sql
-- as: postgres, in appdb
CREATE FUNCTION public.block_contractors_after_hours() RETURNS event_trigger
LANGUAGE plpgsql AS $$
BEGIN
  IF pg_has_role(session_user, 'contractors', 'MEMBER')
     AND extract(hour FROM now() AT TIME ZONE 'Asia/Kolkata') NOT BETWEEN 8 AND 20 THEN
     RAISE EXCEPTION 'Contractor access allowed 08:00-20:59 IST only';
  END IF;
END $$;

CREATE EVENT TRIGGER trg_login_window ON login
    EXECUTE FUNCTION public.block_contractors_after_hours();
```

Safety:

- A buggy login trigger can lock **everyone** out of that database. Superusers can bypass by connecting with `event_triggers` disabled:
  ```bash
  psql "dbname=appdb options='-c event_triggers=off'"
  ```
- Keep the function fast (it runs on every connect) and avoid writes unless needed (write-on-login causes bloat and contention).
- Manage with `ALTER EVENT TRIGGER trg_login_window DISABLE;` / `DROP EVENT TRIGGER trg_login_window;`.

### 13.4 Session logging for access audit

`postgresql.conf`:

```
log_connections        = on
log_disconnections     = on
log_line_prefix        = '%m [%p] %q%u@%d/%a %h '
log_hostname           = off
log_statement          = 'ddl'          # none | ddl | mod | all
log_min_duration_statement = 1000
```

Per-role extra logging:

```sql
ALTER ROLE dba_carol SET log_statement = 'all';
```

---

## 14. Real-World Role Blueprints (RBAC Design)

### 14.1 The recommended layered model

```
 Layer 1  Object owners   (NOLOGIN)   appdb_owner                 ← owns database, schemas, tables
 Layer 2  Privilege roles (NOLOGIN)   app_readonly, app_readwrite ← hold the GRANTs
 Layer 3  Login roles                 alice, bob, svc_orders ...  ← members of layer 2, hold no direct grants
```

Principles:

1. **Grant to roles, not people.** People/services only receive memberships.
2. **Separate owner from user.** Application accounts never own schema objects.
3. **Least privilege + explicit connect.** `PUBLIC` has nothing.
4. **One identity per person/service.** No shared logins; no day-to-day `postgres`.
5. **Everything scripted** (idempotent SQL in version control).

### 14.2 Baseline script (idempotent-style)

```sql
-- as: postgres
-- ---------- Layer 1 / 2 ----------
DO $$
BEGIN
  IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'appdb_owner')    THEN CREATE ROLE appdb_owner NOLOGIN;    END IF;
  IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'app_readonly')   THEN CREATE ROLE app_readonly NOLOGIN;   END IF;
  IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'app_readwrite')  THEN CREATE ROLE app_readwrite NOLOGIN;  END IF;
  IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'contractors')    THEN CREATE ROLE contractors NOLOGIN;    END IF;
END $$;

-- a read-write user can always read
GRANT app_readonly TO app_readwrite;

\c appdb
REVOKE ALL ON DATABASE appdb FROM PUBLIC;
GRANT CONNECT ON DATABASE appdb TO app_readonly;                   -- inherited by app_readwrite
GRANT USAGE   ON SCHEMA sales, reporting TO app_readonly;
GRANT SELECT  ON ALL TABLES IN SCHEMA sales, reporting TO app_readonly;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA reporting TO app_readonly;

GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA sales TO app_readwrite;
GRANT USAGE, SELECT, UPDATE  ON ALL SEQUENCES IN SCHEMA sales TO app_readwrite;

ALTER DEFAULT PRIVILEGES FOR ROLE appdb_owner IN SCHEMA sales
    GRANT SELECT ON TABLES TO app_readonly;
ALTER DEFAULT PRIVILEGES FOR ROLE appdb_owner IN SCHEMA sales
    GRANT INSERT, UPDATE, DELETE ON TABLES TO app_readwrite;
ALTER DEFAULT PRIVILEGES FOR ROLE appdb_owner IN SCHEMA sales
    GRANT USAGE, SELECT, UPDATE ON SEQUENCES TO app_readwrite;
ALTER DEFAULT PRIVILEGES FOR ROLE appdb_owner IN SCHEMA reporting
    GRANT SELECT ON TABLES TO app_readonly;
ALTER DEFAULT PRIVILEGES FOR ROLE appdb_owner IN SCHEMA reporting
    GRANT EXECUTE ON FUNCTIONS TO app_readonly;
```

### 14.3 Human users

```sql
-- Analyst: read-only
CREATE ROLE alice LOGIN PASSWORD 'ChangeMe_#1' IN ROLE app_readonly VALID UNTIL '2027-06-30';
ALTER  ROLE alice SET default_transaction_read_only = on;
ALTER  ROLE alice SET statement_timeout = '5min';

-- Developer: read-write in non-prod, read-only in prod (separate grants per environment)
CREATE ROLE bob LOGIN PASSWORD 'ChangeMe_#1' IN ROLE app_readwrite;

-- Junior DBA: no superuser; maintenance, kill sessions, tune a few parameters, see stats
CREATE ROLE dba_junior LOGIN PASSWORD 'ChangeMe_#1' CONNECTION LIMIT 5;
GRANT pg_monitor, pg_signal_backend, pg_maintain, pg_checkpoint TO dba_junior;
GRANT SET          ON PARAMETER log_min_duration_statement, statement_timeout, work_mem TO dba_junior;
GRANT ALTER SYSTEM ON PARAMETER log_min_duration_statement TO dba_junior;

-- Senior DBA: personal superuser, named, expiring, extra logging
CREATE ROLE dba_carol LOGIN SUPERUSER PASSWORD 'ChangeMe_#1' VALID UNTIL '2027-01-31';
ALTER  ROLE dba_carol SET log_statement = 'all';
```

### 14.4 Application service accounts

```sql
CREATE ROLE svc_orders LOGIN PASSWORD 'ChangeMe_#1' CONNECTION LIMIT 50 IN ROLE app_readwrite;
ALTER  ROLE svc_orders SET search_path = sales, reporting;
ALTER  ROLE svc_orders SET statement_timeout = '30s';
ALTER  ROLE svc_orders SET idle_in_transaction_session_timeout = '30s';
ALTER  ROLE svc_orders SET lock_timeout = '5s';

-- A reporting service: read-only
CREATE ROLE svc_reports LOGIN PASSWORD 'ChangeMe_#1' CONNECTION LIMIT 20 IN ROLE app_readonly;
ALTER  ROLE svc_reports SET default_transaction_read_only = on;
```

Migration tool (Flyway/Liquibase/Alembic) logs in as a role that can `SET ROLE appdb_owner`:

```sql
CREATE ROLE svc_migrate LOGIN PASSWORD 'ChangeMe_#1' CONNECTION LIMIT 3;
GRANT appdb_owner TO svc_migrate;                   -- inherit+set; or WITH INHERIT FALSE and use SET ROLE in scripts
GRANT CREATE ON DATABASE appdb TO appdb_owner;
```
Restrict `svc_migrate` in `pg_hba.conf` to the CI subnet, and optionally `ALTER ROLE svc_migrate NOLOGIN` outside release windows.

### 14.5 Monitoring (PMM / Prometheus postgres_exporter / Datadog)

```sql
CREATE ROLE pmm_monitor LOGIN PASSWORD 'ChangeMe_#1' CONNECTION LIMIT 10;
GRANT pg_monitor TO pmm_monitor;
GRANT pg_use_reserved_connections TO pmm_monitor;       -- optional (needs reserved_connections > 0)
ALTER ROLE pmm_monitor SET statement_timeout = '30s';
ALTER ROLE pmm_monitor SET log_statement = 'none';
```
```sql
-- extension for query stats: create in the database the agent connects to
\c postgres
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
-- postgresql.conf: shared_preload_libraries = 'pg_stat_statements'   (restart required)
```
```
hostssl all pmm_monitor 10.10.5.0/28 scram-sha-256
```

### 14.6 Backup roles

```sql
-- Logical backups (pg_dump / pg_dumpall without superuser)
CREATE ROLE backup_logical LOGIN PASSWORD 'ChangeMe_#1';
GRANT pg_read_all_data TO backup_logical;
ALTER ROLE backup_logical BYPASSRLS;                  -- so RLS doesn't hide rows from the dump

-- Physical backups (pg_basebackup / pgBackRest / Barman streaming)
CREATE ROLE backup_physical LOGIN REPLICATION PASSWORD 'ChangeMe_#1' CONNECTION LIMIT 3;
GRANT pg_checkpoint TO backup_physical;

-- Low-level exclusive/non-exclusive backup API without superuser (PG15+ names)
GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_start(text, boolean) TO backup_physical;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_stop(boolean)        TO backup_physical;
GRANT EXECUTE ON FUNCTION pg_catalog.pg_switch_wal()                TO backup_physical;
```
```
hostssl replication backup_physical 10.10.9.0/28 scram-sha-256
hostssl all         backup_logical  10.10.9.0/28 scram-sha-256
```

### 14.7 Streaming and logical replication

```sql
-- Physical standby
CREATE ROLE replicator LOGIN REPLICATION PASSWORD 'ChangeMe_#1' CONNECTION LIMIT 5;

-- Logical replication publisher side
CREATE ROLE pub_user LOGIN REPLICATION PASSWORD 'ChangeMe_#1';
GRANT USAGE  ON SCHEMA sales TO pub_user;
GRANT SELECT ON ALL TABLES IN SCHEMA sales TO pub_user;

-- Subscriber admin that is NOT a superuser (PG16+)
CREATE ROLE sub_admin LOGIN PASSWORD 'ChangeMe_#1';
GRANT pg_create_subscription TO sub_admin;
GRANT CREATE ON DATABASE appdb TO sub_admin;
```
```
hostssl replication replicator 10.10.0.0/24 scram-sha-256
hostssl appdb       pub_user   10.10.0.0/24 scram-sha-256
```

### 14.8 ETL / data-loading role confined to a staging schema

```sql
-- as: postgres
CREATE SCHEMA staging AUTHORIZATION appdb_owner;
CREATE ROLE etl_loader LOGIN PASSWORD 'ChangeMe_#1' CONNECTION LIMIT 5;

GRANT CONNECT ON DATABASE appdb TO etl_loader;
GRANT USAGE, CREATE ON SCHEMA staging TO etl_loader;          -- may create/drop its own staging tables
GRANT USAGE ON SCHEMA sales TO etl_loader;
GRANT SELECT ON ALL TABLES IN SCHEMA sales TO etl_loader;
GRANT INSERT ON sales.customers, sales.orders TO etl_loader;  -- final load only
GRANT USAGE ON ALL SEQUENCES IN SCHEMA sales TO etl_loader;
ALTER ROLE etl_loader SET search_path = staging, sales;
```

### 14.9 Auditor / compliance read-only

```sql
CREATE ROLE auditor LOGIN PASSWORD 'ChangeMe_#1';
GRANT pg_read_all_settings, pg_read_all_stats TO auditor;      -- config + activity visibility, no data
GRANT CONNECT ON DATABASE appdb TO auditor;
GRANT USAGE ON SCHEMA public TO auditor;
-- no access to application data by design
```

### 14.10 BI user who sees *everything* (current and future tables)

```sql
CREATE ROLE bi_reader LOGIN PASSWORD 'ChangeMe_#1';
GRANT pg_read_all_data TO bi_reader;             -- also needs CONNECT on the database
GRANT CONNECT ON DATABASE appdb TO bi_reader;
ALTER ROLE bi_reader SET default_transaction_read_only = on;
ALTER ROLE bi_reader SET statement_timeout = '10min';
```
`pg_read_all_data` covers future objects automatically (no default-privilege bookkeeping). It does **not** bypass RLS and covers *all* schemas, including sensitive ones — consider whether column/schema-level grants are more appropriate.

### 14.11 Temporary contractor access (auto-expiring)

```sql
CREATE ROLE vendor_jane LOGIN PASSWORD 'ChangeMe_#1'
    VALID UNTIL '2026-11-30 18:00:00+05:30'
    CONNECTION LIMIT 2
    IN ROLE app_readonly, contractors;
ALTER ROLE vendor_jane SET statement_timeout = '2min';
ALTER ROLE vendor_jane SET default_transaction_read_only = on;
```
Pair with the login-window trigger from §13.3 (that function references the `contractors` role) and a weekly cleanup:

```sql
-- expired accounts still sitting in the cluster
SELECT rolname, rolvaliduntil
FROM   pg_roles
WHERE  rolcanlogin AND rolvaliduntil < now();
```

### 14.12 Per-developer sandbox schema

```sql
CREATE SCHEMA dev_bob AUTHORIZATION bob;                  -- bob owns his schema only
GRANT CONNECT ON DATABASE appdb TO bob;                   -- if not inherited already
GRANT USAGE ON SCHEMA sales TO bob;
GRANT SELECT ON ALL TABLES IN SCHEMA sales TO bob;
ALTER ROLE bob IN DATABASE appdb SET search_path = dev_bob, sales;
```

### 14.13 Break-glass emergency role

```sql
CREATE ROLE breakglass_admin NOLOGIN SUPERUSER;                  -- nobody logs in directly
CREATE ROLE dba_carol_bg LOGIN PASSWORD 'ChangeMe_#1';
GRANT breakglass_admin TO dba_carol_bg WITH INHERIT FALSE, SET TRUE;   -- must consciously SET ROLE
ALTER ROLE breakglass_admin SET log_statement = 'all';
```
```sql
-- during an incident (all statements logged under the real session_user)
SET ROLE breakglass_admin;
-- ... emergency work ...
RESET ROLE;
```
Alert on `SET ROLE breakglass_admin` using `log_statement = 'all'` + a log-forwarder rule, or pgaudit (§16.5).

---

## 15. Session Management

### 15.1 See who is connected

```sql
SELECT pid, usename, datname, application_name, client_addr, backend_start,
       state, wait_event_type, wait_event, now() - query_start AS runtime, left(query, 80) AS query
FROM   pg_stat_activity
WHERE  backend_type = 'client backend'
ORDER  BY backend_start;

-- per-role summary
SELECT usename, count(*) AS sessions,
       count(*) FILTER (WHERE state = 'active')              AS active,
       count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_txn
FROM   pg_stat_activity
WHERE  backend_type = 'client backend'
GROUP  BY usename ORDER BY sessions DESC;
```

> Non-superusers only see **their own** query text unless they have `pg_read_all_stats` (or `pg_monitor`).

### 15.2 Cancel / terminate

```sql
SELECT pg_cancel_backend(12345);          -- cancel current query only (SIGINT)
SELECT pg_terminate_backend(12345);       -- kill the session (SIGTERM)

-- all sessions of one role
SELECT pg_terminate_backend(pid)
FROM   pg_stat_activity
WHERE  usename = 'bob' AND pid <> pg_backend_pid();

-- with a timeout to wait for exit (PG14+): returns true if the process ended within 5000 ms
SELECT pg_terminate_backend(12345, 5000);

-- everyone connected to a database (e.g., before DROP DATABASE / restore)
SELECT pg_terminate_backend(pid)
FROM   pg_stat_activity
WHERE  datname = 'appdb' AND pid <> pg_backend_pid();
```

Permission rules: superuser can signal anything; `pg_signal_backend` members can signal **non-superuser** sessions; any role can signal sessions of the **same role**.

### 15.3 Locking out a user immediately

```sql
ALTER ROLE bob NOLOGIN;                                            -- blocks new logins
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE usename = 'bob';   -- kills existing
```
Or block a *database* to everyone temporarily:

```sql
ALTER DATABASE appdb ALLOW_CONNECTIONS false;
-- restore:
ALTER DATABASE appdb ALLOW_CONNECTIONS true;
```

### 15.4 Idle and runaway sessions — automated

```sql
ALTER ROLE ALL IN DATABASE appdb SET idle_in_transaction_session_timeout = '5min';
ALTER ROLE ALL IN DATABASE appdb SET idle_session_timeout = '2h';
```
(Apply stricter limits to application roles; avoid for pgBouncer/server-side pool accounts without testing.)

---

## 16. Auditing and Inventory Queries

### 16.1 Role inventory with risk flags

```sql
SELECT rolname,
       rolcanlogin  AS login,
       rolsuper     AS superuser,
       rolcreaterole AS createrole,
       rolcreatedb  AS createdb,
       rolreplication AS replication,
       rolbypassrls AS bypassrls,
       rolconnlimit AS conn_limit,
       rolvaliduntil
FROM   pg_roles
WHERE  rolname !~ '^pg_'
ORDER  BY rolsuper DESC, rolcreaterole DESC, rolname;
```

### 16.2 Quick "bad smell" checks

```sql
-- Login roles with NO password (OK only if cert/peer/ldap/gss is used for them)
SELECT rolname FROM pg_authid WHERE rolcanlogin AND rolpassword IS NULL;

-- Legacy MD5 hashes
SELECT rolname FROM pg_authid WHERE rolpassword LIKE 'md5%';

-- Login roles that never expire
SELECT rolname FROM pg_roles WHERE rolcanlogin AND rolvaliduntil IS NULL;

-- Expired
SELECT rolname, rolvaliduntil FROM pg_roles WHERE rolvaliduntil < now();

-- Superusers
SELECT rolname FROM pg_roles WHERE rolsuper;

-- Non-superuser roles that can reach the OS / read-all
SELECT m.rolname AS member, r.rolname AS via
FROM   pg_auth_members am
JOIN   pg_roles r ON r.oid = am.roleid
JOIN   pg_roles m ON m.oid = am.member
WHERE  r.rolname IN ('pg_execute_server_program','pg_read_server_files','pg_write_server_files','pg_read_all_data','pg_write_all_data');

-- Roles that can create roles / bypass RLS / replicate
SELECT rolname, rolcreaterole, rolbypassrls, rolreplication
FROM   pg_roles WHERE (rolcreaterole OR rolbypassrls OR rolreplication) AND NOT rolsuper;

-- Tables owned by login roles (should usually be owned by NOLOGIN owners)
SELECT n.nspname, c.relname, r.rolname AS owner
FROM   pg_class c
JOIN   pg_namespace n ON n.oid = c.relnamespace
JOIN   pg_roles r ON r.oid = c.relowner
WHERE  r.rolcanlogin AND c.relkind IN ('r','p') AND n.nspname !~ '^(pg_|information_schema)';

-- PUBLIC privileges that should have been revoked
SELECT datname, datacl FROM pg_database WHERE datacl IS NULL OR datacl::text LIKE '%=Tc/%' OR datacl::text LIKE '%=c/%';
SELECT nspname, nspacl FROM pg_namespace WHERE nspacl::text LIKE '%=UC/%' OR nspacl::text LIKE '%=C/%';

-- SECURITY DEFINER functions (review each)
SELECT n.nspname, p.proname, pg_get_userbyid(p.proowner) AS owner, p.proconfig
FROM   pg_proc p JOIN pg_namespace n ON n.oid = p.pronamespace
WHERE  p.prosecdef AND n.nspname !~ '^(pg_|information_schema)';

-- SECURITY DEFINER functions without a pinned search_path (vulnerable)
SELECT n.nspname, p.proname
FROM   pg_proc p JOIN pg_namespace n ON n.oid = p.pronamespace
WHERE  p.prosecdef AND (p.proconfig IS NULL OR NOT EXISTS (SELECT 1 FROM unnest(p.proconfig) c WHERE c LIKE 'search_path=%'))
  AND  n.nspname !~ '^(pg_|information_schema)';
```

### 16.3 Privilege review per role (information_schema)

```sql
SELECT grantee, table_schema, table_name, string_agg(privilege_type, ', ' ORDER BY privilege_type) AS privs
FROM   information_schema.role_table_grants
WHERE  grantee = 'app_readwrite'
GROUP  BY 1, 2, 3 ORDER BY 2, 3;

SELECT grantee, table_schema, table_name, column_name, privilege_type
FROM   information_schema.column_privileges
WHERE  grantee = 'app_readonly' AND table_schema = 'hr';
```

### 16.4 pg_hba and connection review

```sql
SELECT line_number, type, database, user_name, address, auth_method
FROM   pg_hba_file_rules WHERE error IS NULL ORDER BY line_number;

-- dangerous methods
SELECT * FROM pg_hba_file_rules WHERE auth_method IN ('trust','password','md5');

-- how are people actually connecting
SELECT usename, client_addr, ssl, version AS tls_version, cipher
FROM   pg_stat_ssl s JOIN pg_stat_activity a USING (pid)
WHERE  a.backend_type = 'client backend';

-- Kerberos/GSSAPI sessions
SELECT a.pid, a.usename, g.gss_authenticated, g.principal, g.encrypted
FROM   pg_stat_gssapi g JOIN pg_stat_activity a USING (pid)
WHERE  g.gss_authenticated;
```

### 16.5 pgaudit (statement/object audit)

```bash
# postgresql.conf
shared_preload_libraries = 'pgaudit'          # restart required
pgaudit.log = 'role, ddl'                     # role = GRANT/REVOKE/CREATE ROLE etc.
pgaudit.log_parameter = off                   # avoid writing passwords/values
```
```sql
CREATE EXTENSION pgaudit;

-- audit everything one sensitive user does
ALTER ROLE dba_carol SET pgaudit.log = 'all';

-- object-level auditing: log reads/writes on specific tables
CREATE ROLE auditor_obj NOLOGIN;
ALTER SYSTEM SET pgaudit.role = 'auditor_obj';
SELECT pg_reload_conf();
GRANT SELECT, UPDATE ON hr.employees TO auditor_obj;     -- accesses to these are now logged
```

Minimum native logging set (no extension):

```
log_connections = on
log_disconnections = on
log_statement = 'ddl'          # captures CREATE/ALTER/DROP ROLE, GRANT, REVOKE
```

---

## 17. Backup, Restore and Migration of Roles

Roles are **global objects**; `pg_dump` (single database) does **not** include them.

```bash
# roles + tablespaces only (includes password hashes)
pg_dumpall --globals-only -f globals.sql
# roles only
pg_dumpall --roles-only -f roles.sql
# roles without password hashes (for sharing with vendors/dev)
pg_dumpall --roles-only --no-role-passwords -f roles_nopw.sql

# per-database data (ownership/ACLs are included unless you disable)
pg_dump -Fc -d appdb -f appdb.dump
```

Restore order on a new server:

```bash
psql -f globals.sql postgres                      # 1. roles first
createdb -O appdb_owner appdb                     # 2. (or let pg_restore -C create it)
pg_restore -d appdb --no-owner --role=appdb_owner appdb.dump   # 3. restore under the owner role
```

Options explained:

| Flag | Effect |
|---|---|
| `--no-owner` | Don't emit `ALTER ... OWNER TO`; objects belong to the restoring role (combine with `--role`) |
| `--role=appdb_owner` | Issues `SET ROLE appdb_owner` before restoring |
| `--no-acl` / `-x` | Don't restore GRANT/REVOKE (re-apply your own baseline script) |
| `--no-comments`, `--no-publications`, ... | Skip those object types |

Cross-environment tips:

- Dump **globals** from prod and strip `SUPERUSER` / passwords before loading in dev, or keep a role-bootstrap script in Git instead.
- PostgreSQL 16+ dumps preserve membership `INHERIT/SET/ADMIN` options and grantors; restoring into an older server will not understand them.
- `pg_upgrade` carries roles and ACLs across major versions automatically (as part of the global catalog). After an upgrade from ≤ 14, re-check `public` schema ACLs (§11.1).

---

## 18. Operational Runbooks

### 18.1 Onboard a new application (checklist)

```sql
-- 1. owner + privilege roles (if the application is new)
CREATE ROLE shop_owner NOLOGIN;
CREATE DATABASE shopdb OWNER shop_owner;
\c shopdb
REVOKE ALL ON DATABASE shopdb FROM PUBLIC;
REVOKE ALL ON SCHEMA public FROM PUBLIC;
CREATE ROLE shop_ro NOLOGIN; CREATE ROLE shop_rw NOLOGIN;
GRANT CONNECT ON DATABASE shopdb TO shop_ro;
GRANT shop_ro TO shop_rw;

-- 2. schema and defaults
SET ROLE shop_owner;
CREATE SCHEMA shop;
RESET ROLE;
GRANT USAGE ON SCHEMA shop TO shop_ro;
ALTER DEFAULT PRIVILEGES FOR ROLE shop_owner IN SCHEMA shop GRANT SELECT ON TABLES TO shop_ro;
ALTER DEFAULT PRIVILEGES FOR ROLE shop_owner IN SCHEMA shop GRANT INSERT, UPDATE, DELETE ON TABLES TO shop_rw;
ALTER DEFAULT PRIVILEGES FOR ROLE shop_owner IN SCHEMA shop GRANT USAGE, SELECT, UPDATE ON SEQUENCES TO shop_rw;

-- 3. service accounts with limits
CREATE ROLE svc_shop LOGIN PASSWORD 'ChangeMe_#1' CONNECTION LIMIT 40 IN ROLE shop_rw;
ALTER ROLE svc_shop SET statement_timeout = '30s';
ALTER ROLE svc_shop SET search_path = shop;
```
```
# 4. pg_hba.conf  -> reload -> test from the app subnet
hostssl shopdb svc_shop 10.30.0.0/24 scram-sha-256
```
Test:

```bash
psql "host=db1 dbname=shopdb user=svc_shop sslmode=verify-full" -c "SELECT current_user, current_database();"
```

### 18.2 Password rotation for a service account (zero downtime)

Two-account pattern (blue/green):

```sql
CREATE ROLE svc_shop_b LOGIN PASSWORD 'NewSecret#1' CONNECTION LIMIT 40 IN ROLE shop_rw;
-- switch the app to svc_shop_b, then:
ALTER ROLE svc_shop NOLOGIN;
-- later: DROP OWNED BY svc_shop; DROP ROLE svc_shop;
```
Or in-place with a short overlap handled by the connection pool re-authenticating:

```sql
\password svc_shop
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE usename = 'svc_shop' AND state = 'idle';
```
(Established sessions continue to work after a password change; only new connections need the new password.)

### 18.3 Offboard an employee

See §10.4. Summary: `NOLOGIN` → terminate sessions → `REASSIGN OWNED` + `DROP OWNED` in every database → `DROP ROLE`.

### 18.4 Grant temporary elevated access (with automatic expiry)

```sql
-- give dba_junior extra rights for one change window
GRANT pg_signal_backend TO dba_junior;
-- ... after the window:
REVOKE pg_signal_backend FROM dba_junior;
```
For fully automatic expiry, schedule the REVOKE with `pg_cron`:

```sql
SELECT cron.schedule_in_database('revoke_tmp_access', '0 20 * * *', $$REVOKE pg_signal_backend FROM dba_junior$$, 'postgres');
```
(Requires the `pg_cron` extension.)

### 18.5 Lost / forgotten superuser password

Option 1 — via OS authentication (preferred):

```bash
sudo -u postgres psql                   # works if local peer auth is configured
```
```sql
ALTER ROLE postgres PASSWORD 'NewStrong#Pass9';
```

Option 2 — single-user mode (server must be **stopped**):

```bash
sudo systemctl stop postgresql-17
sudo -u postgres /usr/pgsql-17/bin/postgres --single -D /var/lib/pgsql/17/data postgres
backend> ALTER ROLE postgres PASSWORD 'NewStrong#Pass9';
# Ctrl-D to exit
sudo systemctl start postgresql-17
```

Option 3 — temporarily set `local all postgres trust` in `pg_hba.conf`, reload, change password, **revert** and reload again. Use only on a firewalled host and keep the window minimal.

### 18.6 Locked out by a bad `pg_hba.conf` or login trigger

```bash
# Syntax is validated on reload; invalid lines are ignored and logged. Fix then reload:
sudo -u postgres psql -c "SELECT line_number, error FROM pg_hba_file_rules WHERE error IS NOT NULL;"

# If a PG17 login event trigger is breaking logins:
sudo -u postgres psql "dbname=appdb options='-c event_triggers=off'" -c "DROP EVENT TRIGGER trg_login_window;"
```

### 18.7 Audit sign-off (monthly)

1. Run §16.1–16.4 queries and export results.
2. Remove expired and unused accounts (see `pg_stat_database`/logs for last-login evidence; PostgreSQL has no native "last login" — use log analysis, a login event trigger writing to a table, or pgaudit).
3. Review every `SUPERUSER`, `CREATEROLE`, `REPLICATION`, `BYPASSRLS` holder and every predefined-role grant.
4. Review `pg_hba.conf` diffs from Git.
5. Rotate service-account passwords per policy.

---

## 19. Troubleshooting Common Errors

| Error / symptom | Cause | Fix |
|---|---|---|
| `FATAL: password authentication failed for user "x"` | Wrong password, role expired (`VALID UNTIL`), or password stored as MD5 while hba demands something else | `\password x`; check `rolvaliduntil`; check server log for the real reason (log shows `Role "x" does not exist` / `Password does not match` / `User "x" has an expired password`) |
| `FATAL: no pg_hba.conf entry for host "10.1.1.5", user "x", database "d", SSL encryption` | No matching hba line | Add a line (watch `hostssl` vs `host`, address mask), validate via `pg_hba_file_rules`, `pg_reload_conf()` |
| `FATAL: role "x" is not permitted to log in` | `NOLOGIN` | `ALTER ROLE x LOGIN;` |
| `FATAL: too many connections for role "x"` | `CONNECTION LIMIT` reached | Raise limit or fix pool; check `pg_stat_activity` |
| `FATAL: remaining connection slots are reserved for roles with the SUPERUSER attribute` | `max_connections` exhausted | Terminate idle sessions as superuser; raise limits; use pooling |
| `FATAL: permission denied for database "d"` | Missing `CONNECT` | `GRANT CONNECT ON DATABASE d TO role;` |
| `ERROR: permission denied for schema s` | Missing schema `USAGE` | `GRANT USAGE ON SCHEMA s TO role;` |
| `ERROR: permission denied for table t` | Missing table privilege (or column-level grant only) | `GRANT SELECT ON t ...`; for ORMs check `SELECT *` vs column grants |
| `ERROR: permission denied for sequence t_id_seq` | Inserting into serial without sequence `USAGE` | `GRANT USAGE, SELECT ON SEQUENCE ...` or `ON ALL SEQUENCES` + default privileges |
| `ERROR: permission denied to create role` | Not superuser / no `CREATEROLE` | Use a CREATEROLE role; see §6.4 |
| `ERROR: permission denied to alter role` / `DETAIL: Only roles with the ADMIN option on role "x" may alter this role.` | PG16+ CREATEROLE restriction | `GRANT x TO you WITH ADMIN TRUE` (as superuser) |
| `ERROR: permission denied to grant role "x"` / `Only roles with the ADMIN option...` | Lacking admin option | Same as above |
| `ERROR: must have admin option on role "x"` | Same | Same |
| `ERROR: permission denied to set role "x"` | No `SET` option on membership | `GRANT x TO you WITH SET TRUE` |
| `ERROR: role "x" cannot be dropped because some objects depend on it` | Owns objects / has ACLs | `REASSIGN OWNED` + `DROP OWNED` in **each** database (§5.3) |
| `ERROR: must be owner of table t` | `ALTER`/`DROP`/`GRANT` by non-owner | Use the owner (`SET ROLE appdb_owner`) or make the user a member of it |
| `ERROR: must be superuser to ...` | Operation reserved to superusers (e.g., `CREATE EXTENSION` for untrusted extensions, `COPY ... PROGRAM`) | Use trusted extensions (PG13+: `CREATE EXTENSION` allowed for trusted ones with `CREATE` on db), predefined roles, or a DBA |
| `ERROR: new row violates row-level security policy for table "t"` | RLS `WITH CHECK` failed | Review policies in `pg_policies`; check session vars |
| New tables not readable by app role | Default privileges apply only to objects created by the role named in `FOR ROLE` | Create tables as the owner role, or add defaults for the creator role |
| User can still run queries after `NOLOGIN` | Existing sessions aren't terminated | `pg_terminate_backend` |
| `FATAL: SCRAM authentication requires libpq version 10 or above` | Old client | Upgrade client/driver, or (temporarily) fall back to MD5 hash for that role |
| `psql: error: ... no such role` after restore | Globals not restored first | Restore `pg_dumpall -g` output before database dumps |
| Extension functions fail for readonly users after upgrade to 17 | Safe `search_path` in maintenance (expression index/mat. view function not schema-qualified) | Schema-qualify functions, or `ALTER FUNCTION ... SET search_path = ...` |

Useful debug sessions:

```sql
SELECT current_user, session_user, current_setting('role');
SELECT * FROM pg_roles WHERE rolname = current_user;
\drg
\du+ alice
SELECT has_table_privilege('alice', 'sales.orders', 'SELECT');
```

Enable verbose auth diagnostics in the server log (restart not needed):

```sql
ALTER SYSTEM SET log_connections = on;
ALTER SYSTEM SET log_min_messages = 'warning';
SELECT pg_reload_conf();
```

---

## 20. Best-Practice Checklist

**Identity**
- [ ] No shared logins; one role per human and per service
- [ ] Named DBA superusers; the `postgres` role is restricted to local `peer` auth
- [ ] Groups are `NOLOGIN`; people are members, never direct grantees
- [ ] Owner roles are `NOLOGIN`, non-superuser, and own all schema objects
- [ ] Contractor/temporary accounts have `VALID UNTIL` and `CONNECTION LIMIT`

**Authentication**
- [ ] `password_encryption = scram-sha-256`; zero MD5 verifiers
- [ ] `pg_hba.conf` in version control; no `trust`, no `password`; remote superuser logins rejected
- [ ] TLS enforced (`hostssl` + `hostnossl ... reject`); clients use `sslmode=verify-full`
- [ ] Prefer certificate / Kerberos / LDAP SSO for humans; strong secrets in a vault for services
- [ ] Passwords set via `\password` or pre-hashed SCRAM — never in shell history or logs

**Authorization**
- [ ] `REVOKE ALL ON DATABASE ... FROM PUBLIC`; `public` schema locked down in every database and in `template1`
- [ ] `ALTER DEFAULT PRIVILEGES` configured for every creator role
- [ ] Column-level grants or views for sensitive columns; RLS where row-level separation is needed
- [ ] `pg_execute_server_program`, `pg_read/write_server_files`, `pg_write_all_data` granted to ≤ break-glass users
- [ ] `SECURITY DEFINER` functions have pinned `search_path` and `REVOKE ... FROM PUBLIC`
- [ ] Use `pg_maintain`, `pg_monitor`, parameter privileges instead of superuser for junior DBAs/tools

**Operations**
- [ ] Per-role `statement_timeout`, `idle_in_transaction_session_timeout`, `lock_timeout` for apps
- [ ] `log_connections`, `log_disconnections`, `log_statement = 'ddl'` on; pgaudit for sensitive roles
- [ ] Monthly access review (§18.7); offboarding runbook tested
- [ ] `pg_dumpall --globals-only` included in backups; restore procedure tested
- [ ] Connection capacity planned (`max_connections`, reserved slots, per-role limits, pooler)

---

## 21. Cheat Sheet

```sql
-- CREATE
CREATE ROLE r NOLOGIN;                                  CREATE ROLE u LOGIN PASSWORD '..';
CREATE ROLE u LOGIN PASSWORD '..' IN ROLE r VALID UNTIL '2027-01-01' CONNECTION LIMIT 10;
-- ALTER
ALTER ROLE u NOLOGIN | LOGIN | CREATEDB | NOSUPERUSER | CONNECTION LIMIT 5 | VALID UNTIL '..';
ALTER ROLE u SET statement_timeout = '30s';             ALTER ROLE u RESET ALL;
ALTER ROLE u RENAME TO v;                               \password u
-- MEMBERSHIP
GRANT r TO u;                                           GRANT r TO u WITH INHERIT FALSE, SET TRUE, ADMIN FALSE;
REVOKE r FROM u;                                        REVOKE ADMIN OPTION FOR r FROM u;
SET ROLE r;  RESET ROLE;
-- PRIVILEGES
GRANT CONNECT ON DATABASE d TO r;                       GRANT USAGE ON SCHEMA s TO r;
GRANT SELECT ON ALL TABLES IN SCHEMA s TO r;            GRANT SELECT (c1,c2) ON s.t TO r;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA s TO r; GRANT EXECUTE ON FUNCTION s.f(int) TO r;
GRANT MAINTAIN ON s.t TO r;                             GRANT SET ON PARAMETER p TO r;
REVOKE ALL ON DATABASE d FROM PUBLIC;
ALTER DEFAULT PRIVILEGES FOR ROLE owner IN SCHEMA s GRANT SELECT ON TABLES TO r;
-- OWNERSHIP / DROP
ALTER TABLE s.t OWNER TO owner;
REASSIGN OWNED BY old TO new;   DROP OWNED BY old;   DROP ROLE old;       -- run first two in EACH database
-- PREDEFINED
GRANT pg_monitor | pg_read_all_data | pg_write_all_data | pg_signal_backend | pg_maintain TO u;
-- SESSIONS
SELECT pg_cancel_backend(pid);  SELECT pg_terminate_backend(pid);
-- CHECKS
\du+   \drg   \l+   \dn+   \dp   \ddp   \drds
SELECT has_table_privilege('u','s.t','SELECT');
SELECT * FROM pg_hba_file_rules;   SELECT pg_reload_conf();
```

```bash
createuser -P --login u        dropuser u
pg_dumpall --globals-only > globals.sql
psql "host=h dbname=d user=u sslmode=verify-full require_auth=scram-sha-256"
```

---

## 22. Lab Cleanup

Remove everything created by this guide (run as a superuser; adjust the list to what you actually created).

```sql
-- as: postgres
-- 1. kill sessions and drop the lab databases
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname IN ('appdb','shopdb') AND pid <> pg_backend_pid();
DROP DATABASE IF EXISTS appdb;
DROP DATABASE IF EXISTS shopdb;

-- 2. drop the login and group roles (their objects went with the databases)
DO $$
DECLARE r text;
BEGIN
  FOREACH r IN ARRAY ARRAY[
    'alice','bob','bob_demo','team_lead','intern1','dba_carol','dba_carol_bg','dba_junior',
    'svc_orders','svc_reports','svc_migrate','svc_etl','svc_shop','svc_shop_b',
    'pmm_monitor','backup_logical','backup_physical','replicator','pub_user','sub_admin',
    'etl_loader','auditor','auditor_obj','bi_reader','vendor_jane','dev_ci',
    'hr_admin','hr_staff','corp_users','jdoe','data_team','data_leads','contractors',
    'app_readonly','app_readwrite','appdb_owner','shop_ro','shop_rw','shop_owner',
    'breakglass_admin','maint_job','helpdesk','repl_admin'
  ] LOOP
    IF EXISTS (SELECT 1 FROM pg_roles WHERE rolname = r) THEN
      EXECUTE format('DROP OWNED BY %I', r);     -- clears ACLs in the *current* database (postgres)
      EXECUTE format('DROP ROLE %I', r);
    END IF;
  END LOOP;
END $$;

-- 3. revert server-wide tweaks you may have made
ALTER SYSTEM RESET password_encryption;
SELECT pg_reload_conf();
```

If a role refuses to drop, see §5.3 — it still owns something or holds privileges in another database (e.g., `postgres` or `template1`).

---

*End of guide. Suggested next steps: script §14 as an idempotent baseline in Git, wire §16 queries into a scheduled report, and add a CI check that diffs `pg_hba.conf` and the role inventory.*
