# Module 7: User & Role Management
## Roles & Attributes | Membership & Inheritance | Privileges | Row-Level Security | Predefined Roles | Best Practices

*Datapundit — PostgreSQL DBA Training Series*

---

This is one of the most operationally important modules in the entire series — nearly every real-world PostgreSQL incident involving "wrong person could see/change something" or "application broke after a permissions change" traces back to a gap in this module. It's also an area where PostgreSQL has changed significantly in recent versions (16 and 15 in particular), so this module deliberately calls out what changed and why.

---

## Part A: The Role Concept

### A.1 Everything Is a Role

Since PostgreSQL 8.1, there is **no real distinction between "users" and "groups"** — both are just **roles**. A role that can log in behaves like a "user"; a role that can't is typically used as a "group" to bundle privileges. `CREATE USER` and `CREATE GROUP` still exist purely as convenience syntax.

```sql
CREATE USER alice WITH PASSWORD 'x';     -- shorthand for CREATE ROLE alice LOGIN PASSWORD 'x';
CREATE GROUP analysts;                    -- shorthand for CREATE ROLE analysts NOLOGIN;
CREATE ROLE bob LOGIN PASSWORD 'y';       -- the "real" underlying command
```

```sql
-- Prove it: query the same catalog either way
SELECT rolname, rolcanlogin FROM pg_roles WHERE rolname IN ('alice', 'bob', 'analysts');
```

**Trainer note:** Worth stating outright — `CREATE USER`/`CREATE GROUP` are training wheels. Professional PostgreSQL work almost always uses `CREATE ROLE` directly, since it's one consistent mental model instead of two.

---

## Part B: Creating, Altering, and Dropping Roles

### B.1 CREATE ROLE

```sql
CREATE ROLE app_user WITH
    LOGIN
    PASSWORD 'a_strong_password'
    CONNECTION LIMIT 50
    VALID UNTIL '2027-01-01';
```

### B.2 ALTER ROLE

```sql
-- Change attributes
ALTER ROLE app_user WITH CONNECTION LIMIT 100;

-- Change password
ALTER ROLE app_user WITH PASSWORD 'a_new_password';

-- Rename a role
ALTER ROLE app_user RENAME TO app_service_account;

-- Set a session-level default for a role -- applied every time it connects
ALTER ROLE app_service_account SET statement_timeout = '30s';
ALTER ROLE app_service_account SET search_path = 'app_schema, public';
```

### B.3 DROP ROLE — and Why It Often Fails

```sql
DROP ROLE app_user;
-- ERROR: role "app_user" cannot be dropped because some objects depend on it
```

A role **cannot** be dropped while it still owns objects or holds privileges anywhere in the cluster — this is a deliberate safety mechanism. Part J covers the correct cleanup sequence (`REASSIGN OWNED` + `DROP OWNED`) before `DROP ROLE` will succeed.

### B.4 Listing and Inspecting Roles

```sql
-- All roles and their key attributes
SELECT rolname, rolsuper, rolcreaterole, rolcreatedb, rolcanlogin,
       rolreplication, rolconnlimit, rolvaliduntil
FROM pg_roles
ORDER BY rolname;
```

```
-- Equivalent psql shortcuts
\du            -- list roles and attributes
\du+           -- with descriptions
\drg           -- list role grants (membership) with options -- PG16+
```

---

## Part C: Role Attributes In-Depth

| Attribute | Meaning | DBA Guidance |
|---|---|---|
| `LOGIN` / `NOLOGIN` | Can this role open a connection? | Login roles = people/apps; NOLOGIN roles = privilege containers ("groups") |
| `SUPERUSER` / `NOSUPERUSER` | Bypasses **all** permission checks | Avoid granting broadly — see Part O |
| `CREATEDB` | Can create new databases | Rare to grant outside DBA/automation roles |
| `CREATEROLE` | Can create/alter/drop other roles | **Significantly changed in PostgreSQL 16** — see Part C.1 below |
| `INHERIT` / `NOINHERIT` | Role-level default for whether new memberships automatically inherit privileges | Since PG16 this is really just a *default*; actual inheritance is set per-membership (Part E) |
| `REPLICATION` | Allowed to initiate a replication connection | Needed for streaming replication (Module 8) |
| `BYPASSRLS` | Ignores Row-Level Security policies entirely | Reserve for trusted admin/ETL roles only |
| `CONNECTION LIMIT n` | Max concurrent connections for this role | Useful to cap a noisy application role |
| `PASSWORD` / `VALID UNTIL` | Credential and expiry | See Part D |

