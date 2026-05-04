# Module 2: Databases — Create, Manage, and Organize

---

## Table of Contents

- [1. What is a Database? (Cluster vs Database vs Schema)](#1-what-is-a-database-cluster-vs-database-vs-schema)
  - [Cluster](#cluster)
  - [Database](#database)
  - [Schema](#schema)
- [2. Creating a Database](#2-creating-a-database)
- [3. Listing Databases (\l)](#3-listing-databases-l)
- [4. Switching Databases (\c)](#4-switching-databases-c)
- [5. Dropping a Database](#5-dropping-a-database)
- [6. Database Ownership and Basic Permissions](#6-database-ownership-and-basic-permissions)
  - [Ownership](#ownership)
  - [Roles vs Users](#roles-vs-users)
  - [Key Permissions (Privileges)](#key-permissions-privileges)
  - [Common Permission Setup Pattern](#common-permission-setup-pattern-new-user--new-database)
  - [Checking Existing Permissions](#checking-existing-permissions)
  - [Superuser vs Regular User](#superuser-vs-regular-user)
- [Quick Reference](#quick-reference)

---

## 1. What is a Database? (Cluster vs Database vs Schema)

PostgreSQL has a 3-level hierarchy to organize data. Understanding this prevents a lot of confusion early on.

```
Cluster
└── Database (mydb)
    └── Schema (public / hr / finance)
        └── Tables, Views, Functions, Sequences...
```

### Cluster
A **cluster** is the entire PostgreSQL installation — one running Postgres server (one process, one data directory).
- A single cluster can host **multiple databases**
- All databases in a cluster share the same PostgreSQL users/roles
- Managed by one `postgres` process on one port (default: 5432)

```bash
# One cluster = one running postgres process
pg_ctl status -D /var/lib/postgresql/data
```

> Think of a cluster as the **entire server**.

---

### Database
A **database** is an isolated container inside the cluster.
- Tables in `db1` are completely separate from tables in `db2`
- You connect to **one database at a time** — you cannot join tables across databases directly
- Each database has its own schemas, tables, and objects

```sql
-- You can't do this (cross-database join) in PostgreSQL:
SELECT * FROM db1.public.users JOIN db2.public.orders ON ...;  -- ERROR

-- You'd need an extension (dblink or postgres_fdw) for cross-database queries
```

> Think of a database as a **separate filing cabinet**.

---

### Schema
A **schema** is a namespace inside a database — a logical grouping of tables and other objects.
- Every database has a default schema called **`public`**
- You can create multiple schemas to organize objects (e.g., `hr`, `finance`, `audit`)
- Same table name can exist in different schemas without conflict

```sql
-- Two tables with the same name in different schemas — no conflict
CREATE TABLE hr.employees (...);
CREATE TABLE finance.employees (...);

-- Access with schema prefix
SELECT * FROM hr.employees;
SELECT * FROM finance.employees;
```

**The `search_path`** controls which schema Postgres looks in when no schema is specified:
```sql
SHOW search_path;              -- default: "$user", public
SET search_path TO hr, public; -- now "employees" resolves to hr.employees first
```

### Summary Table

| Level | Analogy | Isolated From |
|---|---|---|
| Cluster | The entire server / building | Other clusters (other ports) |
| Database | A filing cabinet | Other databases in same cluster |
| Schema | A drawer in the cabinet | Other schemas in same database |
| Table | A folder in the drawer | — |

---

## 2. Creating a Database

### Basic Syntax
```sql
CREATE DATABASE database_name;
```

### Full Syntax with Options
```sql
CREATE DATABASE database_name
  OWNER = username
  ENCODING = 'UTF8'
  LC_COLLATE = 'en_US.UTF-8'
  LC_CTYPE = 'en_US.UTF-8'
  TEMPLATE = template1
  CONNECTION LIMIT = -1;
```

| Option | Description | Default |
|---|---|---|
| `OWNER` | Who owns the database | Current user |
| `ENCODING` | Character encoding | Server default (usually UTF8) |
| `LC_COLLATE` | Sort order for strings | Server locale |
| `LC_CTYPE` | Character classification | Server locale |
| `TEMPLATE` | Template database to copy from | `template1` |
| `CONNECTION LIMIT` | Max concurrent connections (-1 = unlimited) | -1 |

### Examples

**Simple:**
```sql
CREATE DATABASE shop;
```

**With owner and encoding:**
```sql
CREATE DATABASE shop
  OWNER = shopuser
  ENCODING = 'UTF8';
```

**With connection limit (useful for staging/dev DBs):**
```sql
CREATE DATABASE staging
  OWNER = devuser
  CONNECTION LIMIT = 10;
```

### Using `createdb` from the terminal (shortcut)
```bash
createdb -U postgres -O shopuser shop
#          ^username   ^owner      ^dbname
```

### Edge Cases

> **Error: `database "shop" already exists`**
> Use `CREATE DATABASE IF NOT EXISTS` — but wait, PostgreSQL does **not** support this syntax (unlike MySQL).
> Workaround from shell:
> ```bash
> psql -U postgres -tc "SELECT 1 FROM pg_database WHERE datname = 'shop'" | grep -q 1 || createdb shop
> ```
> Or catch in PL/pgSQL:
> ```sql
> DO $$
> BEGIN
>   IF NOT EXISTS (SELECT FROM pg_database WHERE datname = 'shop') THEN
>     PERFORM dblink_exec('dbname=postgres', 'CREATE DATABASE shop');
>   END IF;
> END
> $$;
> ```

> **Error: `new encoding (UTF8) is incompatible with the encoding of the template database (SQL_ASCII)`**
> Use `TEMPLATE = template0` (a clean template that allows encoding changes):
> ```sql
> CREATE DATABASE shop
>   ENCODING = 'UTF8'
>   LC_COLLATE = 'en_US.UTF-8'
>   LC_CTYPE = 'en_US.UTF-8'
>   TEMPLATE = template0;
> ```

---

## 3. Listing Databases (`\l`)

### In psql
```sql
\l
```
```
                               List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | Access privileges
-----------+----------+----------+-------------+-------------+-------------------
 mydb      | admin    | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 shop      | shopuser | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
```

**Extended listing** (shows size, description):
```sql
\l+
```

### Via SQL Query (useful in scripts)
```sql
SELECT datname, pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```
```
  datname  |  size
-----------+--------
 mydb      | 8537 kB
 shop      | 7820 kB
 postgres  | 7820 kB
 template1 | 7820 kB
 template0 | 7820 kB
```

### System Databases (Never Delete These)

| Database | Purpose |
|---|---|
| `postgres` | Default admin database, safe to use for admin tasks |
| `template1` | Template for new databases — objects added here appear in new DBs |
| `template0` | Clean pristine template — use when changing encoding/collation |

---

## 4. Switching Databases (`\c`)

You can only be connected to **one database at a time** in psql.

### Basic Switch
```sql
\c shop
```
```
You are now connected to database "shop" as user "admin".
shop=#
```

### Switch with Different User
```sql
\c shop shopuser
```
```
You are now connected to database "shop" as user "shopuser".
```

### Switch with Host and Port (full form)
```sql
\c "host=localhost port=5432 dbname=shop user=admin"
```

### Check Current Database
```sql
SELECT current_database();
```
```
 current_database
------------------
 shop
```

> **Edge Case:** If you `\c` to a database that doesn't exist, psql throws an error but you **stay connected** to the current database — you are not disconnected:
> ```
> shop=# \c nonexistent
> FATAL: database "nonexistent" does not exist
> Previous connection kept
> shop=#    ← still here
> ```

---

## 5. Dropping a Database

### Basic Syntax
```sql
DROP DATABASE database_name;
```

### Safe Version (no error if it doesn't exist)
```sql
DROP DATABASE IF EXISTS shop;
```

### Example
```sql
DROP DATABASE IF EXISTS staging;
```

### Rules and Edge Cases

> **Error: `cannot drop the currently open database`**
> You must switch to another database before dropping:
> ```sql
> \c postgres   -- switch away first
> DROP DATABASE shop;
> ```

> **Error: `database "shop" is being accessed by other users`**
> Other clients are still connected. You must terminate their connections first:
> ```sql
> -- Terminate all connections to "shop"
> SELECT pg_terminate_backend(pid)
> FROM pg_stat_activity
> WHERE datname = 'shop' AND pid <> pg_backend_pid();
>
> -- Now drop safely
> DROP DATABASE shop;
> ```

> **PostgreSQL 13+ shortcut** — force-disconnect all users and drop in one command:
> ```sql
> DROP DATABASE shop WITH (FORCE);
> ```

> **Warning:** `DROP DATABASE` is **permanent and unrecoverable** without a backup. There is no `RECYCLE BIN`. Always double-check before dropping.

Using `dropdb` from the terminal:
```bash
dropdb -U postgres shop
```

---

## 6. Database Ownership and Basic Permissions

### Ownership

Every database has one **owner** — the role that created it (or was assigned ownership).
The owner can:
- Alter and drop the database
- Connect to it
- Grant/revoke permissions on it

```sql
-- Check database owners
SELECT datname, pg_catalog.pg_get_userbyid(datdba) AS owner
FROM pg_catalog.pg_database;
```

**Change database owner:**
```sql
ALTER DATABASE shop OWNER TO newowner;
```

---

### Roles vs Users

In PostgreSQL, **users and roles are the same thing** — a `USER` is just a role with the `LOGIN` privilege.

```sql
-- These two are equivalent:
CREATE USER shopuser WITH PASSWORD 'pass123';
CREATE ROLE shopuser WITH LOGIN PASSWORD 'pass123';
```

---

### Key Permissions (Privileges)

#### 1. Connect to a Database
```sql
-- Grant connect permission
GRANT CONNECT ON DATABASE shop TO shopuser;

-- Revoke connect permission
REVOKE CONNECT ON DATABASE shop FROM shopuser;
```

#### 2. Create Schemas inside a Database
```sql
GRANT CREATE ON DATABASE shop TO shopuser;
```

#### 3. Use a Schema (see objects inside it)
```sql
GRANT USAGE ON SCHEMA public TO shopuser;
```

#### 4. Access Tables inside a Schema
```sql
-- Grant on specific table
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE products TO shopuser;

-- Grant on all current tables in schema
GRANT SELECT ON ALL TABLES IN SCHEMA public TO shopuser;

-- Grant on all future tables too (default privileges)
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO shopuser;
```

---

### Common Permission Setup Pattern (New User + New Database)

```sql
-- Step 1: Create a user
CREATE USER shopuser WITH PASSWORD 'securepass';

-- Step 2: Create the database, owned by that user
CREATE DATABASE shop OWNER shopuser;

-- Step 3: Connect to the database
\c shop

-- Step 4: Grant schema usage
GRANT USAGE ON SCHEMA public TO shopuser;

-- Step 5: Grant table access
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO shopuser;

-- Step 6: Ensure future tables are also accessible
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO shopuser;
```

---

### Checking Existing Permissions

```sql
-- Database-level privileges
\l shop

-- Schema-level privileges
\dn+

-- Table-level privileges
\dp tablename

-- Or via SQL
SELECT grantee, privilege_type
FROM information_schema.role_table_grants
WHERE table_name = 'products';
```

---

### Superuser vs Regular User

| Capability | Superuser | Regular User |
|---|---|---|
| Create databases | Yes | Only if `CREATEDB` granted |
| Create roles | Yes | Only if `CREATEROLE` granted |
| Bypass all permissions | Yes | No |
| Connect to any database | Yes | Only if `CONNECT` granted |

```sql
-- Grant CREATEDB privilege to a user (without making them superuser)
ALTER USER devuser CREATEDB;

-- Create a superuser
CREATE USER adminuser WITH SUPERUSER PASSWORD 'strongpass';
```

> **Edge Case:** Even a superuser cannot connect to a database if `pg_hba.conf` (host-based auth file) blocks them at the OS/network level — `pg_hba.conf` is evaluated before any SQL-level permissions.

---

## Quick Reference

```sql
-- Create
CREATE DATABASE mydb OWNER myuser ENCODING 'UTF8';

-- List (psql)
\l
\l+          -- with size and description

-- List (SQL)
SELECT datname FROM pg_database;

-- Switch
\c mydb
\c mydb myuser

-- Drop
DROP DATABASE IF EXISTS mydb;
DROP DATABASE mydb WITH (FORCE);   -- PostgreSQL 13+

-- Ownership
ALTER DATABASE mydb OWNER TO newowner;

-- Permissions
GRANT CONNECT ON DATABASE mydb TO myuser;
REVOKE CONNECT ON DATABASE mydb FROM myuser;
```
