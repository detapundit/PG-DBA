# Module 3A: Schemas in PostgreSQL
## What a Schema Is | Schema vs. Database | search_path | Privileges | Design Patterns | PostgreSQL vs. MySQL

*Datapundit — PostgreSQL DBA Training Series*

---

"Schema" is one of the most overloaded words in database work. In Oracle it means "a user's set of objects." In generic data-modeling talk it means "the structure of the tables." In MySQL it is literally another word for "database." In PostgreSQL it means something specific and different from all three: **a namespace *inside* a database.**

Getting this wrong causes real problems — people migrating from MySQL create a PostgreSQL *database* per application when they should have created a *schema*, or the reverse, and then discover they can't join across databases. This module fixes that mental model.

**Suggested placement:** this fits naturally right after Module 3 (Architecture), since it extends the object hierarchy introduced there, and before Module 7 (User & Role Management), which assumes you already know what a schema is.

---

## Part A: The PostgreSQL Object Hierarchy

PostgreSQL has **three nesting levels** above a table, not two:

```
Cluster  (one postmaster, one PGDATA directory, one port, e.g. 5432)
│
├── Cluster-wide objects:  roles, tablespaces, the list of databases
│
├── Database: postgres
├── Database: template0 / template1   (templates used by CREATE DATABASE)
│
└── Database: salesdb
    │
    ├── Schema: public
    │   └── table: customers
    │
    ├── Schema: sales
    │   ├── table: orders
    │   ├── view:  monthly_totals
    │   └── function: apply_discount()
    │
    └── Schema: hr
        └── table: employees
```

| Level | Examples of what lives here |
|---|---|
| **Cluster** | Roles, tablespaces, the databases themselves, replication slots |
| **Database** | Schemas, extensions, publications, event triggers, per-database settings, encoding and collation |
| **Schema** | Tables, views, materialized views, sequences, indexes, functions, procedures, types, domains, operators |

**An analogy that works well in training:** the cluster is a building, each database is a **separate locked apartment**, and each schema is a **room** inside an apartment. You can walk freely between rooms in your apartment; to reach another apartment you have to leave and knock on a different door.

---

## Part B: What Is a Schema?

A **schema is a named namespace that groups database objects inside a single database.** Two schemas can each contain a table called `orders` without any conflict, because the full names differ: `sales.orders` and `staging.orders`.

```sql
CREATE SCHEMA sales;
CREATE SCHEMA staging;

CREATE TABLE sales.orders   (order_id int, amount numeric);
CREATE TABLE staging.orders (order_id int, amount numeric);   -- same table name, no conflict
```

### B.1 Fully Qualified Names

```
[database.]schema.object
```

The `database.` prefix is accepted **only if it names the database you're currently connected to** — it's not a way to reach another database (Part C.2 shows the error).

```sql
SELECT * FROM sales.orders;                  -- schema-qualified: the normal form
SELECT * FROM salesdb.sales.orders;          -- works only while connected to salesdb
```

### B.2 Why Use Schemas?

| Purpose | Example |
|---|---|
| **Organize** a large database by function | `sales`, `hr`, `finance`, `reporting` |
| **Avoid name collisions** | Two modules both need a `settings` table |
| **Permission boundaries** | Grant a role access to `reporting` only |
| **Separate lifecycle stages** | `staging` (raw loads) vs `core` (cleaned) |
| **Multi-tenancy** | One schema per customer (see Part G) |
| **Isolate third-party code** | Install extensions into an `extensions` schema |
| **Versioned APIs** | `api_v1`, `api_v2` exposing views over the same data |

### B.3 Built-In Schemas

Every database ships with several schemas you did not create:

| Schema | Purpose |
|---|---|
| `public` | Default schema for new objects; exists in every new database |
| `pg_catalog` | The system catalogs (`pg_class`, `pg_roles`, ...) and all built-in functions and types |
| `information_schema` | SQL-standard views over the catalogs |
| `pg_toast` | Storage for oversized column values (TOAST tables) |
| `pg_temp_N` | A per-session schema holding that session's temporary tables |

```sql
\dn        -- user schemas only
\dnS       -- include system schemas
```

---

## Part C: Schema vs. Database — The Differences That Matter

### C.1 The Core Distinction

A **database** is a hard boundary: a connection belongs to exactly one database and can only see objects inside it. A **schema** is a soft boundary: a naming and permission grouping *within* one database, freely crossable in a single query.

### C.2 Side-by-Side Comparison

