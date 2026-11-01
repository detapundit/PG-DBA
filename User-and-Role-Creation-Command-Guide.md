# PostgreSQL Users & Roles — Complete Command Guide, From Scratch
*Datapundit — PostgreSQL DBA Training Series · companion to Module 7 and Module 3A*

This guide gives **every command in order**: owner, database, schema, read-only role, read-write role, privileges, users, memberships, and worked examples of **INHERIT** and **SET**. Every command and every error message below was run on a real server (PostgreSQL 16.15). Syntax is identical on PostgreSQL 17.

> **Needs PostgreSQL 16 or newer** for `WITH INHERIT / SET / ADMIN` on a grant. The notes in Part 6 explain the older behaviour.

---

## 1. The design (read this first)

```
demodb_owner      (NOLOGIN)   owns  ->  database demodb  ->  schema demodb  ->  tables, sequences, functions

demodb_readonly   (NOLOGIN)   can read
demodb_readwrite  (NOLOGIN)   can read + insert/update/delete      (is a member of demodb_readonly)

anitha       -> member of demodb_readonly                          (a read-only user)
ravi         -> member of demodb_readwrite                         (a read-write user)
deploy_user  -> member of demodb_owner, SET only                   (deploys changes, on purpose)
meera, kiran -> member of demodb_readwrite with different INHERIT / SET settings   (the demo)
```

Three rules behind the design:
1. **The owner owns everything and nobody logs in as the owner.** Ownership gives unlimited power over an object, so keep it away from daily users.
2. **Privileges go on `NOLOGIN` group roles, never directly on people.** Giving or removing access to a person is then a single `GRANT` or `REVOKE`.
3. **A person is a `LOGIN` role that is only a member of those group roles.**

---

## 2. What each step does

| Step | Command | Why |
|---|---|---|
| 1 | `CREATE ROLE demodb_owner NOLOGIN` | The owner role |
| 2 | `CREATE DATABASE demodb OWNER demodb_owner` + `REVOKE ALL ... FROM PUBLIC` | The database, closed to everyone by default (`PUBLIC` normally gets `CONNECT` on every new database) |
| 3 | `CREATE SCHEMA demodb AUTHORIZATION demodb_owner` | The schema, same owner |
| 4 | `CREATE ROLE demodb_readonly / demodb_readwrite NOLOGIN` | The privilege roles |
| 5 | `GRANT CONNECT ON DATABASE` + `GRANT USAGE ON SCHEMA` | **Access** to the database and schema — without both, table privileges are useless |
| 6 | `ALTER DEFAULT PRIVILEGES ... FOR ROLE demodb_owner` | What **future** tables, sequences and functions will receive automatically |
| 7 | `SET ROLE demodb_owner; CREATE TABLE ...` | Create objects **as the owner**, so step 6 applies to them |
| 7b | `GRANT ... ON ALL TABLES IN SCHEMA` | One-off fix for tables that **already existed** |
| 8 | `CREATE ROLE anitha LOGIN PASSWORD ...` | The users |
| 9 | `GRANT demodb_readonly TO anitha` | Put each user in a role |
| 10 | `GRANT ... WITH INHERIT ..., SET ...` | The INHERIT / SET / ADMIN switches |
| 11 | `ALTER ROLE ... SET ...` | Per-user defaults |

---

## 3. Part 1 — The setup script (run as `postgres`)

Save as `01_setup.sql` and run `psql -U postgres -f 01_setup.sql`.

