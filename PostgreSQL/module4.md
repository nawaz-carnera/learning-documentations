# Module 4: Tables — Create, Modify, and Manage

---

## Table of Contents

- [1. Creating a Table (CREATE TABLE)](#1-creating-a-table-create-table)
- [2. Common Data Types](#2-common-data-types)
  - [Integer Types](#integer-types)
  - [Text Types](#text-types)
  - [Boolean](#boolean)
  - [Date and Time Types](#date-and-time-types)
  - [Numeric / Decimal](#numeric--decimal)
  - [JSONB](#jsonb)
  - [UUID](#uuid)
- [3. Choosing the Right Data Type](#3-choosing-the-right-data-type)
- [4. Altering a Table](#4-altering-a-table)
  - [ADD COLUMN](#add-column)
  - [DROP COLUMN](#drop-column)
  - [RENAME COLUMN](#rename-column)
  - [Change Column Type](#change-column-type-alter-column--type)
  - [Set / Drop Default](#set--drop-a-default-value)
  - [Set / Drop NOT NULL](#set--drop-not-null)
- [5. Renaming a Table](#5-renaming-a-table)
- [6. Dropping a Table](#6-dropping-a-table)
- [7. Temporary Tables](#7-temporary-tables)
- [Quick Reference](#quick-reference)

---

## 1. Creating a Table (CREATE TABLE)

### Basic Syntax
```sql
CREATE TABLE table_name (
  column_name  data_type  [constraints],
  ...
);
```

### Full Example
```sql
CREATE TABLE employees (
  id          SERIAL        PRIMARY KEY,
  first_name  VARCHAR(100)  NOT NULL,
  last_name   VARCHAR(100)  NOT NULL,
  email       TEXT          UNIQUE NOT NULL,
  salary      NUMERIC(10,2) DEFAULT 0.00,
  is_active   BOOLEAN       DEFAULT true,
  hired_at    DATE          NOT NULL,
  created_at  TIMESTAMP     DEFAULT NOW()
);
```

### With Schema Qualification
```sql
CREATE TABLE hr.employees (
  id    SERIAL PRIMARY KEY,
  name  TEXT NOT NULL
);
```

### Safe Creation (no error if already exists)
```sql
CREATE TABLE IF NOT EXISTS employees (
  id   SERIAL PRIMARY KEY,
  name TEXT NOT NULL
);
```

### Create a Table from an Existing Table

**Copy structure + data:**
```sql
CREATE TABLE employees_backup AS
  SELECT * FROM employees;
```

**Copy structure only (no data):**
```sql
CREATE TABLE employees_backup AS
  SELECT * FROM employees WHERE false;
```

> **Edge Case:** `CREATE TABLE ... AS SELECT` does **not** copy constraints (PRIMARY KEY, UNIQUE, NOT NULL, indexes). You get the columns and data only — add constraints manually afterwards if needed.

### Inline Constraints vs Table Constraints

```sql
-- Inline constraint (attached to one column)
CREATE TABLE orders (
  id         SERIAL PRIMARY KEY,
  user_id    INTEGER NOT NULL,
  product_id INTEGER NOT NULL
);

-- Table-level constraint (needed for composite keys/FKs)
CREATE TABLE order_items (
  order_id   INTEGER,
  product_id INTEGER,
  quantity   INTEGER NOT NULL,
  PRIMARY KEY (order_id, product_id)   -- composite primary key
);
```

---

## 2. Common Data Types

### Integer Types

| Type | Storage | Range | Use When |
|---|---|---|---|
| `SMALLINT` | 2 bytes | -32,768 to 32,767 | Small counters, status codes |
| `INTEGER` / `INT` | 4 bytes | -2.1B to 2.1B | General-purpose IDs, counts |
| `BIGINT` | 8 bytes | -9.2 quintillion to 9.2 quintillion | Large IDs, row counts, timestamps as epoch |
| `SERIAL` | 4 bytes | 1 to 2.1B | Auto-increment integer PK (shorthand) |
| `BIGSERIAL` | 8 bytes | 1 to 9.2 quintillion | Auto-increment bigint PK |

```sql
-- SERIAL is shorthand for:
id SERIAL PRIMARY KEY
-- which expands to:
id INTEGER NOT NULL DEFAULT nextval('tablename_id_seq') PRIMARY KEY

-- Modern alternative (PostgreSQL 10+):
id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY
id BIGINT  GENERATED ALWAYS AS IDENTITY PRIMARY KEY
```

> **Edge Case:** `SERIAL` creates a sequence but doesn't truly prevent gaps. If a transaction is rolled back, that sequence value is consumed and skipped. Never rely on SERIAL values being gapless.

---

### Text Types

| Type | Description | Use When |
|---|---|---|
| `CHAR(n)` | Fixed-length, padded with spaces | Rare — fixed codes like country codes `'US'` |
| `VARCHAR(n)` | Variable-length, max n characters | When you want to enforce a length limit |
| `TEXT` | Unlimited length | General-purpose strings — preferred in Postgres |

```sql
country_code  CHAR(2),          -- always exactly 2 chars: 'US', 'IN'
username      VARCHAR(50),      -- max 50 characters
description   TEXT              -- no limit
```

> **Edge Case:** In PostgreSQL, `TEXT` and `VARCHAR` have **identical performance** — there is no speed difference. The only reason to use `VARCHAR(n)` is to enforce an application-level length constraint. Unlike MySQL, Postgres doesn't store them differently.

> **Edge Case:** `CHAR(n)` pads with spaces, which can cause subtle bugs:
> ```sql
> SELECT 'US' = 'US   ';  -- true in CHAR comparison (trailing spaces ignored)
> SELECT 'US'::TEXT = 'US   '::TEXT;  -- false
> ```

---

### Boolean

| Value | Accepted Input |
|---|---|
| `TRUE` | `true`, `'t'`, `'yes'`, `'on'`, `'1'`, `'y'` |
| `FALSE` | `false`, `'f'`, `'no'`, `'off'`, `'0'`, `'n'` |
| `NULL` | unknown / not set |

```sql
is_active   BOOLEAN DEFAULT true,
is_verified BOOLEAN DEFAULT false,
is_deleted  BOOLEAN DEFAULT false
```

```sql
-- Querying booleans
SELECT * FROM users WHERE is_active = true;
SELECT * FROM users WHERE is_active;           -- shorthand
SELECT * FROM users WHERE NOT is_active;       -- false
SELECT * FROM users WHERE is_active IS NULL;   -- unknown
```

> **Edge Case:** A `BOOLEAN` column can be `NULL` (meaning "unknown") — it's not just true/false. Always set a `DEFAULT` or `NOT NULL` if you don't want nulls.

---

### Date and Time Types

| Type | Storage | Description | Example |
|---|---|---|---|
| `DATE` | 4 bytes | Date only (no time) | `'2024-12-25'` |
| `TIME` | 8 bytes | Time only (no date) | `'14:30:00'` |
| `TIMESTAMP` | 8 bytes | Date + time, no timezone | `'2024-12-25 14:30:00'` |
| `TIMESTAMPTZ` | 8 bytes | Date + time **with** timezone | `'2024-12-25 14:30:00+05:30'` |
| `INTERVAL` | 16 bytes | Duration / span of time | `'3 days'`, `'2 hours 30 mins'` |

```sql
hired_at      DATE          NOT NULL,
meeting_at    TIME,
created_at    TIMESTAMP     DEFAULT NOW(),
updated_at    TIMESTAMPTZ   DEFAULT NOW(),
trial_period  INTERVAL      DEFAULT '30 days'
```

```sql
-- Common date/time functions
SELECT NOW();                          -- current timestamp with timezone
SELECT CURRENT_DATE;                   -- today's date
SELECT CURRENT_TIME;                   -- current time
SELECT NOW() + INTERVAL '7 days';     -- 7 days from now
SELECT AGE('2000-01-01'::DATE);       -- time elapsed since a date
SELECT EXTRACT(YEAR FROM NOW());       -- extract year
```

> **Edge Case — TIMESTAMP vs TIMESTAMPTZ:**
> - `TIMESTAMP` stores the value as-is with no timezone info — dangerous if your app runs across timezones
> - `TIMESTAMPTZ` converts to UTC on store and converts back to the session timezone on read — always prefer this for production
> ```sql
> SET timezone = 'America/New_York';
> SELECT '2024-06-01 12:00:00'::TIMESTAMPTZ;
> -- stores as UTC, shows in New York time when queried
> ```

---

### Numeric / Decimal

| Type | Description | Use When |
|---|---|---|
| `NUMERIC(p, s)` | Exact decimal, precision `p`, scale `s` | Money, financial data — no rounding errors |
| `DECIMAL(p, s)` | Alias for `NUMERIC` | Same as above |
| `REAL` | 4-byte floating point (~6 decimal digits) | Scientific data where approximation is ok |
| `DOUBLE PRECISION` | 8-byte floating point (~15 decimal digits) | Same as above, more precision |

```sql
price      NUMERIC(10, 2),   -- up to 99,999,999.99 — good for money
tax_rate   NUMERIC(5, 4),    -- e.g. 0.1875 (18.75%)
score      REAL,             -- approximate, fine for analytics
ratio      DOUBLE PRECISION  -- approximate, more decimal digits
```

> **Edge Case — Never use FLOAT/REAL for money:**
> ```sql
> SELECT 0.1::REAL + 0.2::REAL;
> -- result: 0.30000001192092896  ← floating point rounding error!
>
> SELECT 0.1::NUMERIC + 0.2::NUMERIC;
> -- result: 0.3  ← exact
> ```
> Always use `NUMERIC` for currency, tax, rates, or any value where precision matters.

---

### JSONB

PostgreSQL has two JSON types:

| Type | Storage | Indexable | Speed |
|---|---|---|---|
| `JSON` | Stored as plain text | No | Slower (re-parses on every read) |
| `JSONB` | Stored as binary | Yes (GIN index) | Faster — use this |

```sql
CREATE TABLE products (
  id       SERIAL PRIMARY KEY,
  name     TEXT NOT NULL,
  metadata JSONB
);

-- Insert JSON data
INSERT INTO products (name, metadata)
VALUES ('Laptop', '{"brand": "Dell", "specs": {"ram": 16, "ssd": 512}, "tags": ["sale", "new"]}');

-- Query inside JSONB
SELECT metadata -> 'brand' AS brand              FROM products;  -- returns JSON: "Dell"
SELECT metadata ->> 'brand' AS brand             FROM products;  -- returns TEXT: Dell
SELECT metadata -> 'specs' ->> 'ram' AS ram      FROM products;  -- nested: 16
SELECT metadata #>> '{specs,ram}' AS ram         FROM products;  -- path syntax: 16

-- Filter by JSONB value
SELECT * FROM products WHERE metadata ->> 'brand' = 'Dell';
SELECT * FROM products WHERE metadata @> '{"tags": ["sale"]}';  -- contains

-- Update a JSONB key
UPDATE products SET metadata = metadata || '{"discount": true}' WHERE id = 1;

-- Index JSONB for fast queries
CREATE INDEX idx_products_metadata ON products USING GIN (metadata);
```

| Operator | Returns | Meaning |
|---|---|---|
| `->` | JSON | Get key as JSON value |
| `->>` | TEXT | Get key as text value |
| `#>` | JSON | Get value at path |
| `#>>` | TEXT | Get value at path as text |
| `@>` | BOOLEAN | Does left contain right? |
| `?` | BOOLEAN | Does key exist? |

> **Edge Case:** `JSONB` does not preserve key order or duplicate keys — it stores a normalized binary form. If order matters, use `JSON`. Also, `JSONB` silently drops duplicate keys (keeps the last one).

---

### UUID

A **UUID** (Universally Unique Identifier) is a 128-bit value, typically shown as:
`550e8400-e29b-41d4-a716-446655440000`

```sql
-- Enable the extension (needed for uuid_generate_v4() in older Postgres)
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Using uuid-ossp extension
CREATE TABLE sessions (
  id         UUID    PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id    INTEGER NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- PostgreSQL 13+ built-in function (no extension needed)
CREATE TABLE sessions (
  id         UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id    INTEGER NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Insert (auto-generated)
INSERT INTO sessions (user_id) VALUES (42);

-- Insert (manual)
INSERT INTO sessions (id, user_id) VALUES ('550e8400-e29b-41d4-a716-446655440000', 42);
```

**UUID vs SERIAL for primary keys:**

| | UUID | SERIAL / BIGSERIAL |
|---|---|---|
| Globally unique | Yes — safe across DBs, servers | No — only unique within one table |
| Merge/replicate data | Easy | Conflicts likely |
| Guessable | No — good for public APIs | Yes — sequential |
| Storage | 16 bytes | 4–8 bytes |
| Index performance | Slightly slower (random inserts) | Faster (sequential) |
| Readability | Hard to read | Easy |

> **Edge Case:** Random UUIDs (v4) cause **index fragmentation** over time because inserts go to random B-tree positions instead of the end. For high-insert-rate tables, use `UUIDv7` (time-ordered) or `ULID` to keep inserts sequential while still being globally unique.

---

## 3. Choosing the Right Data Type

| Scenario | Recommended Type | Avoid |
|---|---|---|
| Auto-increment PK (small table) | `SERIAL` or `INT GENERATED ALWAYS AS IDENTITY` | `BIGINT` (overkill) |
| Auto-increment PK (large table) | `BIGSERIAL` or `BIGINT GENERATED ALWAYS AS IDENTITY` | `SERIAL` (can overflow) |
| Globally unique ID | `UUID DEFAULT gen_random_uuid()` | `SERIAL` (not globally unique) |
| Short fixed codes (`'US'`, `'M'`) | `CHAR(2)` | `TEXT` (works but less expressive) |
| Variable-length strings | `TEXT` | `CHAR` (padding issues) |
| Enforce max string length | `VARCHAR(n)` | `TEXT` with no constraint |
| Money / financial values | `NUMERIC(12, 2)` | `FLOAT`, `REAL` (precision loss) |
| Approximate scientific values | `DOUBLE PRECISION` | `NUMERIC` (overkill) |
| True/false flag | `BOOLEAN` | `SMALLINT` (0/1) |
| Date only (no time) | `DATE` | `TIMESTAMP` (stores unnecessary time) |
| Timestamps in production app | `TIMESTAMPTZ` | `TIMESTAMP` (timezone bugs) |
| Flexible/dynamic attributes | `JSONB` | Separate EAV tables (complex queries) |
| Status with fixed set of values | `TEXT` + `CHECK` constraint or `ENUM` | Magic integers |

**Golden rules:**
1. Use `TIMESTAMPTZ` not `TIMESTAMP` for any datetime stored in production
2. Use `NUMERIC` not `FLOAT` for money
3. Use `TEXT` over `VARCHAR` unless you need the length constraint
4. Use `BIGINT` for PKs on tables expected to grow large
5. Use `JSONB` over `JSON` — always

---

## 4. Altering a Table

`ALTER TABLE` modifies an existing table's structure.

### ADD COLUMN

```sql
ALTER TABLE employees ADD COLUMN phone VARCHAR(20);

-- With default value
ALTER TABLE employees ADD COLUMN is_remote BOOLEAN DEFAULT false;

-- With NOT NULL (must provide a default or the column will fail on existing rows)
ALTER TABLE employees ADD COLUMN department TEXT NOT NULL DEFAULT 'General';
```

> **Edge Case:** Adding a `NOT NULL` column without a `DEFAULT` to a table that already has rows will fail:
> ```sql
> ALTER TABLE employees ADD COLUMN department TEXT NOT NULL;
> -- ERROR: column "department" contains null values
>
> -- Fix: add with a default, then drop the default if desired
> ALTER TABLE employees ADD COLUMN department TEXT NOT NULL DEFAULT 'General';
> ALTER TABLE employees ALTER COLUMN department DROP DEFAULT;
> ```

> **PostgreSQL 11+:** Adding a column with a `NOT NULL DEFAULT` is instant (no table rewrite). Earlier versions required a full table rewrite — very slow on large tables.

---

### DROP COLUMN

```sql
ALTER TABLE employees DROP COLUMN phone;

-- Safe version
ALTER TABLE employees DROP COLUMN IF EXISTS phone;

-- Drop and remove all objects that depend on the column (foreign keys, indexes, views)
ALTER TABLE employees DROP COLUMN phone CASCADE;
```

> **Edge Case:** If a view or function references the column you're dropping, `DROP COLUMN` without `CASCADE` will fail. Use `CASCADE` to remove dependent objects, or drop/update them manually first.

---

### RENAME COLUMN

```sql
ALTER TABLE employees RENAME COLUMN hired_at TO hire_date;
```

> **Edge Case:** Renaming a column does **not** automatically update views, functions, or application code that references the old name. Those will break silently or at runtime — always search for usages before renaming.

---

### Change Column Type (ALTER COLUMN … TYPE)

```sql
-- Change type
ALTER TABLE employees ALTER COLUMN salary TYPE BIGINT;

-- Change type with explicit cast (when automatic cast isn't possible)
ALTER TABLE employees ALTER COLUMN phone TYPE INTEGER USING phone::INTEGER;

-- Change VARCHAR length
ALTER TABLE employees ALTER COLUMN first_name TYPE VARCHAR(200);
```

> **Edge Case:** Some type changes require a full table rewrite (e.g., `TEXT` → `INTEGER`), which locks the table and is slow on large tables. On production, use a background migration strategy:
> 1. Add a new column with the new type
> 2. Backfill data
> 3. Switch the application to use the new column
> 4. Drop the old column

> **Edge Case:** Increasing `VARCHAR(n)` size (e.g., 50 → 200) is instant. Decreasing it requires a table rewrite and will fail if existing data exceeds the new limit.

---

### Set / Drop a Default Value

```sql
-- Set a default
ALTER TABLE employees ALTER COLUMN is_active SET DEFAULT true;

-- Drop the default
ALTER TABLE employees ALTER COLUMN is_active DROP DEFAULT;
```

> Changing a default only affects **future inserts** — existing rows are not changed.

---

### Set / Drop NOT NULL

```sql
-- Add NOT NULL constraint
ALTER TABLE employees ALTER COLUMN email SET NOT NULL;

-- Remove NOT NULL constraint
ALTER TABLE employees ALTER COLUMN email DROP NOT NULL;
```

> **Edge Case:** Adding `NOT NULL` to a column that has existing `NULL` values will fail:
> ```sql
> -- Fix: update nulls first
> UPDATE employees SET email = 'unknown@company.com' WHERE email IS NULL;
> -- Then add constraint
> ALTER TABLE employees ALTER COLUMN email SET NOT NULL;
> ```

---

## 5. Renaming a Table

```sql
ALTER TABLE old_name RENAME TO new_name;
```

### Examples

```sql
ALTER TABLE employees RENAME TO staff;

-- With schema qualification
ALTER TABLE hr.employees RENAME TO hr.staff;
```

> **Edge Case:** Renaming a table updates the table name in `pg_catalog` but does **not** update:
> - Views that reference the old name
> - Foreign key constraints referencing the old name (these break)
> - Sequences created by `SERIAL` (they keep the old naming convention)
> - Application queries using the old name
>
> Always check dependencies before renaming:
> ```sql
> SELECT dependent_view.relname AS view_name
> FROM pg_depend
> JOIN pg_rewrite ON pg_depend.objid = pg_rewrite.oid
> JOIN pg_class AS dependent_view ON pg_rewrite.ev_class = dependent_view.oid
> JOIN pg_class AS source_table ON pg_depend.refobjid = source_table.oid
> WHERE source_table.relname = 'employees';
> ```

---

## 6. Dropping a Table

### Basic Drop
```sql
DROP TABLE table_name;
```

### Safe Drop (no error if not exists)
```sql
DROP TABLE IF EXISTS table_name;
```

### Drop Multiple Tables
```sql
DROP TABLE IF EXISTS orders, order_items, products;
```

### Drop with CASCADE (removes dependent objects)
```sql
DROP TABLE employees CASCADE;
```

### Examples

```sql
DROP TABLE IF EXISTS employees;

-- Drop and remove all views, foreign keys that depend on this table
DROP TABLE employees CASCADE;
```

> **Edge Case:** If another table has a `FOREIGN KEY` referencing the table you're dropping, a plain `DROP TABLE` will fail:
> ```sql
> DROP TABLE employees;
> -- ERROR: cannot drop table employees because other objects depend on it
> -- DETAIL: constraint orders_employee_id_fkey on table orders depends on table employees
>
> -- Fix option 1: drop the FK first
> ALTER TABLE orders DROP CONSTRAINT orders_employee_id_fkey;
> DROP TABLE employees;
>
> -- Fix option 2: cascade drop everything
> DROP TABLE employees CASCADE;
> ```

> **Edge Case:** `DROP TABLE` is permanent. There is no undo without a backup. For large production tables, consider:
> ```sql
> -- Safer: rename first, verify nothing breaks, then drop later
> ALTER TABLE employees RENAME TO employees_deprecated_20240101;
> -- ... monitor for a few days ...
> DROP TABLE employees_deprecated_20240101;
> ```

---

## 7. Temporary Tables

A **temporary table** exists only for the duration of a session (or transaction). It's automatically dropped when the session ends.

### Create a Temporary Table

```sql
-- Lives for the entire session
CREATE TEMP TABLE temp_results (
  id     INTEGER,
  score  NUMERIC
);

-- Alternative syntax
CREATE TEMPORARY TABLE temp_results (
  id     INTEGER,
  score  NUMERIC
);
```

### Transaction-Scoped Temporary Table (dropped on COMMIT)

```sql
CREATE TEMP TABLE temp_results (
  id    INTEGER,
  score NUMERIC
) ON COMMIT DROP;
```

| `ON COMMIT` Option | Behavior |
|---|---|
| `PRESERVE ROWS` (default) | Data kept after commit, table dropped at session end |
| `DELETE ROWS` | Data deleted after each commit, table structure stays |
| `DROP` | Entire table dropped after commit |

### Practical Uses

```sql
-- Store intermediate results of a complex query
CREATE TEMP TABLE high_value_customers AS
  SELECT customer_id, SUM(amount) AS total
  FROM orders
  GROUP BY customer_id
  HAVING SUM(amount) > 10000;

-- Now use it in further queries without recomputing
SELECT c.name, h.total
FROM customers c
JOIN high_value_customers h ON c.id = h.customer_id;
```

```sql
-- Use for staging/import: load data, validate, then insert
CREATE TEMP TABLE import_staging (
  raw_data TEXT
);

COPY import_staging FROM '/tmp/data.csv';

-- Validate and insert clean rows
INSERT INTO products (name, price)
SELECT
  raw_data::JSON->>'name',
  (raw_data::JSON->>'price')::NUMERIC
FROM import_staging
WHERE (raw_data::JSON->>'price')::NUMERIC > 0;
```

### Key Behaviors and Edge Cases

> **Temp tables are session-private** — two sessions can create temp tables with the same name without conflict. Each session sees only its own.

> **Temp tables live in `pg_temp_*` schema** — they shadow permanent tables with the same name in the session's `search_path`:
> ```sql
> CREATE TABLE products (id SERIAL, name TEXT);       -- permanent
> CREATE TEMP TABLE products (id INT, price NUMERIC); -- temp
>
> SELECT * FROM products;  -- returns temp table, not the permanent one!
> -- Access permanent explicitly:
> SELECT * FROM public.products;
> ```

> **Temp tables are not visible to other sessions** — you cannot share temp table data between connections. Use a regular table or `UNLOGGED TABLE` for shared staging data.

> **UNLOGGED tables** — a middle ground between temp and permanent: faster than regular tables (no WAL writes), survive session ends, but are truncated on crash/restart:
> ```sql
> CREATE UNLOGGED TABLE staging_data (
>   id   SERIAL,
>   data JSONB
> );
> ```

---

## Quick Reference

```sql
-- Create
CREATE TABLE employees (id SERIAL PRIMARY KEY, name TEXT NOT NULL);
CREATE TABLE IF NOT EXISTS employees (...);
CREATE TABLE backup AS SELECT * FROM employees;         -- with data
CREATE TABLE backup AS SELECT * FROM employees WHERE false;  -- structure only

-- Common types
id          SERIAL / BIGSERIAL / INT GENERATED ALWAYS AS IDENTITY
uuid_col    UUID DEFAULT gen_random_uuid()
name        TEXT / VARCHAR(100)
amount      NUMERIC(10, 2)
flag        BOOLEAN DEFAULT false
created_at  TIMESTAMPTZ DEFAULT NOW()
metadata    JSONB

-- Alter: columns
ALTER TABLE t ADD COLUMN col TEXT DEFAULT 'x';
ALTER TABLE t DROP COLUMN IF EXISTS col;
ALTER TABLE t DROP COLUMN col CASCADE;
ALTER TABLE t RENAME COLUMN old TO new;
ALTER TABLE t ALTER COLUMN col TYPE BIGINT;
ALTER TABLE t ALTER COLUMN col TYPE BIGINT USING col::BIGINT;
ALTER TABLE t ALTER COLUMN col SET DEFAULT 'value';
ALTER TABLE t ALTER COLUMN col DROP DEFAULT;
ALTER TABLE t ALTER COLUMN col SET NOT NULL;
ALTER TABLE t ALTER COLUMN col DROP NOT NULL;

-- Rename table
ALTER TABLE old_name RENAME TO new_name;

-- Drop
DROP TABLE IF EXISTS employees;
DROP TABLE employees CASCADE;

-- Temp tables
CREATE TEMP TABLE t (id INT, val TEXT);
CREATE TEMP TABLE t (...) ON COMMIT DROP;
CREATE UNLOGGED TABLE staging (...);
```
