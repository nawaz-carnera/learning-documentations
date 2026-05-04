# Module 1: PostgreSQL Fundamentals

---

## Table of Contents

- [1. What is PostgreSQL?](#1-what-is-postgresql)
- [2. Installing PostgreSQL](#2-installing-postgresql)
  - [Option A: Direct Installation](#option-a-direct-installation)
  - [Option B: Using Docker](#option-b-using-docker-recommended-for-learning)
- [3. Connecting via psql (Command Line)](#3-connecting-via-psql-command-line)
- [4. Connecting via pgAdmin (GUI)](#4-connecting-via-pgadmin-gui)
- [5. Basic psql Commands (Meta-Commands)](#5-basic-psql-commands-meta-commands)
  - [\l — List All Databases](#l--list-all-databases)
  - [\c — Connect to a Database](#c--connect-to-a-database)
  - [\dt — List All Tables](#dt--list-all-tables-in-current-database)
  - [\d — Describe a Table](#d--describe-a-table-or-object)
  - [\q — Quit psql](#q--quit-psql)
- [Quick Reference Card](#quick-reference-card)
- [Common Edge Cases Summary](#common-edge-cases-summary)

---

## 1. What is PostgreSQL?

PostgreSQL (a.k.a. **Postgres**) is a free, open-source **relational database management system (RDBMS)**. It stores data in tables with rows and columns, supports SQL, and is known for being rock-solid, feature-rich, and standards-compliant.

### PostgreSQL vs Other Databases

| Feature | PostgreSQL | MySQL | SQLite | MongoDB |
|---|---|---|---|---|
| Type | Relational (RDBMS) | Relational (RDBMS) | Relational (embedded) | NoSQL (document) |
| License | Open-source (free) | Open-source (free) | Public domain | SSPL (mixed) |
| ACID Compliant | Yes (full) | Partial (depends on engine) | Yes | Yes (since 4.0) |
| JSON Support | Yes (native, queryable) | Limited | No | Native |
| Transactions | Full | Full | Limited | Yes |
| Stored Procedures | Yes | Yes | No | Yes |
| Best For | Complex queries, analytics | Web apps, read-heavy | Local/small apps | Flexible schemas |

**When to choose PostgreSQL:**
- You need complex queries, joins, and aggregations
- Data integrity is critical (banking, e-commerce)
- You want both relational + JSON (document) support in one DB
- You need advanced features: full-text search, window functions, CTEs, custom types

**Key strengths:**
- MVCC (Multi-Version Concurrency Control) — readers don't block writers
- Extensible — add custom types, functions, operators
- Strong standards compliance (SQL:2016)

---

## 2. Installing PostgreSQL

### Option A: Direct Installation

**macOS (Homebrew):**
```bash
brew install postgresql@16
brew services start postgresql@16
```

**Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

**Windows:**
Download installer from: https://www.postgresql.org/download/windows/
(Includes pgAdmin GUI automatically)

---

### Option B: Using Docker (Recommended for Learning)

Docker lets you run PostgreSQL without installing it on your machine — clean, portable, and easy to reset.

**Step 1 — Pull and run a PostgreSQL container:**
```bash
docker run --name my-postgres \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_USER=admin \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  -d postgres:16
```

| Flag | Meaning |
|---|---|
| `--name my-postgres` | Container name |
| `-e POSTGRES_PASSWORD` | Sets the superuser password |
| `-e POSTGRES_USER` | Sets the superuser username (default: postgres) |
| `-e POSTGRES_DB` | Creates a default database |
| `-p 5432:5432` | Maps host port 5432 to container port 5432 |
| `-d` | Run in detached (background) mode |

**Step 2 — Verify container is running:**
```bash
docker ps
```

**Step 3 — Connect into the container's shell:**
```bash
docker exec -it my-postgres bash
```

**Stop / Start / Remove container:**
```bash
docker stop my-postgres
docker start my-postgres
docker rm -f my-postgres   # force remove (deletes data too!)
```

> **Edge Case:** If port 5432 is already in use on your machine (by a local Postgres install), use `-p 5433:5432` and connect on port 5433 instead.

---

## 3. Connecting via `psql` (Command Line)

`psql` is the official PostgreSQL interactive terminal.

### Connection Syntax
```bash
psql -h <host> -p <port> -U <username> -d <database>
```

**Examples:**

Connect to local PostgreSQL:
```bash
psql -U postgres
```

Connect to Docker container from outside:
```bash
psql -h localhost -p 5432 -U admin -d mydb
```

Connect inside the Docker container:
```bash
docker exec -it my-postgres psql -U admin -d mydb
```

Connect with password prompt:
```bash
psql -h localhost -U admin -d mydb -W
```

### Connection String (URL format)
```bash
psql "postgresql://admin:secret@localhost:5432/mydb"
```

### What You See After Connecting
```
psql (16.0)
Type "help" for help.

mydb=#
```
- `mydb=#` — you are connected to `mydb` as a **superuser**
- `mydb=>` — you are connected as a **regular user**

> **Edge Case:** If you get `peer authentication failed`, it means PostgreSQL expects your OS username to match the DB username. Use `-h localhost` to force TCP connection instead of Unix socket — this uses password auth.

---

## 4. Connecting via pgAdmin (GUI)

pgAdmin is the official web-based GUI for PostgreSQL.

### Installation
- Download from: https://www.pgadmin.org/download/
- Or run via Docker:
```bash
docker run --name pgadmin \
  -e PGADMIN_DEFAULT_EMAIL=admin@admin.com \
  -e PGADMIN_DEFAULT_PASSWORD=admin \
  -p 8080:80 \
  -d dpage/pgadmin4
```
Then open: `http://localhost:8080`

### Adding a Server Connection in pgAdmin

1. Open pgAdmin → right-click **Servers** → **Register → Server**
2. **General tab:** Give it a name (e.g., `MyLocalDB`)
3. **Connection tab:**
   - Host: `localhost` (or `host.docker.internal` if pgAdmin runs in Docker)
   - Port: `5432`
   - Database: `mydb`
   - Username: `admin`
   - Password: `secret`
4. Click **Save**

> **Edge Case — pgAdmin in Docker connecting to Postgres in Docker:**
> Use `host.docker.internal` (Mac/Windows) or the container's IP instead of `localhost`. Alternatively, put both containers in the same Docker network:
> ```bash
> docker network create pgnet
> docker run --name my-postgres --network pgnet ...
> docker run --name pgadmin --network pgnet ...
> # Then use "my-postgres" as the host in pgAdmin
> ```

### What You Can Do in pgAdmin
- Browse databases, schemas, tables visually
- Run SQL queries in the **Query Tool** (Tools → Query Tool)
- View table data, explain query plans, manage users

---

## 5. Basic `psql` Commands (Meta-Commands)

These are `psql`-specific commands (not SQL). They start with `\` and do **not** need a semicolon.

### `\l` — List All Databases
```sql
mydb=# \l
```
```
                              List of databases
   Name    |  Owner  | Encoding | ...
-----------+---------+----------+----
 mydb      | admin   | UTF8     |
 postgres  | admin   | UTF8     |
 template0 | admin   | UTF8     |
 template1 | admin   | UTF8     |
```
> `template0` and `template1` are system databases — never drop them.

---

### `\c` — Connect to a Database
```sql
mydb=# \c otherdb
```
```
You are now connected to database "otherdb" as user "admin".
otherdb=#
```

Connect as a different user:
```sql
\c otherdb someuser
```

> **Edge Case:** If the target database doesn't exist, you get an error and stay connected to the current database — you won't get disconnected.

---

### `\dt` — List All Tables in Current Database
```sql
mydb=# \dt
```
```
        List of relations
 Schema |   Name   | Type  | Owner
--------+----------+-------+-------
 public | orders   | table | admin
 public | products | table | admin
 public | users    | table | admin
```

List tables in a specific schema:
```sql
\dt schema_name.*
```

List tables matching a pattern:
```sql
\dt user*     -- lists all tables starting with "user"
```

> **Edge Case:** If you see `Did not find any relations`, the database exists but has no tables yet — or you're in the wrong schema. Try `\dt *.*` to list across all schemas.

---

### `\d` — Describe a Table (or Object)
```sql
mydb=# \d users
```
```
                     Table "public.users"
   Column   |          Type          | Nullable |      Default
------------+------------------------+----------+--------------------
 id         | integer                | not null | nextval('users_id_seq')
 name       | character varying(100) | not null |
 email      | character varying(255) | not null |
 created_at | timestamp              |          | now()

Indexes:
    "users_pkey" PRIMARY KEY, btree (id)
    "users_email_key" UNIQUE CONSTRAINT, btree (email)
```

Other useful `\d` variants:

| Command | Description |
|---|---|
| `\d tablename` | Describe a table (columns, indexes, constraints) |
| `\d+ tablename` | Extended description (includes storage, comments) |
| `\di` | List all indexes |
| `\ds` | List all sequences |
| `\dv` | List all views |
| `\df` | List all functions |
| `\dn` | List all schemas |
| `\du` | List all users/roles |

---

### `\q` — Quit psql
```sql
mydb=# \q
```
Returns you to the terminal shell.

> You can also press `Ctrl + D` to exit.

---

## Quick Reference Card

```
\l          → list databases
\c dbname   → switch to database
\dt         → list tables
\d table    → describe table structure
\d+ table   → detailed table description
\di         → list indexes
\dn         → list schemas
\du         → list users/roles
\q          → quit psql
\?          → help for psql commands
\h SELECT   → SQL syntax help for SELECT
```

---

## Common Edge Cases Summary

| Situation | What Happens | Fix |
|---|---|---|
| Port 5432 already in use | Docker container fails to start | Use `-p 5433:5432` |
| `peer authentication failed` | Unix socket auth fails | Add `-h localhost` to force TCP |
| `\dt` shows nothing | No tables in current schema | Try `\dt *.*` |
| `\c` to non-existent DB | Error, stays in current DB | Create DB first with `CREATE DATABASE name;` |
| pgAdmin can't reach Docker Postgres | `localhost` doesn't resolve | Use `host.docker.internal` or Docker network |