```sql
-- =====================================================================
-- 01_setup.sql : create owner, database, schema, roles, users, grants
-- Run as the postgres superuser:   psql -U postgres -f 01_setup.sql
-- Needs PostgreSQL 16+ for the WITH INHERIT / SET / ADMIN grant options
-- =====================================================================

-- STEP 1. The OWNER role. It owns the database, the schema and every object.
--         NOLOGIN: nobody connects as the owner directly.
CREATE ROLE demodb_owner NOLOGIN;

-- STEP 2. The DATABASE, owned by that role, closed to everyone by default.
CREATE DATABASE demodb OWNER demodb_owner;
REVOKE ALL ON DATABASE demodb FROM PUBLIC;       -- removes the default CONNECT + TEMPORARY for everybody

\c demodb

-- STEP 3. The SCHEMA, owned by the same role.
CREATE SCHEMA demodb AUTHORIZATION demodb_owner;
REVOKE CREATE ON SCHEMA public FROM PUBLIC;       -- nobody builds objects in "public" (already the default on new PG15+ databases)

-- STEP 4. The PRIVILEGE roles ("groups"): no login, they only hold privileges.
CREATE ROLE demodb_readonly  NOLOGIN;
CREATE ROLE demodb_readwrite NOLOGIN;
GRANT demodb_readonly TO demodb_readwrite WITH INHERIT TRUE, SET FALSE;   -- read-write = everything read-only has, plus more

-- STEP 5. DATABASE and SCHEMA access for those roles.
GRANT CONNECT ON DATABASE demodb TO demodb_readonly, demodb_readwrite;
GRANT USAGE   ON SCHEMA   demodb   TO demodb_readonly, demodb_readwrite;

-- STEP 6. DEFAULT PRIVILEGES: what every FUTURE object created by demodb_owner receives.
ALTER DEFAULT PRIVILEGES FOR ROLE demodb_owner IN SCHEMA demodb GRANT SELECT                         ON TABLES    TO demodb_readonly;
ALTER DEFAULT PRIVILEGES FOR ROLE demodb_owner IN SCHEMA demodb GRANT INSERT, UPDATE, DELETE         ON TABLES    TO demodb_readwrite;
ALTER DEFAULT PRIVILEGES FOR ROLE demodb_owner IN SCHEMA demodb GRANT USAGE, SELECT                  ON SEQUENCES TO demodb_readwrite;
ALTER DEFAULT PRIVILEGES FOR ROLE demodb_owner IN SCHEMA demodb GRANT EXECUTE                        ON FUNCTIONS TO demodb_readonly, demodb_readwrite;

-- STEP 7. Create the objects AS the owner (default privileges are keyed to the creator).
SET ROLE demodb_owner;
CREATE TABLE demodb.customers (
    customer_id serial PRIMARY KEY,
    name        text NOT NULL,
    city        text
);
CREATE TABLE demodb.orders (
    order_id    serial PRIMARY KEY,
    customer_id int REFERENCES demodb.customers (customer_id),
    amount      numeric(10,2) NOT NULL,
    order_date  date DEFAULT current_date
);
RESET ROLE;

-- STEP 7b. Tables that ALREADY existed before you set default privileges need a one-off GRANT.
--          (Harmless here, because the tables above already received them.)
GRANT SELECT                 ON ALL TABLES    IN SCHEMA demodb TO demodb_readonly;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES    IN SCHEMA demodb TO demodb_readwrite;
GRANT USAGE, SELECT          ON ALL SEQUENCES IN SCHEMA demodb TO demodb_readwrite;

-- STEP 8. The USERS (login roles).  Replace the passwords!
CREATE ROLE anitha      LOGIN PASSWORD 'ChangeMe_Anitha1'  VALID UNTIL '2027-12-31' CONNECTION LIMIT 10;
CREATE ROLE ravi        LOGIN PASSWORD 'ChangeMe_Ravi1'    VALID UNTIL '2027-12-31' CONNECTION LIMIT 10;
CREATE ROLE deploy_user LOGIN PASSWORD 'ChangeMe_Deploy1'  VALID UNTIL '2027-12-31';

-- STEP 9. Put each user in the right role.
GRANT demodb_readonly  TO anitha;                                    -- read-only user
GRANT demodb_readwrite TO ravi;                                      -- read-write user
GRANT demodb_owner     TO deploy_user WITH INHERIT FALSE, SET TRUE;  -- may become the owner, but only on purpose

-- STEP 10. INHERIT vs SET: the two switches on every membership (PostgreSQL 16+)
CREATE ROLE meera LOGIN PASSWORD 'ChangeMe_Meera1' VALID UNTIL '2027-12-31';
CREATE ROLE kiran LOGIN PASSWORD 'ChangeMe_Kiran1' VALID UNTIL '2027-12-31';
CREATE ROLE sunita LOGIN PASSWORD 'ChangeMe_Sunita1' VALID UNTIL '2027-12-31';
GRANT demodb_readwrite TO meera  WITH INHERIT FALSE, SET TRUE;   -- no automatic rights; must run SET ROLE
GRANT demodb_readwrite TO kiran  WITH INHERIT TRUE,  SET FALSE;  -- automatic rights; may NOT run SET ROLE
GRANT demodb_readonly  TO sunita WITH ADMIN OPTION;              -- may add/remove others in demodb_readonly

-- STEP 11. Optional per-user defaults.
ALTER ROLE ravi SET statement_timeout = '30s';
ALTER ROLE anitha IN DATABASE demodb SET search_path = demodb, public;
```