### C.1 CREATEROLE — The PostgreSQL 16 Security Fix

Before PG16, a `CREATEROLE` (non-superuser) role could alter or drop **almost any** non-superuser role in the cluster — a well-known privilege-escalation risk (a `CREATEROLE` user could, for example, alter another role's password and log in as them).

**PostgreSQL 16 fixed this:** a `CREATEROLE` user now only has authority over roles it has been explicitly granted **`ADMIN OPTION`** on — which happens automatically for roles it creates itself, but not for pre-existing roles.

```sql
-- On PG16+, creating a role automatically grants the creator ADMIN OPTION on it
CREATE ROLE ops_admin WITH CREATEROLE LOGIN PASSWORD 'x';

-- As ops_admin:
CREATE ROLE etl_service LOGIN PASSWORD 'y';   -- ops_admin now has ADMIN OPTION on etl_service
                                                -- but NOT automatic INHERIT or SET rights on it
```

**Trainer note:** This is a frequent "why did my automation break after upgrading to PG16" support ticket — scripts/Ansible roles/ORM migrations that relied on a `CREATEROLE` account managing pre-existing roles created by someone else may suddenly get `permission denied to grant role`. The fix is an explicit `GRANT ... WITH ADMIN OPTION` from a superuser, not disabling the protection.

---

## Part D: Password Management

```sql
-- Check which hashing scheme new passwords use
SHOW password_encryption;   -- scram-sha-256 is the modern, recommended default

-- Explicitly encrypted (redundant if password_encryption is already scram-sha-256,
-- but good practice to be explicit in scripts)
ALTER ROLE app_user WITH ENCRYPTED PASSWORD 'a_strong_password';

-- Expire a password/account on a specific date -- useful for contractor
-- accounts or scheduled credential rotation
ALTER ROLE contractor_bob VALID UNTIL '2026-12-31';

-- Force a role to have no password at all (login only via non-password
-- methods like peer/cert) -- NOT the same as NOLOGIN
ALTER ROLE service_cert_auth WITH PASSWORD NULL;
```

```sql
-- Find roles with no expiry set at all -- a useful periodic audit query
SELECT rolname, rolvaliduntil
FROM pg_roles
WHERE rolcanlogin = true AND rolvaliduntil IS NULL;
```

---

## Part E: Role Membership & Privilege Inheritance

### E.1 The Basic Idea — Roles as Groups

```sql
CREATE ROLE analysts NOLOGIN;
CREATE ROLE alice LOGIN PASSWORD 'x';
GRANT analysts TO alice;    -- alice is now a "member" of analysts
```

### E.2 INHERIT — Do Members Get Privileges Automatically?

This is where a lot of confusion happens, and it's a genuinely different mechanism **before and after PostgreSQL 16**.

**Before PG16 (and still the default behavior today):** `INHERIT` was a role-level attribute — a role either inherited from *everything* it was a member of, or *nothing*.

**Since PostgreSQL 16:** inheritance can be controlled **per membership grant**, which is far more precise:

```sql
CREATE ROLE joe LOGIN;
CREATE ROLE admin;
CREATE ROLE wheel;
CREATE ROLE island;

GRANT admin  TO joe   WITH INHERIT TRUE;                 -- joe automatically gets admin's privileges
GRANT wheel  TO admin WITH INHERIT FALSE;                -- admin does NOT auto-inherit wheel's privileges
GRANT island TO joe   WITH INHERIT TRUE, SET FALSE;      -- joe inherits island's privileges, but can't SET ROLE island
```

```sql
-- joe: has privileges granted directly, plus admin's, plus island's (via INHERIT)
-- joe does NOT get wheel's privileges -- the chain breaks at the WITH INHERIT FALSE hop
-- joe CANNOT "SET ROLE island" -- that grant explicitly disabled SET
```

### E.3 The SET Option — Explicit Role-Switching Rights

```sql
GRANT reporting_admin TO alice WITH SET TRUE;
```
```sql
-- alice can now temporarily "become" reporting_admin for a session
SET ROLE reporting_admin;
-- ... do work as reporting_admin, objects created here are OWNED by reporting_admin, not alice ...
RESET ROLE;
```

### E.4 The ADMIN Option — Who Can Grant This Role to Others

```sql
GRANT analysts TO alice WITH ADMIN OPTION;
```
Now `alice` can grant/revoke `analysts` membership to/from other roles herself, without needing superuser or `CREATEROLE`.

```sql
-- See exactly what options exist on every membership grant (PG16+)
\drg
```

```sql
-- Or query it directly
SELECT r.rolname AS member, g.rolname AS granted_role,
       m.admin_option, m.inherit_option, m.set_option
FROM pg_auth_members m
JOIN pg_roles r ON m.member = r.oid
JOIN pg_roles g ON m.roleid = g.oid;
```

**Trainer note:** This is worth a dedicated whiteboard walkthrough — `ADMIN` controls *can you grant this role to others*, `INHERIT` controls *do you automatically get its privileges*, `SET` controls *can you explicitly switch into it*. All three are independent, and pre-PG16 muscle memory (where INHERIT was all-or-nothing at the role level) doesn't map cleanly anymore.

---

## Part F: SET ROLE and Session Authorization

```sql
-- Temporarily assume another role's identity/privileges for this session
SET ROLE billing_admin;
SELECT current_user;    -- now shows billing_admin
SELECT session_user;    -- still shows your original login role

RESET ROLE;              -- back to your own privileges

SET ROLE NONE;           -- explicitly drop to session_user's own identity
```

```sql
-- Check whether a role would even be allowed to switch into another one
SELECT pg_has_role('alice', 'reporting_admin', 'USAGE');   -- can alice use it (inherit or SET)?
SELECT pg_has_role('alice', 'reporting_admin', 'MEMBER');  -- is alice a member at all?
```

---

## Part G: Object Privileges — GRANT / REVOKE

### G.1 Privilege Types by Object

| Object | Relevant Privileges |
|---|---|
| Table | `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `REFERENCES`, `TRIGGER` |
| Sequence | `USAGE`, `SELECT`, `UPDATE` |
| Schema | `USAGE` (can "see into" it), `CREATE` (can create objects in it) |
| Database | `CONNECT`, `CREATE`, `TEMPORARY` |
| Function/Procedure | `EXECUTE` |
| Foreign Data Wrapper / Server | `USAGE` |

### G.2 Granting and Revoking

```sql
-- Table-level
GRANT SELECT, INSERT, UPDATE ON orders TO app_user;
REVOKE UPDATE ON orders FROM app_user;

-- Column-level (a genuinely underused, useful feature)
GRANT SELECT (customer_id, order_date) ON orders TO reporting_readonly;
GRANT UPDATE (status) ON orders TO order_fulfillment_service;

-- Schema-level
GRANT USAGE ON SCHEMA app_schema TO app_user;
GRANT CREATE ON SCHEMA app_schema TO app_migrations_role;

-- Database-level
GRANT CONNECT ON DATABASE salesdb TO app_user;

-- All tables currently in a schema (does NOT cover future tables -- see Part I)
GRANT SELECT ON ALL TABLES IN SCHEMA app_schema TO reporting_readonly;

-- Functions
GRANT EXECUTE ON FUNCTION apply_discount(int, numeric) TO billing_service;

-- Sequences (commonly needed alongside INSERT for SERIAL/IDENTITY columns)
GRANT USAGE, SELECT ON SEQUENCE orders_order_id_seq TO app_user;
```

### G.3 CASCADE / RESTRICT on REVOKE

```sql
-- If alice granted bob SELECT on a table, and you revoke alice's own SELECT
-- privilege, what happens to bob's grant (which alice made)?
REVOKE SELECT ON orders FROM alice CASCADE;   -- also revokes grants alice made downstream
REVOKE SELECT ON orders FROM alice RESTRICT;  -- (default) errors out if dependent grants exist
```

### G.4 WITH GRANT OPTION

```sql
-- Let app_admin re-grant this privilege to others, not just hold it
GRANT SELECT ON orders TO app_admin WITH GRANT OPTION;
```

---

## Part H: The PUBLIC Pseudo-Role — and the PostgreSQL 15 Security Change

### H.1 What PUBLIC Means

`PUBLIC` isn't a real role — it's shorthand for "every current and future role." Granting something to `PUBLIC` grants it to everyone, forever, automatically — which is exactly why it deserves caution.

### H.2 The PostgreSQL 15 Change (Important — Affects Every New Cluster)

**Up through PostgreSQL 14**, every new database's `public` schema granted `PUBLIC` both `USAGE` **and** `CREATE` — meaning *any* role that could connect to a database could create tables in it. This was flagged as a security concern (CVE-2018-1058) for years before the default finally changed.

**Since PostgreSQL 15**, new databases no longer grant `CREATE` on the `public` schema to `PUBLIC` — only the database owner (via the new `pg_database_owner` role) can create there by default. `USAGE` is still granted to `PUBLIC` (so existing objects remain visible), just not `CREATE`.

```sql
-- Check current privileges on the public schema
\dn+ public

-- On an upgraded cluster (pg_dump/pg_upgrade preserves OLD behavior), apply
-- the modern secure default explicitly:
ALTER SCHEMA public OWNER TO pg_database_owner;
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
```

**Trainer note:** This is worth calling out explicitly because `pg_upgrade` and `pg_dump`/`restore` **preserve old permissions** — so a cluster upgraded from PG14 to PG17 does **not** automatically get the safer PG15+ default. Only genuinely new databases created on PG15+ get it out of the box. Any DBA doing a major-version upgrade should apply the two commands above manually.

### H.3 Other Default PUBLIC Grants Worth Knowing

```sql
-- PUBLIC also gets CONNECT + TEMPORARY on every new database by default,
-- and USAGE on built-in languages (sql, plpgsql) -- audit and tighten
-- deliberately if you need a genuinely locked-down database
REVOKE CONNECT ON DATABASE salesdb FROM PUBLIC;
GRANT CONNECT ON DATABASE salesdb TO app_user, reporting_readonly;
```

---

## Part I: Default Privileges for Future Objects

`GRANT ... ON ALL TABLES IN SCHEMA` (Part G.2) only affects **existing** objects at the moment you run it. New tables created afterward won't automatically carry those grants — that's what `ALTER DEFAULT PRIVILEGES` is for.

```sql
-- Any table THIS role creates in this schema, in the future, will
-- automatically grant SELECT to reporting_readonly
ALTER DEFAULT PRIVILEGES FOR ROLE app_migrations_role IN SCHEMA app_schema
    GRANT SELECT ON TABLES TO reporting_readonly;

-- Same idea for sequences (commonly forgotten, breaks inserts on new tables)
ALTER DEFAULT PRIVILEGES FOR ROLE app_migrations_role IN SCHEMA app_schema
    GRANT USAGE, SELECT ON SEQUENCES TO app_user;

-- See what default privilege rules are currently configured
\ddp
```

**Trainer note:** The `FOR ROLE` clause matters — default privileges are keyed to *which role creates the object*, not the schema alone. A very common real-world bug: default privileges are set up correctly for the DBA's own role, but the actual migration tool runs as a different service account, so newly created tables silently don't get the intended grants.

---

## Part J: Ownership — and Safely Dropping a Role

### J.1 Ownership Bypasses the Grant System

Every object has exactly one **owner**, who can always alter/drop it regardless of any `GRANT`s — ownership and privilege-granting are two separate mechanisms.

```sql
ALTER TABLE orders OWNER TO app_admin;
```

### J.2 Cleaning Up Before DROP ROLE

```sql
-- Step 1: hand off everything the role OWNS to another role
REASSIGN OWNED BY app_user TO app_admin;

-- Step 2: remove any privileges/default-privilege entries the role still holds
-- (REASSIGN OWNED does not remove privileges GRANTed TO the role on others' objects)
DROP OWNED BY app_user;

-- Step 3: now this succeeds
DROP ROLE app_user;
```

---

## Part K: Row-Level Security (RLS) In Depth

Introduced briefly in Module 1 — here's the full mechanism, since it's fundamentally a role/privilege feature.

### K.1 Enabling RLS

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Even the table owner is subject to RLS unless you also force it off for them --
-- by default, owners bypass RLS; FORCE changes that:
ALTER TABLE orders FORCE ROW LEVEL SECURITY;
```

### K.2 Creating Policies

```sql
-- USING clause: which existing rows can this role see/affect?
CREATE POLICY region_isolation ON orders
    FOR SELECT
    TO sales_role
    USING (region = current_setting('app.current_region'));

-- WITH CHECK clause: what values are allowed on INSERT/UPDATE?
CREATE POLICY region_insert_check ON orders
    FOR INSERT
    TO sales_role
    WITH CHECK (region = current_setting('app.current_region'));
```

### K.3 PERMISSIVE vs. RESTRICTIVE Policies

```sql
-- PERMISSIVE (default): multiple policies are OR'd together -- satisfying any one is enough
CREATE POLICY manager_sees_all ON orders FOR SELECT TO manager_role USING (true);

-- RESTRICTIVE: ANDed with other policies -- narrows access further, doesn't widen it
CREATE POLICY block_archived ON orders AS RESTRICTIVE
    FOR SELECT
    USING (status != 'archived');
```

### K.4 BYPASSRLS

```sql
-- Roles with this attribute skip RLS entirely -- reserve for trusted ETL/admin roles
ALTER ROLE etl_service WITH BYPASSRLS;
```

---

## Part L: Predefined (Built-In) Roles

PostgreSQL ships several pre-created roles granting narrow, specific superuser-adjacent capabilities — use these instead of granting full `SUPERUSER` whenever the need is narrow.

| Role | Grants |
|---|---|
| `pg_read_all_data` | `SELECT` on every table/view/sequence in every database |
| `pg_write_all_data` | `INSERT`/`UPDATE`/`DELETE` on every table in every database |
| `pg_monitor` | Read access to monitoring views/functions (`pg_stat_*`, etc.) — ideal for observability tooling |
| `pg_read_all_settings` | Can view all `postgresql.conf` settings, including ones normally superuser-only to view |
| `pg_read_all_stats` | Access to statistics views without full `pg_monitor` scope |
| `pg_signal_backend` | Can call `pg_cancel_backend()`/`pg_terminate_backend()` on other sessions (not superuser ones) |
| `pg_checkpoint` | Can issue `CHECKPOINT` without being a superuser |
| `pg_create_subscription` | Can create logical replication subscriptions (Module 8) |

```sql
-- A monitoring tool (e.g. a PMM/Prometheus exporter) should use this
-- instead of a superuser account
CREATE ROLE monitoring_exporter LOGIN PASSWORD 'x';
GRANT pg_monitor TO monitoring_exporter;
```

---

## Part M: Common DBA Role Patterns

### M.1 The Read-Only Reporting Role

```sql
CREATE ROLE reporting_readonly NOLOGIN;
GRANT CONNECT ON DATABASE salesdb TO reporting_readonly;
GRANT USAGE ON SCHEMA app_schema TO reporting_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA app_schema TO reporting_readonly;
ALTER DEFAULT PRIVILEGES FOR ROLE app_migrations_role IN SCHEMA app_schema
    GRANT SELECT ON TABLES TO reporting_readonly;

CREATE ROLE bi_tool_user LOGIN PASSWORD 'x';
GRANT reporting_readonly TO bi_tool_user;
```

### M.2 The Application Service Account

```sql
CREATE ROLE app_role NOLOGIN;
GRANT CONNECT ON DATABASE salesdb TO app_role;
GRANT USAGE ON SCHEMA app_schema TO app_role;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app_schema TO app_role;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA app_schema TO app_role;

CREATE ROLE prod_app_user LOGIN PASSWORD 'x' CONNECTION LIMIT 100;
GRANT app_role TO prod_app_user WITH INHERIT TRUE;
```

### M.3 The DBA/Ops Role (Not Superuser)

```sql
CREATE ROLE dba_ops NOLOGIN CREATEDB CREATEROLE;
GRANT pg_monitor TO dba_ops;
GRANT pg_signal_backend TO dba_ops;

CREATE ROLE shri LOGIN PASSWORD 'x';
GRANT dba_ops TO shri WITH INHERIT TRUE, SET TRUE;
```

**Trainer note:** All three patterns follow the same structure deliberately — a `NOLOGIN` "role" role holding the actual privileges, and separate `LOGIN` "person/app" roles that are simply granted membership. This indirection is what makes onboarding/offboarding trivial (`GRANT`/`REVOKE` one membership) instead of re-running a dozen individual `GRANT` statements per person.

---

## Part N: Auditing and Monitoring Roles/Privileges

```sql
-- Full role inventory with attributes
SELECT rolname, rolsuper, rolcreatedb, rolcreaterole, rolcanlogin, rolreplication
FROM pg_roles ORDER BY rolname;

-- Full membership graph, with PG16+ per-grant options
SELECT r.rolname AS member, g.rolname AS granted_role,
       m.admin_option, m.inherit_option, m.set_option
FROM pg_auth_members m
JOIN pg_roles r ON m.member = r.oid
JOIN pg_roles g ON m.roleid = g.oid
ORDER BY member;

-- All table-level grants for a specific role
SELECT table_schema, table_name, privilege_type
FROM information_schema.role_table_grants
WHERE grantee = 'app_user';

-- Any role that still holds SUPERUSER -- review this list regularly
SELECT rolname FROM pg_roles WHERE rolsuper = true;

-- Roles that can log in but have never actually connected recently
-- (cross-reference with pg_stat_activity / your logging over time)
SELECT usename FROM pg_stat_activity;
```

```
-- psql shortcuts worth knowing cold
\du+     -- roles + attributes + descriptions
\dp      -- table/sequence privileges in current schema
\z       -- alias for \dp
\ddp     -- default privileges
\drg     -- role membership grants with options (PG16+)
```

---

## Part O: Security Best Practices Checklist

1. **Never connect applications as `postgres` or any `SUPERUSER` role.** Use narrowly scoped roles (Part M) for every application and every tool.
2. **Use `NOLOGIN` group roles as privilege containers**, and grant membership to `LOGIN` roles — never attach privileges directly to a dozen individual login roles.
3. **Prefer predefined roles (Part L)** — `pg_monitor`, `pg_signal_backend`, etc. — over granting `SUPERUSER` for narrow operational needs.
4. **On PG16+, be deliberate about `INHERIT`/`SET`/`ADMIN`** when granting membership — don't assume old all-or-nothing inheritance behavior.
5. **Explicitly apply the PG15+ secure `public` schema defaults** (`ALTER SCHEMA public OWNER TO pg_database_owner; REVOKE CREATE ON SCHEMA public FROM PUBLIC;`) on any cluster upgraded from an older version — this does **not** happen automatically.
6. **Set `VALID UNTIL` on contractor/temporary accounts**, and audit for login roles with no expiry at all.
7. **Use `ALTER DEFAULT PRIVILEGES`**, not just one-time `GRANT`s, so new objects don't silently end up under-permissioned (or over-permissioned).
8. **Enable and use RLS deliberately** for genuinely row-scoped multi-tenant data, and reserve `BYPASSRLS` for a small, audited set of roles.
9. **Run the Part N audit queries on a schedule**, not just during an incident — role sprawl and forgotten `SUPERUSER` grants are cumulative problems.
10. **Before `DROP ROLE`, always run `REASSIGN OWNED` + `DROP OWNED` first** — don't work around the resulting error by granting the role to something else "temporarily" and forgetting about it.

---

## Discussion Questions / Exercises for Trainees

1. Create a `NOLOGIN` group role, two `LOGIN` roles that are members of it, and demonstrate that revoking the group's privileges immediately affects both members.
2. On a PG16+ instance, reproduce the `INHERIT TRUE` / `INHERIT FALSE` example from Part E.2 yourself and confirm which privileges are actually visible to each role via `pg_has_role()`.
3. Simulate the CREATEROLE change from Part C.1: create a `CREATEROLE` role, have it create a new role, then try to have a *different* `CREATEROLE` role (with no `ADMIN OPTION`) alter that new role's password. Confirm it fails, and explain why that's a feature, not a bug.
4. Check whether your training lab's databases have the PG15+ secure `public` schema default or the old permissive one, and fix it if needed.
5. Build the three role patterns from Part M end-to-end on a scratch database, including onboarding a "new employee" (grant membership) and offboarding them (revoke membership) without touching a single individual table grant.
6. Enable RLS on a test table, write one `PERMISSIVE` and one `RESTRICTIVE` policy, and verify their combined effect matches Part K.3's explanation.

## Key Takeaways

- PostgreSQL has **one unified role system** — "users" and "groups" are just roles with or without `LOGIN`.
- **PostgreSQL 16 fundamentally changed both `CREATEROLE`'s scope (now requires explicit `ADMIN OPTION`) and role membership itself** (`INHERIT`/`SET`/`ADMIN` are now independent, per-grant options, not a single role-level flag) — this is essential, current knowledge, not legacy trivia.
- **PostgreSQL 15 changed the default `public` schema permissions** — `CREATE` is no longer granted to `PUBLIC` by default on new databases, but upgraded clusters keep the old, less secure behavior unless fixed manually.
- `GRANT`/`REVOKE` operate on existing objects only; **`ALTER DEFAULT PRIVILEGES`** is what governs objects created in the future — and it's keyed to the creating role, not just the schema.
- **Ownership and granted privileges are separate systems** — an object's owner always has full control, and a role must be cleaned up via `REASSIGN OWNED`/`DROP OWNED` before `DROP ROLE` will succeed.
- Use **predefined roles** (`pg_monitor`, `pg_signal_backend`, etc.) instead of `SUPERUSER` wherever the actual need is narrow.
- The **group-role + membership pattern** (Part M) is the single most valuable habit to instill — it turns onboarding/offboarding into one `GRANT`/`REVOKE`, not a privilege archaeology exercise.
