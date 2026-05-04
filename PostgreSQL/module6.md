# Module 6: DML — Insert, Update, Delete, and Upsert

---

## Table of Contents

- [1. INSERT INTO ... VALUES](#1-insert-into--values)
  - [Basic Insert](#basic-insert)
  - [Insert with Column List](#insert-with-column-list)
  - [Omitting Columns with Defaults](#omitting-columns-with-defaults)
- [2. INSERT with Multiple Rows](#2-insert-with-multiple-rows)
- [3. INSERT INTO ... SELECT](#3-insert-into--select)
- [4. UPDATE with WHERE](#4-update-with-where)
  - [Basic Update](#basic-update)
  - [Update Multiple Columns](#update-multiple-columns)
  - [Update with Expressions](#update-with-expressions)
  - [UPDATE with JOIN (UPDATE ... FROM)](#update-with-join-update--from)
- [5. DELETE with WHERE](#5-delete-with-where)
  - [Basic Delete](#basic-delete)
  - [DELETE with JOIN (DELETE ... USING)](#delete-with-join-delete--using)
  - [DELETE vs TRUNCATE](#delete-vs-truncate)
- [6. RETURNING Clause](#6-returning-clause)
  - [RETURNING with INSERT](#returning-with-insert)
  - [RETURNING with UPDATE](#returning-with-update)
  - [RETURNING with DELETE](#returning-with-delete)
  - [Using RETURNING with CTEs](#using-returning-with-ctes)
- [7. UPSERT (INSERT ... ON CONFLICT)](#7-upsert-insert--on-conflict)
  - [ON CONFLICT DO NOTHING](#on-conflict-do-nothing)
  - [ON CONFLICT DO UPDATE](#on-conflict-do-update)
  - [Partial Index Conflict Target](#partial-index-conflict-target)
- [Quick Reference](#quick-reference)

---

## 1. INSERT INTO ... VALUES

### Basic Insert

```sql
INSERT INTO table_name (col1, col2, col3)
VALUES (val1, val2, val3);
```

### Setup (tables used throughout this module)

```sql
CREATE TABLE users (
  id         SERIAL PRIMARY KEY,
  name       TEXT    NOT NULL,
  email      TEXT    UNIQUE NOT NULL,
  role       TEXT    NOT NULL DEFAULT 'customer',
  is_active  BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE orders (
  id         SERIAL PRIMARY KEY,
  user_id    INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  status     TEXT    NOT NULL DEFAULT 'pending',
  total      NUMERIC(10,2) NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Insert with Column List

Always specify the column list — it makes inserts explicit and safe against future schema changes:

```sql
INSERT INTO users (name, email, role)
VALUES ('Alice', 'alice@example.com', 'admin');
```

### Without Column List (positional — fragile, avoid)

```sql
-- Values must match ALL columns in exact table order
INSERT INTO users VALUES (DEFAULT, 'Bob', 'bob@example.com', 'customer', true, NOW());
-- Breaks silently if a new column is added to the table
```

### Omitting Columns with Defaults

Columns with `DEFAULT` or that are `NULLABLE` can be omitted:

```sql
-- role defaults to 'customer', is_active to true, created_at to NOW()
INSERT INTO users (name, email)
VALUES ('Charlie', 'charlie@example.com');

-- Explicit DEFAULT keyword also works
INSERT INTO users (name, email, role)
VALUES ('Dana', 'dana@example.com', DEFAULT);
```

> **Edge Case — Serial / Identity columns on explicit insert:**
> ```sql
> -- Trying to insert a specific ID with GENERATED ALWAYS AS IDENTITY:
> INSERT INTO users (id, name, email) VALUES (99, 'Eve', 'eve@example.com');
> -- ERROR: cannot insert into column "id" — it's GENERATED ALWAYS
>
> -- With SERIAL (old style), you CAN insert manually, but it risks future conflicts:
> INSERT INTO users (id, name, email) VALUES (99, 'Eve', 'eve@example.com');
> -- OK with SERIAL, but the sequence doesn't advance — next auto-insert
> -- might collide with 99 when the sequence eventually reaches it.
>
> -- Fix: reset the sequence after manual ID inserts
> SELECT setval('users_id_seq', MAX(id)) FROM users;
> ```

> **Edge Case — Inserting into a specific schema:**
> ```sql
> INSERT INTO hr.employees (name, email) VALUES ('Frank', 'frank@corp.com');
> ```

---

## 2. INSERT with Multiple Rows

Insert multiple rows in a single statement — far more efficient than individual inserts.

### Syntax

```sql
INSERT INTO table_name (col1, col2)
VALUES
  (val1a, val2a),
  (val1b, val2b),
  (val1c, val2c);
```

### Example

```sql
INSERT INTO users (name, email, role)
VALUES
  ('Alice',   'alice@example.com',   'admin'),
  ('Bob',     'bob@example.com',     'customer'),
  ('Charlie', 'charlie@example.com', 'customer'),
  ('Dana',    'dana@example.com',    'staff'),
  ('Eve',     'eve@example.com',     'customer');
```

### With RETURNING to get all inserted IDs

```sql
INSERT INTO users (name, email)
VALUES
  ('Frank', 'frank@example.com'),
  ('Grace', 'grace@example.com')
RETURNING id, name;
```
```
 id |  name
----+-------
  6 | Frank
  7 | Grace
```

> **Edge Case — One row fails, all rows fail:**
> Multi-row inserts run as a single statement inside an implicit transaction. If any row violates a constraint, the entire insert is rolled back — not just the bad row:
> ```sql
> INSERT INTO users (name, email)
> VALUES
>   ('Henry', 'henry@example.com'),   -- unique email, would succeed
>   ('Alice', 'alice@example.com');   -- duplicate email, violates UNIQUE
> -- ERROR: duplicate key value violates unique constraint "users_email_key"
> -- Result: Henry is NOT inserted either
>
> -- To insert valid rows and skip duplicates, use ON CONFLICT DO NOTHING:
> INSERT INTO users (name, email)
> VALUES
>   ('Henry', 'henry@example.com'),
>   ('Alice', 'alice@example.com')
> ON CONFLICT (email) DO NOTHING;
> -- Henry inserted, Alice skipped — no error
> ```

> **Edge Case — Performance:** Multi-row inserts are significantly faster than individual inserts because they reduce round-trips and transaction overhead. For very large datasets (100k+ rows), use `COPY` instead:
> ```sql
> COPY users (name, email, role) FROM '/tmp/users.csv' CSV HEADER;
> ```

---

## 3. INSERT INTO ... SELECT

Copy or transform rows from one table (or query result) into another.

### Syntax

```sql
INSERT INTO target_table (col1, col2)
SELECT expr1, expr2
FROM source_table
WHERE condition;
```

### Copy rows from another table

```sql
-- Archive inactive users into a separate table
CREATE TABLE users_archive (LIKE users INCLUDING ALL);

INSERT INTO users_archive
SELECT * FROM users
WHERE is_active = false;
```

### Transform data on insert

```sql
-- Migrate data with transformation
INSERT INTO orders_v2 (user_id, status, total_cents, created_at)
SELECT
  user_id,
  UPPER(status),             -- normalize status to uppercase
  (total * 100)::INTEGER,    -- convert dollars to cents
  created_at
FROM orders
WHERE created_at < '2024-01-01';
```

### Insert derived/aggregated data

```sql
-- Create a monthly summary table
CREATE TABLE monthly_revenue (
  year    INTEGER,
  month   INTEGER,
  revenue NUMERIC(12,2),
  PRIMARY KEY (year, month)
);

INSERT INTO monthly_revenue (year, month, revenue)
SELECT
  EXTRACT(YEAR  FROM created_at)::INTEGER,
  EXTRACT(MONTH FROM created_at)::INTEGER,
  SUM(total)
FROM orders
WHERE status = 'delivered'
GROUP BY 1, 2
ORDER BY 1, 2;
```

### Cross-schema insert

```sql
INSERT INTO archive.orders
SELECT * FROM public.orders
WHERE created_at < NOW() - INTERVAL '1 year';
```

> **Edge Case — Column count and type must match:**
> The `SELECT` must return the same number of columns as the `INSERT` target list, in the same order, with compatible types. Postgres will attempt implicit casting, but mismatches cause errors:
> ```sql
> INSERT INTO orders (user_id, total)
> SELECT user_id, status   -- status is TEXT, total is NUMERIC → ERROR
> FROM orders;
> ```

> **Edge Case — INSERT INTO ... SELECT does not reset sequences:**
> If you copy rows including their `id` values, the sequence for the target table is not updated. Future auto-incremented inserts may collide:
> ```sql
> INSERT INTO users_backup SELECT * FROM users;
> -- users_backup sequence is still at 1, but rows already have IDs up to 100
>
> -- Fix after bulk copy:
> SELECT setval('users_backup_id_seq', MAX(id)) FROM users_backup;
> ```

---

## 4. UPDATE with WHERE

`UPDATE` modifies existing rows. **Always use `WHERE`** — without it, every row in the table is updated.

### Basic Update

```sql
UPDATE table_name
SET column = value
WHERE condition;
```

```sql
-- Update a single row
UPDATE users
SET role = 'admin'
WHERE id = 5;

-- Update multiple rows matching condition
UPDATE users
SET is_active = false
WHERE created_at < NOW() - INTERVAL '1 year';
```

### Update Multiple Columns

```sql
UPDATE users
SET
  name       = 'Alice Smith',
  email      = 'alice.smith@example.com',
  is_active  = true
WHERE id = 1;
```

### Update with Expressions

```sql
-- Append a suffix to all customer emails
UPDATE users
SET email = email || '.old'
WHERE role = 'customer' AND is_active = false;

-- Increase all order totals by 10%
UPDATE orders
SET total = total * 1.10
WHERE status = 'pending';

-- Conditional update using CASE
UPDATE orders
SET status = CASE
  WHEN total > 1000 THEN 'priority'
  WHEN total > 500  THEN 'standard'
  ELSE 'economy'
END
WHERE status = 'pending';
```

### UPDATE with JOIN (UPDATE ... FROM)

PostgreSQL uses `UPDATE ... FROM` syntax (not `UPDATE ... JOIN`):

```sql
-- Update orders based on data in another table
UPDATE orders o
SET status = 'cancelled'
FROM users u
WHERE o.user_id = u.id
  AND u.is_active = false
  AND o.status = 'pending';
```

```sql
-- Update using a subquery
UPDATE users
SET role = 'vip'
WHERE id IN (
  SELECT user_id
  FROM orders
  GROUP BY user_id
  HAVING SUM(total) > 5000
);
```

```sql
-- Update with a CTE
WITH big_spenders AS (
  SELECT user_id
  FROM orders
  GROUP BY user_id
  HAVING SUM(total) > 5000
)
UPDATE users
SET role = 'vip'
FROM big_spenders
WHERE users.id = big_spenders.user_id;
```

> **Edge Case — UPDATE without WHERE:**
> ```sql
> UPDATE users SET is_active = false;
> -- Updates EVERY row in the table — no warning, no confirmation
> -- Always double-check: run SELECT with the same WHERE first
> SELECT COUNT(*) FROM users WHERE <your condition>;
> -- Then run the UPDATE
> ```

> **Edge Case — UPDATE ... FROM produces a row per join match:**
> If the `FROM` table has multiple matching rows for one target row, the target row is updated multiple times — the final value is unpredictable:
> ```sql
> -- If a user has 3 pending orders, this updates user 1 three times (last write wins)
> UPDATE users u
> SET role = 'active'
> FROM orders o
> WHERE u.id = o.user_id AND o.status = 'pending';
>
> -- Fix: use DISTINCT or a subquery to get one row per user
> UPDATE users
> SET role = 'active'
> WHERE id IN (SELECT DISTINCT user_id FROM orders WHERE status = 'pending');
> ```

> **Edge Case — Updating a column referenced in WHERE:**
> ```sql
> UPDATE users SET id = id + 100 WHERE id > 50;
> -- PostgreSQL evaluates WHERE against the original values, not the updated ones
> -- No infinite loop — safe behavior
> ```

---

## 5. DELETE with WHERE

`DELETE` removes rows. **Always use `WHERE`** — without it, every row is deleted.

### Basic Delete

```sql
DELETE FROM table_name WHERE condition;
```

```sql
-- Delete a single row by ID
DELETE FROM users WHERE id = 10;

-- Delete multiple rows
DELETE FROM orders WHERE status = 'cancelled' AND created_at < NOW() - INTERVAL '6 months';

-- Delete all inactive users
DELETE FROM users WHERE is_active = false;
```

### DELETE with JOIN (DELETE ... USING)

PostgreSQL uses `DELETE ... USING` to join other tables:

```sql
-- Delete orders belonging to inactive users
DELETE FROM orders o
USING users u
WHERE o.user_id = u.id
  AND u.is_active = false;
```

```sql
-- Delete using a subquery
DELETE FROM orders
WHERE user_id IN (
  SELECT id FROM users WHERE is_active = false
);
```

```sql
-- Delete using a CTE
WITH inactive_users AS (
  SELECT id FROM users WHERE is_active = false
)
DELETE FROM orders
USING inactive_users
WHERE orders.user_id = inactive_users.id;
```

> **Edge Case — DELETE without WHERE:**
> ```sql
> DELETE FROM orders;
> -- Deletes every row — logged, transactional, but irreversible without a backup
> -- Use TRUNCATE instead if you intentionally want to clear all rows (faster)
> ```

> **Edge Case — DELETE and FK CASCADE:**
> If a parent row is deleted and child rows have `ON DELETE CASCADE`, those child rows are also deleted silently. Understand your FK chains before bulk deleting:
> ```sql
> DELETE FROM users WHERE id = 1;
> -- Also deletes: all orders for user 1 (cascade)
> -- Also deletes: all order_items for those orders (if cascade set there too)
> ```

> **Edge Case — DELETE is slow on large tables:** `DELETE` is fully MVCC-logged and leaves dead tuples that `VACUUM` must later clean up. For clearing all rows, `TRUNCATE` is orders of magnitude faster and reclaims space immediately.

---

### DELETE vs TRUNCATE

| | DELETE | TRUNCATE |
|---|---|---|
| Removes specific rows | Yes (with WHERE) | No — always removes all rows |
| Transactional (rollback) | Yes | Yes (in PostgreSQL) |
| Fires row triggers | Yes | No (unless `TRUNCATE` trigger defined) |
| Resets SERIAL/IDENTITY | No | Yes (with `RESTART IDENTITY`) |
| Speed on large tables | Slow (MVCC overhead) | Very fast (deallocates pages) |
| WHERE clause | Yes | No |
| FK constraint check | Yes | Yes (unless CASCADE) |

```sql
-- TRUNCATE examples
TRUNCATE orders;                              -- fast clear, keeps sequence
TRUNCATE orders RESTART IDENTITY;            -- also resets id sequence to 1
TRUNCATE orders CASCADE;                     -- truncates orders + all FK-dependent tables
TRUNCATE orders, order_items RESTART IDENTITY CASCADE;
```

> **Edge Case — TRUNCATE and foreign keys:**
> ```sql
> TRUNCATE users;
> -- ERROR: cannot truncate table "users" because other tables reference it
>
> -- Fix: truncate with cascade
> TRUNCATE users CASCADE;  -- truncates users AND orders (and anything else referencing users)
> ```

---

## 6. RETURNING Clause

`RETURNING` is a PostgreSQL-specific clause that returns data from rows affected by `INSERT`, `UPDATE`, or `DELETE` — eliminating the need for a separate `SELECT` query.

### RETURNING with INSERT

```sql
-- Get the auto-generated ID after insert
INSERT INTO users (name, email)
VALUES ('Alice', 'alice@example.com')
RETURNING id;
```
```
 id
----
  1
```

```sql
-- Return multiple columns
INSERT INTO users (name, email, role)
VALUES ('Bob', 'bob@example.com', 'admin')
RETURNING id, name, created_at;
```
```
 id | name |          created_at
----+------+-------------------------------
  2 | Bob  | 2024-06-01 10:23:45.123456+00
```

```sql
-- Return everything
INSERT INTO orders (user_id, total)
VALUES (1, 149.99)
RETURNING *;
```

```sql
-- Multi-row insert — RETURNING gives one row per inserted row
INSERT INTO users (name, email)
VALUES
  ('Charlie', 'charlie@example.com'),
  ('Dana',    'dana@example.com')
RETURNING id, name;
```
```
 id |  name
----+---------
  3 | Charlie
  4 | Dana
```

---

### RETURNING with UPDATE

```sql
-- See what was actually changed
UPDATE users
SET is_active = false
WHERE created_at < NOW() - INTERVAL '1 year'
RETURNING id, name, email;
```
```
 id |  name  |       email
----+--------+--------------------
  7 | Frank  | frank@example.com
 12 | Grace  | grace@example.com
```

```sql
-- Return old and new values (using RETURNING with alias)
UPDATE orders
SET total = total * 1.10
WHERE status = 'pending'
RETURNING id, total AS new_total;
```

---

### RETURNING with DELETE

```sql
-- See exactly which rows were deleted
DELETE FROM users
WHERE is_active = false
RETURNING id, name, email;
```
```
 id |  name  |       email
----+--------+--------------------
  7 | Frank  | frank@example.com
 12 | Grace  | grace@example.com
```

```sql
-- Archive rows while deleting them (in one transaction)
INSERT INTO users_archive
  SELECT * FROM users WHERE is_active = false;

DELETE FROM users
WHERE is_active = false
RETURNING id;
```

---

### Using RETURNING with CTEs

The most powerful pattern — use `RETURNING` inside a CTE to chain DML operations:

```sql
-- Delete old orders and simultaneously insert them into archive
WITH deleted_orders AS (
  DELETE FROM orders
  WHERE created_at < NOW() - INTERVAL '1 year'
  RETURNING *
)
INSERT INTO orders_archive
SELECT * FROM deleted_orders;
```

```sql
-- Insert a user and immediately insert a default order for them
WITH new_user AS (
  INSERT INTO users (name, email)
  VALUES ('Heidi', 'heidi@example.com')
  RETURNING id
)
INSERT INTO orders (user_id, total, status)
SELECT id, 0.00, 'draft'
FROM new_user;
```

```sql
-- Update and log the change in one statement
WITH updated AS (
  UPDATE users
  SET role = 'vip'
  WHERE id = 5
  RETURNING id, role AS new_role
)
INSERT INTO audit_log (table_name, row_id, action, new_value, changed_at)
SELECT 'users', id, 'role_upgrade', new_role, NOW()
FROM updated;
```

> **Edge Case — RETURNING does not work with ON CONFLICT DO NOTHING for skipped rows:**
> ```sql
> INSERT INTO users (name, email)
> VALUES ('Alice', 'alice@example.com')
> ON CONFLICT (email) DO NOTHING
> RETURNING id;
> -- Returns nothing (empty) if the row was skipped — not an error, just no output
> -- Returns the id only if the row was actually inserted
> ```

> **Edge Case — RETURNING reflects the final state after triggers:**
> If a `BEFORE` trigger modifies the row, `RETURNING` returns the post-trigger values, not the values you passed in.

---

## 7. UPSERT (INSERT ... ON CONFLICT)

**Upsert** = **UP**date or in**SERT** — insert a row if it doesn't exist, update it if it does. PostgreSQL implements this with `ON CONFLICT`.

### Syntax

```sql
INSERT INTO table_name (col1, col2)
VALUES (val1, val2)
ON CONFLICT (conflict_target)
DO NOTHING | DO UPDATE SET col = expr;
```

The **conflict target** is the column (or constraint name) that PostgreSQL checks for a duplicate. It must be a `PRIMARY KEY`, `UNIQUE` constraint, or unique index.

---

### ON CONFLICT DO NOTHING

Silently skip the insert if a conflict occurs — no error, no update.

```sql
-- Skip insert if email already exists
INSERT INTO users (name, email)
VALUES ('Alice', 'alice@example.com')
ON CONFLICT (email) DO NOTHING;
```

```sql
-- Bulk insert, skip duplicates
INSERT INTO users (name, email)
VALUES
  ('Alice',   'alice@example.com'),
  ('Bob',     'bob@example.com'),
  ('Charlie', 'charlie@example.com')
ON CONFLICT (email) DO NOTHING;
-- Existing emails are skipped, new ones are inserted
```

```sql
-- Using constraint name instead of column name
INSERT INTO users (name, email)
VALUES ('Alice', 'alice@example.com')
ON CONFLICT ON CONSTRAINT users_email_key DO NOTHING;
```

> **Edge Case:** `DO NOTHING` does not update the row, does not reset the sequence, and `RETURNING` returns nothing for skipped rows. The existing row is completely untouched.

---

### ON CONFLICT DO UPDATE

Update the existing row when a conflict is detected. The special `EXCLUDED` table refers to the row that failed to insert.

```sql
-- Insert or update (full upsert)
INSERT INTO users (name, email, role)
VALUES ('Alice', 'alice@example.com', 'admin')
ON CONFLICT (email)
DO UPDATE SET
  name = EXCLUDED.name,
  role = EXCLUDED.role;
```

```sql
-- Update only specific columns, keep others unchanged
INSERT INTO users (name, email, role, is_active)
VALUES ('Alice', 'alice@example.com', 'admin', true)
ON CONFLICT (email)
DO UPDATE SET
  role      = EXCLUDED.role,
  is_active = EXCLUDED.is_active;
  -- name is NOT updated — keeps the original name
```

```sql
-- Update only if value actually changed (avoid unnecessary writes)
INSERT INTO users (name, email, role)
VALUES ('Alice', 'alice@example.com', 'admin')
ON CONFLICT (email)
DO UPDATE SET
  role = EXCLUDED.role
WHERE users.role IS DISTINCT FROM EXCLUDED.role;
-- Only updates if role actually changed — no-op if same value
```

```sql
-- Track update timestamps
INSERT INTO users (name, email, role)
VALUES ('Alice', 'alice@example.com', 'admin')
ON CONFLICT (email)
DO UPDATE SET
  role       = EXCLUDED.role,
  updated_at = NOW();
```

```sql
-- Increment a counter (e.g. page views)
CREATE TABLE page_views (
  page_url TEXT    PRIMARY KEY,
  views    INTEGER NOT NULL DEFAULT 0
);

INSERT INTO page_views (page_url, views)
VALUES ('/home', 1)
ON CONFLICT (page_url)
DO UPDATE SET views = page_views.views + 1;
-- First visit: inserts with views = 1
-- Subsequent visits: increments existing counter
```

```sql
-- Upsert with RETURNING
INSERT INTO users (name, email, role)
VALUES ('Alice', 'alice@example.com', 'admin')
ON CONFLICT (email)
DO UPDATE SET role = EXCLUDED.role
RETURNING id, name, role, (xmax = 0) AS was_inserted;
-- xmax = 0 means the row was freshly inserted
-- xmax != 0 means the row was updated
```

> **`EXCLUDED` table:** Inside `DO UPDATE SET`, `EXCLUDED` is a virtual table holding the values from the failed INSERT. `users.col` refers to the current value in the table; `EXCLUDED.col` refers to the value you tried to insert.

```sql
-- Demonstrating EXCLUDED vs table reference
INSERT INTO products (sku, name, price)
VALUES ('ABC123', 'Widget', 29.99)
ON CONFLICT (sku)
DO UPDATE SET
  name  = EXCLUDED.name,    -- use the new name being inserted
  price = EXCLUDED.price,   -- use the new price being inserted
  -- keep whichever price is lower:
  price = LEAST(products.price, EXCLUDED.price);
```

---

### Partial Index Conflict Target

If the unique index is a **partial index** (with a `WHERE` clause), you must specify the same condition in the conflict target:

```sql
-- Unique index only on active users
CREATE UNIQUE INDEX users_email_active_idx
  ON users (email)
  WHERE is_active = true;

-- Conflict target must match the partial index condition
INSERT INTO users (name, email, is_active)
VALUES ('Alice', 'alice@example.com', true)
ON CONFLICT (email) WHERE is_active = true
DO UPDATE SET name = EXCLUDED.name;
```

> **Edge Case — Conflict target must be unambiguous:**
> You cannot use `ON CONFLICT DO UPDATE` without a conflict target — Postgres needs to know which constraint to check:
> ```sql
> -- ERROR: no conflict target specified
> INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com')
> ON CONFLICT DO UPDATE SET name = EXCLUDED.name;
>
> -- Must specify the conflict target:
> ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name;
> ```

> **Edge Case — ON CONFLICT only catches the specified constraint:**
> ```sql
> CREATE TABLE users (
>   id    SERIAL PRIMARY KEY,
>   email TEXT UNIQUE,
>   ssn   TEXT UNIQUE
> );
>
> INSERT INTO users (email, ssn) VALUES ('a@b.com', '123-45-6789')
> ON CONFLICT (email) DO NOTHING;
> -- If email conflicts → skipped (DO NOTHING)
> -- If ssn conflicts but email is new → ERROR (ssn conflict not handled)
> ```

> **Edge Case — Upsert is not atomic across concurrent sessions without care:**
> Under high concurrency, two sessions doing upsert on the same key simultaneously can both pass the "conflict" check and race. PostgreSQL handles this correctly with `ON CONFLICT` — it uses advisory locks internally to serialize the conflict resolution. You do not need to add extra locking.

---

## Quick Reference

```sql
-- INSERT
INSERT INTO t (col1, col2) VALUES (v1, v2);
INSERT INTO t (col1, col2) VALUES (v1, v2), (v3, v4), (v5, v6);  -- multi-row
INSERT INTO t (col1, col2) SELECT col1, col2 FROM other WHERE cond;

-- UPDATE
UPDATE t SET col = val WHERE cond;
UPDATE t SET col1 = v1, col2 = v2 WHERE cond;
UPDATE t SET col = col * 1.1 WHERE cond;                          -- expression
UPDATE t SET col = val FROM other o WHERE t.fk = o.id;            -- join

-- DELETE
DELETE FROM t WHERE cond;
DELETE FROM t USING other o WHERE t.fk = o.id AND o.col = val;    -- join

-- TRUNCATE
TRUNCATE t;
TRUNCATE t RESTART IDENTITY;
TRUNCATE t CASCADE;

-- RETURNING (append to INSERT / UPDATE / DELETE)
INSERT INTO t (col) VALUES (val) RETURNING *;
UPDATE t SET col = val WHERE cond RETURNING id, col;
DELETE FROM t WHERE cond RETURNING id;

-- UPSERT
INSERT INTO t (col) VALUES (val)
  ON CONFLICT (col) DO NOTHING;

INSERT INTO t (col1, col2) VALUES (v1, v2)
  ON CONFLICT (col1)
  DO UPDATE SET col2 = EXCLUDED.col2;

INSERT INTO t (col1, col2) VALUES (v1, v2)
  ON CONFLICT (col1)
  DO UPDATE SET col2 = EXCLUDED.col2
  WHERE t.col2 IS DISTINCT FROM EXCLUDED.col2;  -- only update if changed

INSERT INTO t (col1, col2) VALUES (v1, v2)
  ON CONFLICT ON CONSTRAINT constraint_name DO NOTHING;

-- RETURNING + CTE (move rows atomically)
WITH deleted AS (
  DELETE FROM t WHERE cond RETURNING *
)
INSERT INTO archive SELECT * FROM deleted;
```