### What the privilege letters mean
In `\dp` / `\ddp` output: `r` = SELECT, `a` = INSERT, `w` = UPDATE, `d` = DELETE, `U` = USAGE, `X` = EXECUTE, `C` = CREATE. So `demodb_readonly=r/...` is read-only and `demodb_readwrite=awd/...` is insert + update + delete.

---

## 4. Part 2 — Connecting as each user

```bash
# Interactive (prompts for the password)
psql -h 127.0.0.1 -U anitha -d demodb

# One-off command with a password (avoid in shared shells)
PGPASSWORD='ChangeMe_Ravi1' psql -h 127.0.0.1 -U ravi -d demodb -c "SELECT current_user"
```

Allow these users in `pg_hba.conf` (first matching line wins), then reload:

```
# TYPE  DATABASE  USER                                      ADDRESS        METHOD
host    demodb   anitha,ravi,meera,kiran,sunita,deploy_user  10.0.0.0/24    scram-sha-256
```
```sql
SELECT pg_reload_conf();
SELECT line_number, type, database, user_name, address, auth_method, error FROM pg_hba_file_rules;
```

Change a password later:
```sql
ALTER ROLE ravi WITH PASSWORD 'A_New_Strong_Password';
```

---

## 5. Part 3 — Prove it works

Save as `02_verify_demo.sql` and run `psql -U postgres -d demodb -f 02_verify_demo.sql`. It uses `SET SESSION AUTHORIZATION user`, which makes a superuser behave **exactly** as that login user (plain `SET ROLE` is checked against *your* privileges, so a superuser could become anyone).