| Aspect | Database | Schema |
|---|---|---|
| **What it is** | An isolated container with its own catalogs | A namespace inside one database |
| **Connection scope** | A connection is bound to **one** database | One connection sees **all** schemas in its database |
| **Switching** | `\c other_db` opens a **new connection** (new backend process) | `SET search_path` just changes name lookup; no reconnect |
| **Cross-querying** | **Not possible** in plain SQL; needs `dblink` or `postgres_fdw` | A plain `JOIN` across schemas just works |
| **Foreign keys** | Cannot reference a table in another database | Can reference a table in another schema |
| **Transactions** | One transaction lives in one database | One transaction can touch many schemas |
| **Encoding / locale / collation** | Fixed **per database** at `CREATE DATABASE` time | Inherited from the database; cannot differ per schema |
| **Extensions** | Installed per database | Extension objects are placed into a chosen schema |
| **Catalogs** | Each database has **its own** copy of `pg_class`, `pg_attribute`, etc. | All schemas in a database share one set of catalogs |
| **Privileges** | `CONNECT`, `CREATE`, `TEMPORARY` | `USAGE`, `CREATE` |
| **`pg_hba.conf`** | Rules match on database name | Rules **cannot** match on schema |
| **Per-object settings** | `ALTER DATABASE ... SET ...`, `CONNECTION LIMIT` | None — there is no per-schema setting or limit |
| **Backup (logical)** | `pg_dump -d dbname` | `pg_dump -n schema` selects schemas within a database |
| **Backup (physical)** | Whole cluster only | Whole cluster only |
| **Logical replication** | Publications are defined within one database | `CREATE PUBLICATION ... FOR TABLES IN SCHEMA` (PostgreSQL 15+) |
| **Moving objects** | Cannot move a table to another database; dump/restore or FDW | `ALTER TABLE ... SET SCHEMA` is instant and transactional |
| **Drop** | `DROP DATABASE` (must not be connected to it) | `DROP SCHEMA ... CASCADE` |
| **On disk** | A directory under `base/<database_oid>/` | **No directory at all** — see Lab 6 |

### C.3 The Cross-Database Limitation, Demonstrated

```sql
-- Connected to database "schema_lab"
SELECT * FROM schema_lab_b.public.remote_items;
-- ERROR:  cross-database references are not implemented: schema_lab_b.public.remote_items
```

The workarounds all involve extra machinery:

```sql
-- Option 1: dblink (contrib module) -- you must declare the result column types
CREATE EXTENSION IF NOT EXISTS dblink;
SELECT * FROM dblink('dbname=schema_lab_b',
                     'SELECT item_id, item_name FROM public.remote_items')
       AS r(item_id int, item_name text);

-- Option 2: postgres_fdw (contrib module) -- maps remote tables as foreign tables
CREATE EXTENSION IF NOT EXISTS postgres_fdw;
```

### C.4 What This Means for a DBA

- **Resource overhead:** every database carries its own catalog tables, and autovacuum works through databases one at a time. Hundreds of small databases are measurably heavier than hundreds of schemas in one database.
- **Isolation strength:** a role that cannot `CONNECT` to a database cannot see anything in it, including its catalogs. A role blocked only at the schema level can still see schema *names* through the catalogs (see `\dn` and `pg_namespace`).
- **Irreversible choices:** encoding and locale are database-level and effectively permanent (Module 2, Part B.4). If two applications genuinely need different encodings or collations, that alone justifies separate databases.

---

## Part D: Working with Schemas — Command Reference

### D.1 Create

```sql
CREATE SCHEMA sales;
CREATE SCHEMA IF NOT EXISTS sales;

-- Create a schema owned by a specific role
CREATE SCHEMA hr AUTHORIZATION hr_admin;

-- AUTHORIZATION alone: schema name defaults to the role name
CREATE SCHEMA AUTHORIZATION alice;

-- Create a schema and some objects in one atomic statement
CREATE SCHEMA reporting
    CREATE TABLE daily_totals (day date, total numeric)
    CREATE VIEW  recent_totals AS SELECT * FROM daily_totals WHERE day > now() - interval '30 days';
```

### D.2 Alter and Drop

```sql
ALTER SCHEMA sales RENAME TO sales_v1;
ALTER SCHEMA sales OWNER TO sales_admin;

DROP SCHEMA staging;            -- fails if the schema still contains objects (RESTRICT is the default)
DROP SCHEMA staging CASCADE;    -- drops the schema AND everything in it -- use with care
```

### D.3 Move Objects Between Schemas

```sql
ALTER TABLE staging.orders SET SCHEMA sales;
ALTER FUNCTION staging.calc() SET SCHEMA sales;
```

### D.4 Inspect

```sql
-- psql shortcuts
\dn                  -- list schemas
\dn+                 -- with owner, privileges, description
\dt sales.*          -- tables in one schema
\dt *.*              -- tables in every schema
\df sales.*          -- functions in a schema

-- Catalog queries
SELECT nspname, pg_get_userbyid(nspowner) AS owner
FROM pg_namespace
WHERE nspname NOT LIKE 'pg\_%' AND nspname <> 'information_schema'
ORDER BY nspname;

SELECT schemaname, count(*) AS tables
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
GROUP BY schemaname
ORDER BY schemaname;
```

### D.5 Size of a Schema

