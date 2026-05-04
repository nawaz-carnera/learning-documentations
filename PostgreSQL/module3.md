# Module 3: Schemas — Organize Your Database

---

## Table of Contents

- [1. What is a Schema and Why It Exists](#1-what-is-a-schema-and-why-it-exists)
- [2. The Default `public` Schema](#2-the-default-public-schema)
- [3. Creating a Schema](#3-creating-a-schema)
- [4. Listing Schemas (\dn)](#4-listing-schemas-dn)
- [5. Schema search_path](#5-schema-search_path)
- [6. Qualifying Table Names (schema.table)](#6-qualifying-table-names-schematable)
- [7. Dropping a Schema](#7-dropping-a-schema)
- [Quick Reference](#quick-reference)

---

## 1. What is a Schema and Why It Exists

A **schema** is a named namespace inside a database. It holds tables, views, functions, sequences, and other objects — grouped logically under one name.

```
Database: shop
├── Schema: public        → general tables (products, orders)
├── Schema: hr            → employee tables (staff, payroll)
├── Schema: audit         → audit/log tables (change_log, access_log)
└── Schema: reporting     → views and reporting tables
```

### Why schemas exist

**1. Organization** — Group related tables together instead of dumping everything into one flat namespace.

**2. Avoid name collisions** — Two schemas can have tables with the same name without conflict:
```sql
hr.employees       -- HR team's table
finance.employees  -- Finance team's table
-- No conflict. Both exist in the same database.
```

**3. Multi-tenancy** — One database, one schema per tenant (client):
```sql
tenant_acme.orders
tenant_globex.orders
-- Same structure, isolated data per customer.
```

**4. Permission isolation** — Grant a user access to only their schema, not the entire database:
```sql
GRANT USAGE ON SCHEMA hr TO hr_user;       -- hr_user sees only hr schema
GRANT USAGE ON SCHEMA finance TO fin_user; -- fin_user sees only finance schema
```

**5. Logical separation without the cost of separate databases** — Schemas share the same DB connection, transaction, and can JOIN freely across each other.

```sql
-- Cross-schema JOIN works perfectly (unlike cross-database)
SELECT e.name, p.amount
FROM hr.employees e
JOIN finance.payroll p ON e.id = p.employee_id;
```

---

## 2. The Default `public` Schema

Every new PostgreSQL database comes with a schema called **`public`** created automatically.

- When you create a table without specifying a schema, it goes into `public` by default
- All users can use the `public` schema by default (in older Postgres versions)
- It exists purely for convenience — there's nothing magic about it technically

```sql
-- These two are identical when search_path includes public:
CREATE TABLE products (...);
CREATE TABLE public.products (...);
```

### Public schema permissions changed in PostgreSQL 15+

Before Postgres 15, all users had `CREATE` privilege on the `public` schema by default.
From **Postgres 15 onwards**, this was revoked for security reasons.

```sql
-- Postgres 15+: you must explicitly grant CREATE on public schema
GRANT CREATE ON SCHEMA public TO myuser;

-- Or grant to all users
GRANT CREATE ON SCHEMA public TO PUBLIC;  -- PUBLIC = all users
```

> **Edge Case:** If you upgrade from Postgres 14 to 15 and users suddenly can't create tables, this permission change is usually the cause.

---

## 3. Creating a Schema

### Basic Syntax
```sql
CREATE SCHEMA schema_name;
```

### Create with an Owner
```sql
CREATE SCHEMA schema_name AUTHORIZATION owner_name;
```

### Create Only If It Doesn't Exist
```sql
CREATE SCHEMA IF NOT EXISTS schema_name;
```

### Examples

```sql
-- Simple schema
CREATE SCHEMA hr;

-- Schema owned by a specific user
CREATE SCHEMA finance AUTHORIZATION financeuser;

-- Schema with the same name as the user (shortcut — owned by that user automatically)
CREATE SCHEMA AUTHORIZATION hruser;
-- Creates schema named "hruser", owned by "hruser"

-- Safe creation
CREATE SCHEMA IF NOT EXISTS audit;
```

### Create a Schema and Objects Inside It (one block)
```sql
CREATE SCHEMA hr
  CREATE TABLE employees (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL
  )
  CREATE TABLE departments (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL
  );
```

> All objects defined in the block belong to the `hr` schema.

---

## 4. Listing Schemas (`\dn`)

### In psql
```sql
\dn
```
```
      List of schemas
  Name   |     Owner
---------+---------------
 audit   | admin
 finance | financeuser
 hr      | hruser
 public  | pg_database_owner
```

**Extended listing** (includes privileges):
```sql
\dn+
```
```
                          List of schemas
  Name   |       Owner       |           Access privileges
---------+-------------------+----------------------------------------
 audit   | admin             | admin=UC/admin
 finance | financeuser       | financeuser=UC/financeuser
 hr      | hruser            | hruser=UC/hruser
 public  | pg_database_owner | pg_database_owner=UC/pg_database_owner+
         |                   | =U/pg_database_owner
```

Privilege letters: `U` = USAGE, `C` = CREATE

### Via SQL
```sql
-- List all schemas in current database
SELECT schema_name, schema_owner
FROM information_schema.schemata
ORDER BY schema_name;
```

```sql
-- Includes system schemas
SELECT nspname AS schema, pg_get_userbyid(nspowner) AS owner
FROM pg_namespace
ORDER BY nspname;
```

### System Schemas (Don't Touch)

| Schema | Purpose |
|---|---|
| `public` | Default user schema |
| `pg_catalog` | PostgreSQL system tables (pg_class, pg_attribute, etc.) |
| `information_schema` | SQL-standard views about DB objects |
| `pg_toast` | Internal storage for large values (TOAST) |
| `pg_temp_*` | Temporary tables for active sessions |

---

## 5. Schema `search_path`

The `search_path` tells PostgreSQL **which schemas to look in** (and in what order) when a table name is used without a schema prefix.

### Check the current search_path
```sql
SHOW search_path;
```
```
   search_path
-----------------
 "$user", public
```

- `"$user"` — first looks for a schema with the **same name as the current user** (e.g., if you're `hruser`, it checks `hruser` schema first)
- `public` — falls back to the `public` schema

### How it resolves names

```sql
-- Assume search_path = "$user", public
-- Connected as: admin
-- Schemas present: public, hr, finance

SELECT * FROM employees;
-- Step 1: look for admin.employees   → not found
-- Step 2: look for public.employees  → found! uses this one
```

### Set search_path for the current session
```sql
SET search_path TO hr, public;

-- Now unqualified "employees" resolves to hr.employees first
SELECT * FROM employees;  -- uses hr.employees
```

### Set search_path permanently for a user
```sql
ALTER USER hruser SET search_path TO hr, public;
```

### Set search_path permanently for a database
```sql
ALTER DATABASE shop SET search_path TO hr, public;
```

### Set search_path for a specific function
```sql
CREATE FUNCTION get_staff()
RETURNS TABLE(name TEXT) AS $$
  SELECT name FROM employees;
$$ LANGUAGE sql
SET search_path = hr, public;  -- scoped to this function only
```

### Edge Cases

> **Ambiguity:** If `employees` exists in both `hr` and `public`, and `search_path = hr, public`, then `SELECT * FROM employees` uses `hr.employees`. The `public` one is silently ignored — no error, no warning.

> **Schema not in search_path:** If you create a table in `finance` schema but `search_path` doesn't include `finance`, an unqualified query will fail:
> ```sql
> SET search_path TO public;
> SELECT * FROM payroll;  -- ERROR: relation "payroll" does not exist
> SELECT * FROM finance.payroll;  -- OK, explicit schema prefix works
> ```

> **Security risk (`search_path` injection):** A malicious user could create a `public.pg_tables` that shadows the real system view. Best practice: always use explicit schema names in functions and stored procedures, and set `search_path = "$user"` (no public) for security-sensitive roles.

---

## 6. Qualifying Table Names (schema.table)

Using the full `schema.table` notation is called **qualified naming**. It makes your SQL unambiguous regardless of `search_path`.

### Syntax
```
schema_name.table_name
database_name.schema_name.table_name  -- 3-part (rare, mostly for FDW/dblink)
```

### Examples

```sql
-- Unqualified (relies on search_path)
SELECT * FROM employees;

-- Qualified (explicit, always works)
SELECT * FROM hr.employees;
SELECT * FROM finance.payroll;

-- Cross-schema JOIN (qualified names required for clarity)
SELECT e.name, p.salary
FROM hr.employees e
JOIN finance.payroll p ON e.id = p.employee_id;

-- INSERT into a specific schema
INSERT INTO audit.change_log (table_name, changed_at)
VALUES ('employees', NOW());

-- CREATE TABLE in a specific schema
CREATE TABLE finance.budgets (
  id     SERIAL PRIMARY KEY,
  amount NUMERIC(12, 2)
);
```

### When to always use qualified names

| Situation | Recommendation |
|---|---|
| Multiple schemas with same table name | Always qualify |
| Functions / stored procedures | Always qualify (avoids `search_path` attacks) |
| Migration scripts | Always qualify (runs correctly regardless of session settings) |
| Application DB queries | Qualify or set `search_path` on the connection |
| Quick ad-hoc psql queries | Unqualified is fine |

---

## 7. Dropping a Schema

### Drop an Empty Schema
```sql
DROP SCHEMA schema_name;
```

### Drop Only If It Exists
```sql
DROP SCHEMA IF EXISTS schema_name;
```

### Drop a Schema and Everything Inside It (`CASCADE`)
```sql
DROP SCHEMA schema_name CASCADE;
```
> `CASCADE` drops all tables, views, functions, sequences, and other objects inside the schema. **This is irreversible.**

### Drop Multiple Schemas at Once
```sql
DROP SCHEMA IF EXISTS hr, finance, audit CASCADE;
```

### Examples

```sql
-- Drop empty schema (fails if it has any objects)
DROP SCHEMA audit;

-- Safe drop of empty schema
DROP SCHEMA IF EXISTS audit;

-- Drop schema with all its contents
DROP SCHEMA hr CASCADE;

-- Drop multiple schemas and all their contents
DROP SCHEMA IF EXISTS hr, finance CASCADE;
```

### Edge Cases

> **Error: `cannot drop schema public because other objects depend on it`** — Use `CASCADE` or remove objects manually first.

> **Dropping `public` schema:** You can drop it, but don't unless you know what you're doing. Many tools and ORMs assume `public` exists.
> ```sql
> DROP SCHEMA public CASCADE;  -- removes all tables in public too!
> -- Recreate it:
> CREATE SCHEMA public;
> GRANT ALL ON SCHEMA public TO PUBLIC;
> ```

> **`CASCADE` is silent** — it won't list what it deleted. Before running `DROP SCHEMA ... CASCADE`, check what's inside:
> ```sql
> -- Preview what you're about to delete
> SELECT table_schema, table_name
> FROM information_schema.tables
> WHERE table_schema = 'hr';
> ```

> **Schema with active connections:** Dropping a schema does not require disconnecting users. If a user is mid-transaction using a table inside that schema and you drop the schema with `CASCADE`, their transaction will fail with an error when they next touch those objects.

---

## Quick Reference

```sql
-- Create
CREATE SCHEMA hr;
CREATE SCHEMA IF NOT EXISTS hr;
CREATE SCHEMA finance AUTHORIZATION financeuser;

-- List (psql)
\dn        -- basic
\dn+       -- with privileges

-- List (SQL)
SELECT schema_name FROM information_schema.schemata;

-- search_path
SHOW search_path;
SET search_path TO hr, public;
ALTER USER hruser SET search_path TO hr, public;
ALTER DATABASE shop SET search_path TO hr, public;

-- Qualified naming
SELECT * FROM hr.employees;
INSERT INTO finance.payroll VALUES (...);

-- Drop
DROP SCHEMA IF EXISTS hr;
DROP SCHEMA hr CASCADE;           -- removes all objects inside
DROP SCHEMA IF EXISTS hr, finance CASCADE;
```