```sql
-- =====================================================================
-- 02_verify_demo.sql : prove that every role behaves as designed
-- Run as postgres:   psql -U postgres -d demodb -f 02_verify_demo.sql
-- (SET SESSION AUTHORIZATION = "behave exactly as this login user"; superuser only)
-- Lines marked  -- EXPECT  show what you should see.
-- =====================================================================
\set ON_ERROR_STOP off
INSERT INTO demodb.customers (name, city) VALUES ('Seed Customer', 'Mangalore');   -- sample row (we are superuser here)

\echo '--- anitha (read-only)'
SET SESSION AUTHORIZATION anitha;
SELECT count(*) FROM demodb.customers;                                  -- EXPECT: works
INSERT INTO demodb.customers (name) VALUES ('x');                       -- EXPECT: ERROR permission denied for table customers
RESET SESSION AUTHORIZATION;

\echo '--- ravi (read-write)'
SET SESSION AUTHORIZATION ravi;
INSERT INTO demodb.customers (name, city) VALUES ('Ravi Test', 'Udupi'); -- EXPECT: INSERT 0 1 (sequence works too)
UPDATE demodb.customers SET city = 'Udupi' WHERE name = 'Ravi Test';     -- EXPECT: UPDATE 1
DELETE FROM demodb.customers WHERE name = 'Ravi Test';                   -- EXPECT: DELETE 1
CREATE TABLE demodb.illegal (i int);                                     -- EXPECT: ERROR permission denied for schema demodb
RESET SESSION AUTHORIZATION;

\echo '--- meera (INHERIT FALSE, SET TRUE): nothing automatic, must SET ROLE'
SET SESSION AUTHORIZATION meera;
INSERT INTO demodb.customers (name) VALUES ('m1');                       -- EXPECT: ERROR permission denied for schema demodb (not even USAGE was inherited)
SET ROLE demodb_readwrite;
SELECT session_user, current_user;                                      -- EXPECT: meera | demodb_readwrite
INSERT INTO demodb.customers (name) VALUES ('m2');                       -- EXPECT: INSERT 0 1
RESET ROLE;
RESET SESSION AUTHORIZATION;

\echo '--- kiran (INHERIT TRUE, SET FALSE): automatic, cannot SET ROLE'
SET SESSION AUTHORIZATION kiran;
INSERT INTO demodb.customers (name) VALUES ('k1');                       -- EXPECT: INSERT 0 1
SET ROLE demodb_readwrite;                                               -- EXPECT: ERROR permission denied to set role "demodb_readwrite"
RESET SESSION AUTHORIZATION;

\echo '--- deploy_user (member of owner with INHERIT FALSE, SET TRUE)'
SET SESSION AUTHORIZATION deploy_user;
CREATE TABLE demodb.t_new (i int);                                       -- EXPECT: ERROR permission denied for schema demodb (nothing inherited)
SET ROLE demodb_owner;
CREATE TABLE demodb.t_new (i int);                                       -- EXPECT: CREATE TABLE (owned by demodb_owner)
RESET ROLE;
RESET SESSION AUTHORIZATION;
SET SESSION AUTHORIZATION anitha;
SELECT count(*) FROM demodb.t_new;                                       -- EXPECT: 0  (default privileges gave anitha SELECT automatically)
RESET SESSION AUTHORIZATION;

\echo '--- sunita (ADMIN OPTION): can pass the role on'
SET SESSION AUTHORIZATION sunita;
GRANT demodb_readonly TO kiran;                                          -- EXPECT: GRANT ROLE
REVOKE demodb_readonly FROM kiran;                                       -- EXPECT: REVOKE ROLE
GRANT demodb_readwrite TO kiran;                                         -- EXPECT: ERROR (she has no ADMIN on demodb_readwrite)
RESET SESSION AUTHORIZATION;

\echo '--- an outsider cannot even connect'
CREATE ROLE outsider LOGIN;
SELECT has_database_privilege('outsider', 'demodb', 'CONNECT');        -- EXPECT: f
DROP ROLE outsider;

\echo '--- catalog views of the result'
SELECT m.rolname AS member, g.rolname AS member_of, am.admin_option, am.inherit_option, am.set_option
FROM pg_auth_members am JOIN pg_roles m ON m.oid = am.member JOIN pg_roles g ON g.oid = am.roleid
WHERE g.rolname LIKE 'demodb\_%' ORDER BY 1, 2;
\dn+ demodb
\ddp
\dp demodb.*
```

**Real output of the key lines (PostgreSQL 16.15):**

```
anitha  INSERT           ->  ERROR:  permission denied for table customers
ravi    CREATE TABLE     ->  ERROR:  permission denied for schema demodb
meera   INSERT (no SET)  ->  ERROR:  permission denied for schema demodb
meera   after SET ROLE   ->  session_user = meera | current_user = demodb_readwrite   then INSERT 0 1
kiran   INSERT           ->  INSERT 0 1
kiran   SET ROLE         ->  ERROR:  permission denied to set role "demodb_readwrite"
deploy_user CREATE TABLE ->  ERROR:  permission denied for schema demodb
deploy_user after SET ROLE demodb_owner -> CREATE TABLE
sunita  GRANT demodb_readwrite TO kiran ->
        ERROR:  permission denied to grant role "demodb_readwrite"
        DETAIL:  Only roles with the ADMIN option on role "demodb_readwrite" may grant this role.
```
```
   member    |    member_of    | admin_option | inherit_option | set_option
-------------+-----------------+--------------+----------------+-----------
 anitha      | demodb_readonly  | f            | t              | t
 deploy_user | demodb_owner     | f            | f              | t
 kiran       | demodb_readwrite | f            | t              | f
 meera       | demodb_readwrite | f            | f              | t
 ravi        | demodb_readwrite | f            | t              | t
 sunita      | demodb_readonly  | t            | t              | t
 demodb_readwrite | demodb_readonly | f         | t              | f
```

---