There is no built-in `pg_schema_size()`, so sum it from the catalogs. `pg_total_relation_size` already includes a table's indexes and TOAST data.

```sql
SELECT n.nspname AS schema,
       pg_size_pretty(sum(pg_total_relation_size(c.oid))) AS total_size
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind IN ('r', 'm')                         -- tables and materialized views
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
  AND n.nspname NOT LIKE 'pg\_toast%'
GROUP BY n.nspname
ORDER BY sum(pg_total_relation_size(c.oid)) DESC;
```

---

## Part E: search_path — How Unqualified Names Are Resolved

### E.1 The Rule

When you write `SELECT * FROM orders` without a schema, PostgreSQL walks the **`search_path`** list left to right and uses the first schema that contains a matching object.

```sql
SHOW search_path;
--    search_path
-- -----------------
--  "$user", public
```

The default, `"$user", public`, means: *first look in a schema named after the current role (if one exists), then in `public`.*

### E.2 Two Implicit Rules

1. **`pg_catalog` is always searched**, and it is searched *first* unless you list it explicitly. This is why `count()` and `int4` always resolve to the built-ins.
2. **`pg_temp`** (your session's temporary schema) is searched first for tables and views — but never for functions or operators.

### E.3 Where New Objects Are Created

An unqualified `CREATE TABLE t (...)` goes into the **first schema in `search_path` that exists and that you may create in** — typically `public`. If none qualifies, you get an error.

```sql
SELECT current_schema();         -- where an unqualified CREATE would land
SELECT current_schemas(false);   -- the effective path, excluding implicit system schemas
SELECT current_schemas(true);    -- including implicit pg_catalog / pg_temp
```

### E.4 Setting search_path at Different Levels

```sql
-- This session only
SET search_path = sales, public;

-- Every new connection by one role
ALTER ROLE app_user SET search_path = sales, public;

-- Every new connection to one database
ALTER DATABASE salesdb SET search_path = sales, public;

-- A specific role connecting to a specific database
ALTER ROLE app_user IN DATABASE salesdb SET search_path = sales, public;
```

**Precedence, highest to lowest:** session `SET` → `ALTER ROLE ... IN DATABASE` → `ALTER ROLE` → `ALTER DATABASE` → `postgresql.conf`. Role and database settings take effect only for *new* sessions.

### E.5 Security: search_path Is an Attack Surface

If an attacker can create an object in a schema that appears *earlier* in someone's `search_path`, they can shadow a legitimate object — a classic way to hijack a function or table reference run by a more privileged user (this class of problem was publicly tracked as CVE-2018-1058).

Defensive habits for DBAs:

```sql
-- 1. Functions that run with elevated rights must pin their own search_path,
--    with pg_temp LAST so session-temporary objects can't shadow anything
CREATE FUNCTION sales.refund(p_id int) RETURNS void
LANGUAGE plpgsql SECURITY DEFINER
SET search_path = sales, pg_temp
AS $$ BEGIN UPDATE sales.orders SET amount = 0 WHERE order_id = p_id; END $$;

-- 2. Don't let ordinary roles create objects in public
--    (the default for NEW databases since PostgreSQL 15; upgraded clusters
--     keep the old behavior until you fix it -- see Module 7, Part H)
REVOKE CREATE ON SCHEMA public FROM PUBLIC;

-- 3. Prefer schema-qualified names in application code and migrations
```

---

## Part F: Privileges on Schemas

To use an object you need **two** things: the right privilege on the object **and** `USAGE` on the schema containing it.

| Schema privilege | Allows |
|---|---|
| `USAGE` | Look up objects inside the schema (necessary to touch anything in it) |
| `CREATE` | Create new objects in the schema |

```sql
GRANT USAGE ON SCHEMA sales TO analyst;                       -- can now "see into" sales
GRANT SELECT ON ALL TABLES IN SCHEMA sales TO analyst;        -- existing tables only
ALTER DEFAULT PRIVILEGES IN SCHEMA sales
    GRANT SELECT ON TABLES TO analyst;                        -- tables created later by the current role

REVOKE ALL ON SCHEMA hr FROM PUBLIC;                          -- lock a schema down
```

The two distinct errors are worth recognizing instantly:

```
ERROR:  permission denied for schema sales      -- missing USAGE on the schema
ERROR:  permission denied for table orders      -- has USAGE, missing SELECT on the table
```

**Trainer note:** a very common support ticket is "I granted SELECT on the table and it still says permission denied" — nearly always a missing schema-level `USAGE`. The full privilege model is in Module 7.

---

## Part G: Design Patterns — Schema or Separate Database?

### G.1 Common Schema Patterns

| Pattern | Layout |
|---|---|
| **Functional split** | `sales`, `hr`, `finance` in one database |
| **ETL stages** | `staging` → `core` → `reporting` |
| **Extension isolation** | `CREATE EXTENSION pg_stat_statements SCHEMA extensions;` |
| **API layer** | `api` schema holds only views/functions; `internal` holds the real tables; app roles only get `USAGE` on `api` |
| **Per-user sandboxes** | A schema named after each role, picked up automatically by `"$user"` in `search_path` |

### G.2 Multi-Tenancy: Three Ways to Do It

| Approach | Isolation | Operational cost | Typical fit |
|---|---|---|---|
| **Shared tables + `tenant_id` + Row-Level Security** | Logical (policy-enforced) | Lowest; one set of tables to migrate | Thousands to millions of small tenants |
| **Schema per tenant** | Medium (privilege-enforced) | Migrations must run N times; catalogs grow with tenant count; `pg_dump` slows down | Tens to a few thousand tenants; per-tenant restore/drop matters |
| **Database per tenant** | Strongest | Highest; connection pooling per database, per-database autovacuum, more catalog overhead | Few, large tenants; hard isolation or per-tenant encoding/extensions |

Schema-per-tenant's big wins are cheap per-tenant operations — `pg_dump -n tenant_42` to export, `DROP SCHEMA tenant_42 CASCADE` to offboard. Its costs show up at scale: a schema migration becomes a loop over every schema, and a very large number of schemas means a very large number of catalog rows to vacuum and cache.

### G.3 Decision Guide

Choose a **separate database** when you need any of:

- Different **encoding, locale, or collation**
- A genuinely different set of **extensions** per application
- A **hard** isolation boundary (a role that cannot even `CONNECT`)
- Per-application **connection limits** (`ALTER DATABASE ... CONNECTION LIMIT`) or per-database settings
- Applications that must never accidentally join each other's data

Choose **schemas in one database** when you need:

- **Joins or foreign keys** between the groups of objects
- A **single transaction** touching several groups
- Lighter operational overhead
- Simple, privilege-based separation

### G.4 Common Misconceptions

| Misconception | Reality |
|---|---|
| "A schema is a lightweight database" | It shares the database's connection, catalogs, encoding, and extensions; it is only a namespace |
| "`CREATE SCHEMA` creates a new database like in MySQL" | It creates a namespace inside the current database only |
| "`\c` and `SET search_path` are similar" | `\c` ends your connection and opens a new one to another database; `SET search_path` changes name lookup within the current one |
| "Granting `SELECT` on a table is enough" | The schema also needs `USAGE` |
| "Schemas are separate folders on disk" | They are not; a table's file location has no schema component (Lab 6) |
| "Dropping a schema is like dropping a database" | `DROP SCHEMA ... CASCADE` removes the objects but leaves the database, its connections, and its other schemas untouched |

---

## Part H: PostgreSQL vs. MySQL

### H.1 The Single Most Important Fact

In MySQL, **`SCHEMA` and `DATABASE` are synonyms.** `CREATE SCHEMA` and `CREATE DATABASE` do exactly the same thing, and `SHOW SCHEMAS` is the same as `SHOW DATABASES`. MySQL has only **two** levels: server → database (schema) → objects.

PostgreSQL has **three**: cluster → database → schema → objects. The word "schema" is the same; the concept is not.

```
MySQL                                   PostgreSQL
-----                                   ----------
Server (mysqld)                         Cluster (postmaster + PGDATA)
 ├── database  ≡  schema                 ├── database
 │    └── tables                         │    ├── schema
 │                                       │    │    └── tables
 └── database  ≡  schema                 │    └── schema
      └── tables                         └── database ...
```

### H.2 Concept Mapping

| Concept | MySQL | PostgreSQL |
|---|---|---|
| `CREATE SCHEMA x` | Creates a **database** named `x` | Creates a namespace inside the **current** database |
| Container of tables | database (= schema) | schema, inside a database |
| Switch context | `USE x;` | `\c x` (different **database**, new connection) or `SET search_path = x;` (different **schema**) |
| Default namespace | The "current database" from `USE` or the connection string | `search_path` (a **list**) |
| Unqualified name, nothing selected | `ERROR 1046: No database selected` | Falls back through `search_path`; error only if nothing creatable is on it |
| Name form | `db.table` | `schema.table` (and `db.schema.table` only for the current db) |
| Cross-database join | Works natively: `FROM a.t JOIN b.t` | **Impossible** in plain SQL; needs `dblink`/`postgres_fdw` |
| Cross-schema join | n/a (same thing as cross-database) | Works natively |
| Foreign key across namespaces | InnoDB allows referencing a table in another database on the same server | Allowed across **schemas**; impossible across **databases** |
| Move a table | `RENAME TABLE db1.t TO db2.t;` | `ALTER TABLE s1.t SET SCHEMA s2;` (within one database only) |
| Users / roles | `'user'@'host'`, server-wide | Roles, **cluster-wide**, no host component |
| Privilege granularity | `GRANT ... ON db.*` / `db.table` | Database (`CONNECT`) + schema (`USAGE`) + object grants |
| Character set / collation | Per database, table, or column; `ALTER DATABASE` can change the default | Encoding fixed **per database** at creation; only collation can vary per column |
| System namespaces | `mysql`, `information_schema`, `performance_schema`, `sys` | `pg_catalog`, `information_schema`, `pg_toast` |
| On disk | A database is a **directory** in the data dir | A database is a directory under `base/`; **schemas have no directory** |
| Identifier case | Database/table name case sensitivity depends on the OS and `lower_case_table_names` | Unquoted identifiers are folded to lowercase on every OS |
| Per-"container" limits/settings | Rarely used | `ALTER DATABASE ... SET` and `CONNECTION LIMIT` at database level; none at schema level |

### H.3 The Same Task, Side by Side

**MySQL** — two "schemas" (really, two databases) joined directly:

```sql
CREATE DATABASE sales;
CREATE DATABASE hr;

CREATE TABLE sales.orders    (order_id INT PRIMARY KEY, emp_id INT, amount DECIMAL(10,2));
CREATE TABLE hr.employees    (emp_id INT PRIMARY KEY, name VARCHAR(50));

SELECT e.name, SUM(o.amount)
FROM sales.orders o
JOIN hr.employees e ON e.emp_id = o.emp_id
GROUP BY e.name;
```

**PostgreSQL** — one database, two schemas, the same join:

```sql
CREATE DATABASE company;
\c company

CREATE SCHEMA sales;
CREATE SCHEMA hr;

CREATE TABLE sales.orders  (order_id int PRIMARY KEY, emp_id int, amount numeric(10,2));
CREATE TABLE hr.employees  (emp_id int PRIMARY KEY, name varchar(50));

SELECT e.name, SUM(o.amount)
FROM sales.orders o
JOIN hr.employees e ON e.emp_id = o.emp_id
GROUP BY e.name;
```

The queries are character-for-character the same. What differs is that PostgreSQL needed **one** `CREATE DATABASE` plus two `CREATE SCHEMA` statements, and if you had created `sales` and `hr` as separate PostgreSQL *databases*, that join would fail.

### H.4 The Migration Rule of Thumb

> **A MySQL "database" almost always maps to a PostgreSQL *schema*, not a PostgreSQL *database*** — especially if the application joins across MySQL databases or enforces cross-database foreign keys.

A sound mapping for a MySQL server hosting one application:

| MySQL | PostgreSQL |
|---|---|
| The MySQL server | One PostgreSQL cluster |
| The group of MySQL databases belonging to one application | **One PostgreSQL database** |
| Each MySQL database within that group | **One schema** inside it |

Map to separate PostgreSQL databases only where the MySQL databases are genuinely independent applications that never query each other.

### H.5 Command Cheat Sheet for MySQL DBAs

| Task | MySQL | PostgreSQL |
|---|---|---|
| List databases | `SHOW DATABASES;` | `\l` or `SELECT datname FROM pg_database;` |
| List schemas | `SHOW SCHEMAS;` (same as above) | `\dn` or `SELECT nspname FROM pg_namespace;` |
| Switch database | `USE sales;` | `\c sales` |
| List tables | `SHOW TABLES;` / `SHOW TABLES FROM sales;` | `\dt` / `\dt sales.*` |
| Describe a table | `DESCRIBE orders;` | `\d sales.orders` |
| Current database | `SELECT DATABASE();` | `SELECT current_database();` |
| Current schema | *(same as database)* | `SELECT current_schema();` |
| Grant on a whole container | `GRANT SELECT ON sales.* TO 'u'@'h';` | `GRANT CONNECT ON DATABASE d TO u;` + `GRANT USAGE ON SCHEMA sales TO u;` + `GRANT SELECT ON ALL TABLES IN SCHEMA sales TO u;` |
| Dump one container | `mysqldump sales > sales.sql` | `pg_dump -n sales dbname > sales.sql` (one schema) or `pg_dump sales` (a whole **database**) |
| Drop a container | `DROP DATABASE sales;` | `DROP SCHEMA sales CASCADE;` or `DROP DATABASE sales;` — **different things** |
| Size of a container | `information_schema.TABLES` aggregate | `pg_database_size('d')` for a database; catalog sum for a schema (Part D.5) |

### H.6 Gotchas for MySQL DBAs Starting on PostgreSQL

1. **"I ran `CREATE SCHEMA` expecting a new database."** It only adds a namespace to the current one.
2. **"Where is my `USE` command?"** There isn't one for schemas; use qualified names or set `search_path`. `\c` switches *databases*, and it reconnects.
3. **"My cross-database join broke."** It was never possible; restructure into schemas in one database, or use a foreign data wrapper.
4. **"`GRANT ALL ON sales.*` is one line in MySQL."** PostgreSQL needs database, schema, and object grants separately, plus `ALTER DEFAULT PRIVILEGES` for future tables.
5. **"Case-sensitivity surprises."** MySQL on Linux treats `Orders` and `orders` as different tables by default; PostgreSQL lowercases unquoted names everywhere, and quoted mixed-case names then must be quoted forever.
6. **"`SHOW CREATE DATABASE` set my charset; how do I change it?"** In PostgreSQL the encoding is fixed for the database's lifetime; plan a new database and a dump/restore.

---

## Part I: Hands-On Lab

**Environment:** Rocky Linux 9, PostgreSQL 17 (PGDG), as configured in Module 2. Run `psql` as the `postgres` OS user. If `pg_dump`/`pg_restore` aren't on your `PATH`, use `/usr/pgsql-17/bin/pg_dump` etc.

**Estimated time:** 60 minutes.

### Lab 0: Prerequisites — Create Databases, Schemas, and Data

```bash
psql -U postgres
```

```sql
-- Two databases: one for the main lab, one to demonstrate the cross-database limit
CREATE DATABASE schema_lab;
CREATE DATABASE schema_lab_b;

\c schema_lab_b
CREATE TABLE public.remote_items (item_id int PRIMARY KEY, item_name text);
INSERT INTO public.remote_items VALUES (1, 'widget'), (2, 'gadget');

\c schema_lab

-- Three schemas in the main database
CREATE SCHEMA hr;
CREATE SCHEMA sales;
CREATE SCHEMA staging;

CREATE TABLE hr.employees (
    emp_id serial PRIMARY KEY,
    name   text NOT NULL,
    dept   text
);

-- sales.orders has a foreign key into a DIFFERENT schema -- allowed within one database
CREATE TABLE sales.orders (
    order_id   serial PRIMARY KEY,
    emp_id     int REFERENCES hr.employees(emp_id),
    amount     numeric(10,2),
    order_date date
);

-- Same table name as sales.orders, different schema
CREATE TABLE staging.orders (
    order_id  int,
    emp_id    int,
    amount    numeric(10,2),
    order_date date,
    loaded_at timestamptz DEFAULT now()
);

INSERT INTO hr.employees (name, dept)
VALUES ('Anitha', 'Sales'), ('Ravi', 'Engineering'), ('Meera', 'Sales');

INSERT INTO sales.orders (emp_id, amount, order_date)
SELECT (i % 3) + 1, (random() * 1000)::numeric(10,2), DATE '2026-01-01' + (i % 200)
FROM generate_series(1, 1000) AS i;

INSERT INTO staging.orders (order_id, emp_id, amount, order_date)
SELECT i, 1, 10.00 * i, DATE '2026-06-01'
FROM generate_series(1, 5) AS i;
```

Verify:

```sql
SELECT (SELECT count(*) FROM hr.employees)  AS employees,
       (SELECT count(*) FROM sales.orders)   AS sales_orders,
       (SELECT count(*) FROM staging.orders) AS staging_orders;
```

**Expected:** `3 | 1000 | 5`

---

### Lab 1: Explore the Hierarchy

```sql
\l                -- databases: schema_lab and schema_lab_b should be listed
\dn               -- schemas in schema_lab: hr, public, sales, staging
\dnS              -- now with system schemas: pg_catalog, information_schema, pg_toast
\dt *.*           -- every table in every schema (long output: includes catalogs)
\dt sales.*       -- just one schema
```

```sql
SELECT current_database(), current_schema(), current_user;
```

**Observe:** three levels — one cluster, several databases, several schemas. `current_schema()` returns `public`, the first existing schema in the default `search_path`.

---

### Lab 2: Same Name, Two Schemas

```sql
SELECT count(*) FROM sales.orders;      -- 1000
SELECT count(*) FROM staging.orders;    -- 5

-- Unqualified: which one do you get?
SHOW search_path;
SELECT count(*) FROM orders;
```

**Expected:** the last query fails with `relation "orders" does not exist`, because neither `sales` nor `staging` is on the default `search_path`. Now fix that deliberately:

```sql
SET search_path = sales, public;
SELECT count(*) FROM orders;            -- 1000: resolved to sales.orders

SET search_path = staging, sales;
SELECT count(*) FROM orders;            -- 5: staging comes first, so it shadows sales

RESET search_path;
```

**Observe:** the same unqualified SQL returned different data purely because of `search_path` order. This is exactly why Part E.5's security advice matters.

---

### Lab 3: Cross-Schema vs. Cross-Database

**3.1 Cross-schema works with a plain join:**

```sql
SELECT e.name, count(*) AS orders, sum(o.amount) AS total
FROM sales.orders o
JOIN hr.employees e ON e.emp_id = o.emp_id
GROUP BY e.name
ORDER BY e.name;
```

**3.2 A foreign key across schemas is enforced:**

```sql
INSERT INTO sales.orders (emp_id, amount, order_date) VALUES (999, 10, now());
-- ERROR:  insert or update on table "orders" violates foreign key constraint ...
```

**3.3 Naming the current database is allowed:**

```sql
SELECT count(*) FROM schema_lab.sales.orders;     -- works: same database
```

**3.4 Naming a different database is not:**

```sql
SELECT * FROM schema_lab_b.public.remote_items;
-- ERROR:  cross-database references are not implemented: schema_lab_b.public.remote_items
```

**3.5 The workaround, `dblink`:**

```sql
CREATE EXTENSION IF NOT EXISTS dblink;

SELECT *
FROM dblink('dbname=schema_lab_b', 'SELECT item_id, item_name FROM public.remote_items')
     AS r(item_id int, item_name text);
```

**Observe:** `dblink` requires the extension (the `postgresql17-contrib` package from Module 2), explicit result column types, and a separate connection under the hood — none of which a cross-schema join needs.

---

### Lab 4: search_path Levels and Function Safety

**4.1 Set it at the role level:**

```sql
CREATE ROLE report_user LOGIN PASSWORD 'ChangeMe_123';
ALTER ROLE report_user SET search_path = sales, public;

SELECT rolname, rolconfig FROM pg_roles WHERE rolname = 'report_user';
```

**4.2 Show a late-bound function being fooled by the caller's `search_path`:**

```sql
CREATE FUNCTION sales.order_count() RETURNS bigint
LANGUAGE sql AS $$ SELECT count(*) FROM orders $$;

SET search_path = sales, public;
SELECT sales.order_count();        -- 1000

SET search_path = staging, sales;
SELECT sales.order_count();        -- 5   <-- same function, different answer: it picked up staging.orders
```

**4.3 Pin the function's own `search_path`:**

```sql
ALTER FUNCTION sales.order_count() SET search_path = sales, pg_temp;

SELECT sales.order_count();        -- 1000 again, regardless of the caller's search_path
RESET search_path;
```

**Observe:** a function without its own `search_path` setting resolves names using whoever calls it. Pinning it (with `pg_temp` last) removes that dependence.

---

### Lab 5: Schema Privileges — The Two Different Errors

```sql
CREATE ROLE analyst LOGIN PASSWORD 'ChangeMe_123';
GRANT CONNECT ON DATABASE schema_lab TO analyst;
```

Impersonate the role from your superuser session (no password or `pg_hba.conf` change required):

```sql
SET ROLE analyst;

SELECT * FROM sales.orders LIMIT 1;
-- ERROR:  permission denied for schema sales        <-- no USAGE on the schema

RESET ROLE;
GRANT USAGE ON SCHEMA sales TO analyst;
SET ROLE analyst;

SELECT * FROM sales.orders LIMIT 1;
-- ERROR:  permission denied for table orders        <-- has USAGE, no SELECT on the table

RESET ROLE;
GRANT SELECT ON ALL TABLES IN SCHEMA sales TO analyst;
SET ROLE analyst;

SELECT * FROM sales.orders LIMIT 1;                  -- works
SELECT * FROM hr.employees LIMIT 1;
-- ERROR:  permission denied for schema hr           <-- other schemas remain closed

RESET ROLE;
```

**Observe:** two distinct failure modes, one grant each. A role needs *both* the schema `USAGE` and the object privilege.

---

### Lab 6: Schemas Have No Directory on Disk

```sql
SELECT oid AS database_oid FROM pg_database WHERE datname = 'schema_lab';

SELECT pg_relation_filepath('sales.orders')   AS sales_orders,
       pg_relation_filepath('staging.orders') AS staging_orders,
       pg_relation_filepath('hr.employees')   AS hr_employees;
```

**Observe:** every path has the form `base/<database_oid>/<relfilenode>` — the same directory for all three tables, even though they belong to three different schemas. There is no `sales/` or `hr/` folder.

```bash
ls /var/lib/pgsql/17/data/base/<database_oid>/ | head
```

**Trainer note:** contrast this with MySQL, where each database is a directory under the data directory. In PostgreSQL the *database* is the directory-level unit; schemas exist only in the catalogs (`pg_namespace`).

---

### Lab 7: Move and Drop

```sql
-- Moving staging.orders into sales collides with the existing table name
ALTER TABLE staging.orders SET SCHEMA sales;
-- ERROR:  relation "orders" already exists in schema "sales"

-- Rename first, then move
ALTER TABLE staging.orders RENAME TO orders_staged;
ALTER TABLE staging.orders_staged SET SCHEMA sales;
\dt sales.*

-- Dropping a non-empty schema is refused
CREATE TABLE staging.scratch (id int);
DROP SCHEMA staging;
-- ERROR:  cannot drop schema staging because other objects depend on it

DROP SCHEMA staging CASCADE;
\dn
```

**Observe:** `SET SCHEMA` is instant (a catalog change, no data copied), `DROP SCHEMA` is protected by default, and `CASCADE` removed only the schema and its contents; `schema_lab`, `hr`, and `sales` are untouched.

---

### Lab 8: Schema-Level Backup and the Cross-Schema Dependency Trap

```bash
# Dump only the sales schema
pg_dump -U postgres -d schema_lab -n sales -F c -f /tmp/sales_only.dump

# Restore it into a fresh, empty database
psql -U postgres -c "CREATE DATABASE schema_lab_restore;"
pg_restore -U postgres -d schema_lab_restore /tmp/sales_only.dump
```

**Observe:** `pg_restore` reports an error for the foreign key, because `sales.orders` references `hr.employees` and the `hr` schema was not part of the dump. The data and table still restore; only that constraint fails. Confirm:

```bash
psql -U postgres -d schema_lab_restore -c "SELECT count(*) FROM sales.orders;"
```

**Lesson:** a schema-scoped `pg_dump -n` does not pull in other schemas it depends on. When schemas are linked by foreign keys or shared functions, dump them together (`-n sales -n hr`).

---

### Lab 9: Measure a Schema

```sql
\c schema_lab

SELECT n.nspname AS schema,
       pg_size_pretty(sum(pg_total_relation_size(c.oid))) AS total_size
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind IN ('r', 'm')
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
  AND n.nspname NOT LIKE 'pg\_toast%'
GROUP BY n.nspname
ORDER BY sum(pg_total_relation_size(c.oid)) DESC;

SELECT pg_size_pretty(pg_database_size('schema_lab')) AS whole_database;
```

---

### Cleanup

```sql
\c postgres
DROP DATABASE schema_lab;
DROP DATABASE schema_lab_b;
DROP DATABASE schema_lab_restore;
DROP ROLE report_user;
DROP ROLE analyst;
```

```bash
rm -f /tmp/sales_only.dump
```

`DROP ROLE` succeeds here because dropping the databases removed every object and grant the roles held (the per-database `GRANT CONNECT` is cleared when the database goes). If it complains, run `REASSIGN OWNED` / `DROP OWNED` first (Module 7, Part J).

### Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `relation "orders" does not exist` right after creating `sales.orders` | The schema isn't on `search_path` | Qualify the name or `SET search_path` |
| `CREATE EXTENSION dblink` fails: `could not open extension control file` | The contrib package isn't installed | `dnf install -y postgresql17-contrib` (Module 2) |
| `SET ROLE analyst` fails: `permission denied to set role` | You're not connected as a superuser or a member of that role | Reconnect as `postgres` |
| `DROP DATABASE` fails: `is being accessed by other users` | A session is still connected to it | `\c postgres` first, and close other sessions |
| `pg_restore` ends with `errors ignored on restore: 1` | The cross-schema FK from Lab 8 | Expected for this lab; dump dependent schemas together |

---

## Discussion Questions / Exercises for Trainees

1. In your own words, explain why `SET search_path` and `\c` look similar but do completely different things.
2. A developer says, "We'll create one database per microservice so they're isolated." What would you ask them before agreeing, and which of Part G.3's criteria would justify it?
3. Two applications need to join `customers` and `invoices` data in a single query. Would you place them in separate databases or separate schemas, and why?
4. A role has `SELECT` on `sales.orders` but gets `permission denied for schema sales`. Diagnose and fix it.
5. A colleague from a MySQL background migrates five MySQL databases to five PostgreSQL databases and then discovers their reports no longer run. Explain what happened and how to restructure it.
6. Why might a SaaS product with 20,000 small tenants prefer a `tenant_id` column with Row-Level Security over schema-per-tenant?
7. Write the `ALTER FUNCTION` statement that makes a `SECURITY DEFINER` function safe against `search_path` hijacking, and explain why `pg_temp` is listed last.

## Key Takeaways

- A PostgreSQL **schema is a namespace inside a database**. The hierarchy is **cluster → database → schema → objects**, one level deeper than MySQL's.
- **A connection belongs to one database**; schemas inside it are freely joinable. Cross-database queries need `dblink` or `postgres_fdw`, and foreign keys cannot cross databases.
- **Encoding, locale, extensions, catalogs, and connection limits are per database**; schemas share all of them. If you need different values, you need a different database.
- In **MySQL, `SCHEMA` is simply a synonym for `DATABASE`**. A MySQL database usually maps to a PostgreSQL **schema**, not a PostgreSQL database, especially when the application joins across them.
- Unqualified names are resolved through **`search_path`**; it can be set per session, role, or database. It is also a security surface, so pin it in `SECURITY DEFINER` functions and keep `pg_temp` last.
- Using an object needs **both** schema `USAGE` and the object privilege, and the two failures produce different error messages.
- **Schemas have no on-disk directory**; a schema-scoped `pg_dump -n` does not automatically include other schemas it depends on.
- Choose **schemas** when objects must join, share transactions, and stay operationally light; choose **separate databases** for hard isolation, different encodings or extensions, or per-application limits.