## 6. Part 4 — INHERIT and SET explained

Every membership (`GRANT role TO user`) has **three independent switches**:

| Switch | Question it answers |
|---|---|
| **INHERIT** | Does the user **automatically** have the role's privileges? |
| **SET** | May the user **switch into** the role with `SET ROLE`? |
| **ADMIN** | May the user **grant or revoke** this role for other people? |

| Grant | Automatic privileges? | `SET ROLE` allowed? | Typical use | Example in this guide |
|---|---|---|---|---|
| plain `GRANT r TO u` | yes | yes | Normal group membership | `anitha`, `ravi` |
| `WITH INHERIT TRUE, SET FALSE` | yes | **no** | Ordinary user who must *never become* the group, so nothing they create is owned by it | `kiran` |
| `WITH INHERIT FALSE, SET TRUE` | **no** | yes | Powerful role you should use **deliberately**, never by accident | `meera`, `deploy_user` |
| `WITH INHERIT FALSE, SET FALSE` | no | no | Membership kept only for `ADMIN` or bookkeeping | — |
| `WITH ADMIN OPTION` | (as above) | (as above) | Team lead who onboards people | `sunita` |

### Examples, one by one

```sql
-- INHERIT TRUE, SET FALSE : works straight away, cannot switch role
GRANT demodb_readwrite TO kiran WITH INHERIT TRUE, SET FALSE;
--   kiran: INSERT works                       SET ROLE demodb_readwrite  -> permission denied

-- INHERIT FALSE, SET TRUE : nothing works until you switch on purpose
GRANT demodb_readwrite TO meera WITH INHERIT FALSE, SET TRUE;
--   meera: INSERT fails (not even schema USAGE)
--   meera: SET ROLE demodb_readwrite;  INSERT works;   RESET ROLE;  -> back to nothing

-- Deployer: no standing power, but can become the owner when needed
GRANT demodb_owner TO deploy_user WITH INHERIT FALSE, SET TRUE;
--   deploy_user:  SET ROLE demodb_owner;  CREATE TABLE ...;  RESET ROLE;

-- Team lead who can pass demodb_readonly on to others
GRANT demodb_readonly TO sunita WITH ADMIN OPTION;
--   sunita:  GRANT demodb_readonly TO kiran;   works
--   sunita:  GRANT demodb_readwrite TO kiran;  fails (no ADMIN on that role)
```

**Why this matters for ownership.** Objects belong to the role that was **current when they were created**. After `SET ROLE demodb_owner`, a new table is owned by `demodb_owner` and receives the default privileges from step 6 — so `anitha` can read it immediately (see the `t_new` check in the demo). A table created as plain `deploy_user`, or as `postgres`, is owned by *them* and gets **no** default privileges.

**`session_user` vs `current_user`.** `session_user` is who logged in; `current_user` is the role whose rights apply right now. `SET ROLE` changes only `current_user`.

**Changing a switch later:** grant again with the new options (on PG16+ this updates the existing grant), or revoke and re-grant:
```sql
REVOKE demodb_readwrite FROM meera;
GRANT  demodb_readwrite TO meera WITH INHERIT TRUE, SET TRUE;
```

**Before PostgreSQL 16.** There was **no `SET` switch** (any member could `SET ROLE`), and inheritance was a **per-user attribute** instead of per-grant: `ALTER ROLE meera NOINHERIT;` turned inheritance off for **all** of that user's memberships. The `WITH INHERIT / SET` syntax above does not exist there.

---

## 7. Part 5 — Changing and removing access

Save as `03_remove_and_cleanup.sql`.

```sql
-- =====================================================================
-- 03_remove_and_cleanup.sql : take users away, and tear the whole demo down
-- Run as postgres:   psql -U postgres -d demodb -f 03_remove_and_cleanup.sql
-- =====================================================================

-- A. Change a user's access (move anitha from read-only to read-write)
REVOKE demodb_readonly  FROM anitha;
GRANT  demodb_readwrite TO   anitha;

-- B. Block a user from logging in but keep the account
ALTER ROLE anitha NOLOGIN;

-- C. Remove a user completely (run this in EVERY database where the user owns or holds anything)
REASSIGN OWNED BY anitha TO demodb_owner;
DROP OWNED BY anitha;
DROP ROLE anitha;

-- D. Tear down the whole demo
\c postgres
DROP DATABASE demodb;
DROP ROLE ravi, deploy_user, meera, kiran, sunita;
DROP ROLE demodb_readwrite, demodb_readonly, demodb_owner;
```

Notes:
- **`REVOKE role FROM user`** removes access immediately for new statements; already-open sessions lose the privileges on their next statement.
- **`ALTER ROLE ... NOLOGIN`** stops new logins but does not end existing sessions. To end them: `SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE usename = 'anitha';`
- **A role is cluster-wide, but `REASSIGN OWNED` / `DROP OWNED` work on one database at a time.** Repeat step C in every database where the user owns anything, or `DROP ROLE` fails and tells you which database.

---

## 8. Cheat sheet — look at what you built

| Question | Command |
|---|---|
| List roles and attributes | `\du+` |
| Who is a member of what, with options (16+) | `\drg` |
| Databases, owners, who may connect | `\l` |
| Schemas, owners, schema privileges | `\dn+` |
| Table / sequence privileges | `\dp demodb.*` |
| Default privileges for future objects | `\ddp` |
| Can this user do that? | `SELECT has_table_privilege('anitha','demodb.orders','SELECT');` · `has_schema_privilege(...)` · `has_database_privilege(...)` · `pg_has_role('ravi','demodb_readonly','USAGE')` |
| All superusers (review often) | `SELECT rolname FROM pg_roles WHERE rolsuper;` |
| Login roles with no expiry | `SELECT rolname FROM pg_roles WHERE rolcanlogin AND rolvaliduntil IS NULL;` |

---

## 9. Common errors and what they mean

| Message | Cause | Fix |
|---|---|---|
| `permission denied for schema demodb` | No `USAGE` on the schema — or a member who does **not** inherit | `GRANT USAGE ON SCHEMA`; or check the grant's `INHERIT` option / use `SET ROLE` |
| `permission denied for table orders` | Has schema access, lacks the table privilege | `GRANT SELECT/INSERT/... ON TABLE` |
| `permission denied for sequence ..._seq` | `INSERT` into a `serial` column needs the sequence | `GRANT USAGE, SELECT ON SEQUENCE` (or defaults, step 6) |
| `permission denied to set role "x"` | The membership has `SET FALSE` (or none) | Grant with `SET TRUE` |
| `permission denied to grant role "x"` | No `ADMIN` option on that role | `GRANT x TO you WITH ADMIN OPTION` (as someone who can) |
| `must be owner of table x` | `ALTER`/`DROP` by a non-owner | Act as the owner (`SET ROLE demodb_owner`) |
| `role "x" cannot be dropped because some objects depend on it` | Still owns objects or holds privileges | `REASSIGN OWNED` + `DROP OWNED`, in each database named |
| New table unreadable by `demodb_readonly` | Created by someone **other** than the role named in `ALTER DEFAULT PRIVILEGES ... FOR ROLE` (often `postgres`) | Create it as the owner, or `ALTER TABLE ... OWNER TO demodb_owner` and run `GRANT SELECT ON ALL TABLES IN SCHEMA` |
| `no pg_hba.conf entry for host ...` / `password authentication failed` | Authentication, not privileges | Fix `pg_hba.conf` / password, then reload |

---

## 10. Rules of thumb

1. **One owner role, `NOLOGIN`.** Create objects through it, never as `postgres`.
2. **Privileges on group roles, people only as members.**
3. **Both layers are needed:** `CONNECT` on the database **and** `USAGE` on the schema, **plus** the object privilege.
4. **Set default privileges before you create tables**, and remember they follow the **creating role**.
5. **Use `WITH INHERIT FALSE, SET TRUE` for powerful roles** (owner, DBA) so they are used on purpose.
6. **Never make an application role a superuser.** If something fails with a permission error, fix the specific privilege.
7. **Give every login role an expiry and a password stored as SCRAM** (`SHOW password_encryption;`).
8. **Review regularly:** superusers, login roles without expiry, and the membership list in Part 3.
