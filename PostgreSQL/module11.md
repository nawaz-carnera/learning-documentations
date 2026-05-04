# Module 11: Views and Materialized Views

---

## Table of Contents

- [1. What a View Is and Why](#1-what-a-view-is-and-why)
- [2. Creating a View (CREATE VIEW)](#2-creating-a-view-create-view)
  - [Basic Syntax](#basic-syntax)
  - [Views Over Joins](#views-over-joins)
  - [Views Over Aggregations](#views-over-aggregations)
  - [Views with Schema Qualification](#views-with-schema-qualification)
- [3. Querying a View](#3-querying-a-view)
- [4. Updating a View (CREATE OR REPLACE VIEW)](#4-updating-a-view-create-or-replace-view)
- [5. Dropping a View](#5-dropping-a-view)
- [6. Materialized Views](#6-materialized-views)
  - [Creating a Materialized View](#creating-a-materialized-view)
  - [Querying a Materialized View](#querying-a-materialized-view)
  - [Indexes on Materialized Views](#indexes-on-materialized-views)
  - [When to Use Materialized Views](#when-to-use-materialized-views)
- [7. Refreshing Materialized Views](#7-refreshing-materialized-views)
  - [REFRESH MATERIALIZED VIEW](#refresh-materialized-view)
  - [CONCURRENTLY](#concurrently)
  - [Automating Refresh](#automating-refresh)
- [Quick Reference](#quick-reference)

---

## 1. What a View Is and Why

A **view** is a **named, saved SELECT query** stored in the database. It looks and behaves like a table — you can `SELECT` from it, `JOIN` it, filter it — but it holds no data itself. Every time you query a view, PostgreSQL runs the underlying query fresh.

```
Without a view:                    With a view:
────────────────────────────────   ──────────────────────────
SELECT                             SELECT *
  c.name,                          FROM customer_order_summary
  COUNT(o.id)  AS orders,          WHERE country = 'US';
  SUM(o.total) AS spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE c.country = 'US'
GROUP BY c.id, c.name;
```

### Why Use Views

**1. Simplify complex queries** — wrap a multi-join, multi-aggregate query into a single name that anyone can query with `SELECT * FROM view_name`.

**2. Enforce a consistent "interface"** — the underlying tables can change (columns added, renamed) and you update only the view definition, not every application query.

**3. Security and access control** — grant users access to a view without granting access to the underlying tables. The view can hide sensitive columns (e.g., SSN, salary) or filter rows to only those the user should see.

**4. DRY (Don't Repeat Yourself)** — define business logic (status mappings, calculations, joins) once in a view instead of copy-pasting the same SQL across reports.

**5. Logical separation** — separate the "raw data" layer (tables) from the "business meaning" layer (views).

### What a View Is NOT

| | Regular View | Materialized View | Table |
|---|---|---|---|
| Stores data | No — runs query live | Yes — snapshot stored on disk | Yes |
| Always up to date | Yes — always fresh | No — stale until refreshed | Yes |
| Can be indexed | No | Yes | Yes |
| Write through (INSERT/UPDATE) | Sometimes (limited) | No | Yes |
| Query cost | Full recompute every time | Reads stored data | Reads stored data |

---

## 2. Creating a View (CREATE VIEW)

### Basic Syntax

```sql
CREATE VIEW view_name AS
SELECT ...;

-- With schema
CREATE VIEW schema_name.view_name AS
SELECT ...;

-- With column name aliases (explicitly rename output columns)
CREATE VIEW view_name (col1, col2, col3) AS
SELECT a, b, c FROM table;
```

### Setup

```sql
CREATE TABLE customers (
  id        SERIAL PRIMARY KEY,
  name      TEXT NOT NULL,
  email     TEXT UNIQUE NOT NULL,
  country   TEXT NOT NULL DEFAULT 'US',
  tier      TEXT NOT NULL DEFAULT 'standard',
  is_active BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE orders (
  id          SERIAL PRIMARY KEY,
  customer_id INTEGER REFERENCES customers(id),
  status      TEXT NOT NULL DEFAULT 'pending',
  total       NUMERIC(10,2) NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE order_items (
  id           SERIAL PRIMARY KEY,
  order_id     INTEGER NOT NULL REFERENCES orders(id),
  product_name TEXT NOT NULL,
  quantity     INTEGER NOT NULL,
  unit_price   NUMERIC(10,2) NOT NULL
);

INSERT INTO customers VALUES
  (1,'Alice',  'alice@example.com',  'US','gold',    true),
  (2,'Bob',    'bob@example.com',    'UK','standard', true),
  (3,'Charlie','charlie@example.com','US','standard', true),
  (4,'Dana',   'dana@example.com',   'CA','gold',     false),
  (5,'Eve',    'eve@example.com',    'US','standard', true);

INSERT INTO orders VALUES
  (1,1,'delivered',250.00, NOW() - INTERVAL '60 days'),
  (2,1,'delivered',430.00, NOW() - INTERVAL '30 days'),
  (3,2,'pending',   95.00, NOW() - INTERVAL '5 days'),
  (4,3,'delivered', 60.00, NOW() - INTERVAL '45 days'),
  (5,3,'cancelled',120.00, NOW() - INTERVAL '20 days'),
  (6,4,'delivered',875.00, NOW() - INTERVAL '10 days'),
  (7,5,'pending',   40.00, NOW() - INTERVAL '2 days');

INSERT INTO order_items VALUES
  (1,1,'Laptop',    1,999.99),
  (2,1,'Mouse',     2, 29.99),
  (3,2,'Desk',      1,349.99),
  (4,2,'Chair',     2,199.99),
  (5,3,'Notebook',  3,  4.99),
  (6,4,'Headphones',1,149.99),
  (7,6,'Laptop',    1,999.99),
  (8,6,'Desk',      2,349.99);
```

### Simple View

```sql
-- View: active customers only
CREATE VIEW active_customers AS
SELECT id, name, email, country, tier
FROM customers
WHERE is_active = true;
```

```sql
-- View: delivered orders only
CREATE VIEW delivered_orders AS
SELECT id, customer_id, total, created_at
FROM orders
WHERE status = 'delivered';
```

### Views Over Joins

```sql
-- View: order details with customer name (most common pattern)
CREATE VIEW order_summary AS
SELECT
  o.id              AS order_id,
  c.name            AS customer_name,
  c.country,
  c.tier,
  o.status,
  o.total,
  o.created_at
FROM orders o
JOIN customers c ON o.customer_id = c.id;
```

```sql
-- View: full line-item detail with order context
CREATE VIEW order_line_items AS
SELECT
  o.id                                  AS order_id,
  c.name                                AS customer,
  o.status,
  oi.product_name,
  oi.quantity,
  oi.unit_price,
  oi.quantity * oi.unit_price           AS line_total,
  o.created_at
FROM order_items oi
JOIN orders    o ON oi.order_id    = o.id
JOIN customers c ON o.customer_id  = c.id;
```

### Views Over Aggregations

```sql
-- View: customer spending summary
CREATE VIEW customer_spend_summary AS
SELECT
  c.id                              AS customer_id,
  c.name,
  c.country,
  c.tier,
  COUNT(o.id)                       AS total_orders,
  COUNT(o.id) FILTER
    (WHERE o.status = 'delivered')  AS delivered_orders,
  COALESCE(SUM(o.total)
    FILTER (WHERE o.status = 'delivered'), 0) AS total_spent,
  MAX(o.created_at)                 AS last_order_date
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name, c.country, c.tier;
```

```sql
-- View: daily revenue (useful for dashboards)
CREATE VIEW daily_revenue AS
SELECT
  created_at::DATE          AS day,
  COUNT(*)                  AS order_count,
  SUM(total)                AS revenue,
  ROUND(AVG(total), 2)      AS avg_order_value
FROM orders
WHERE status = 'delivered'
GROUP BY created_at::DATE;
```

### Views with Schema Qualification

```sql
-- Create a reporting schema and put views there
CREATE SCHEMA reporting;

CREATE VIEW reporting.customer_spend_summary AS
SELECT ...;

CREATE VIEW reporting.daily_revenue AS
SELECT ...;

-- Grant analysts access to reporting views but not raw tables
GRANT USAGE ON SCHEMA reporting TO analyst_role;
GRANT SELECT ON ALL TABLES IN SCHEMA reporting TO analyst_role;
-- analyst_role can query views but cannot touch customers/orders directly
```

> **Edge Case — Column names must be unique in a view:**
> ```sql
> -- ERROR: two columns named "id"
> CREATE VIEW order_summary AS
> SELECT o.id, c.id, o.total           -- ambiguous: two "id" columns
> FROM orders o JOIN customers c ON o.customer_id = c.id;
>
> -- Fix: alias to make names unique
> SELECT o.id AS order_id, c.id AS customer_id, o.total ...
> ```

> **Edge Case — `SELECT *` in views is fragile:**
> ```sql
> CREATE VIEW all_customers AS SELECT * FROM customers;
>
> -- Later, a new column "phone" is added to customers:
> ALTER TABLE customers ADD COLUMN phone TEXT;
>
> SELECT * FROM all_customers;
> -- Does NOT include "phone" — the view captures the column list at creation time
>
> -- Fix: recreate the view after schema changes
> CREATE OR REPLACE VIEW all_customers AS SELECT * FROM customers;
> ```

---

## 3. Querying a View

A view is queried exactly like a table — it supports `SELECT`, `WHERE`, `ORDER BY`, `JOIN`, `GROUP BY`, `LIMIT`, and any other clause.

```sql
-- Simple query
SELECT * FROM active_customers;

-- With filter
SELECT * FROM active_customers WHERE country = 'US';

-- With sorting and limit
SELECT * FROM customer_spend_summary
ORDER BY total_spent DESC
LIMIT 5;

-- Join two views
SELECT
  css.name,
  css.total_spent,
  dr.revenue AS company_daily_revenue_today
FROM customer_spend_summary css
CROSS JOIN (
  SELECT revenue FROM daily_revenue WHERE day = CURRENT_DATE
) dr
ORDER BY css.total_spent DESC;

-- Join a view to a table
SELECT
  os.order_id,
  os.customer_name,
  os.total,
  oi.product_name,
  oi.quantity
FROM order_summary os
JOIN order_items oi ON os.order_id = oi.order_id
WHERE os.status = 'delivered'
ORDER BY os.created_at DESC;

-- Aggregate on top of a view
SELECT country, COUNT(*) AS customers, SUM(total_spent) AS country_revenue
FROM customer_spend_summary
GROUP BY country
ORDER BY country_revenue DESC;
```

### Listing Views

```sql
-- In psql
\dv                    -- list all views in current schema
\dv reporting.*        -- list views in reporting schema
\dv+ view_name         -- show view definition

-- Via SQL
SELECT table_name, view_definition
FROM information_schema.views
WHERE table_schema = 'public'
ORDER BY table_name;

-- Show the SQL behind a view
SELECT pg_get_viewdef('customer_spend_summary', true);
```

### How PostgreSQL Executes a View

When you write `SELECT * FROM my_view WHERE country = 'US'`, PostgreSQL **inlines** the view definition:

```sql
-- What you write:
SELECT * FROM active_customers WHERE country = 'US';

-- What PostgreSQL actually runs (view expanded inline):
SELECT id, name, email, country, tier
FROM customers
WHERE is_active = true
  AND country = 'US';     -- ← your filter pushed inside
```

This means the query planner can use indexes on the underlying table — views do not add overhead compared to writing the SQL directly.

> **Edge Case — View performance is identical to inline SQL:** A regular view is just a macro — it has zero storage cost and the planner fully optimizes it. The only case where views add overhead is when they prevent certain planner optimizations (rare, usually with very complex views).

---

## 4. Updating a View (CREATE OR REPLACE VIEW)

### Replace an Existing View

`CREATE OR REPLACE VIEW` rewrites the view definition without dropping it. Dependent objects (other views, grants) are preserved.

```sql
CREATE OR REPLACE VIEW active_customers AS
SELECT id, name, email, country, tier, created_at  -- added created_at
FROM customers
WHERE is_active = true;
```

### Rules for CREATE OR REPLACE

PostgreSQL imposes restrictions on what `CREATE OR REPLACE` can change:

```sql
-- ALLOWED: add new columns (must be added at the end)
CREATE OR REPLACE VIEW active_customers AS
SELECT id, name, email, country, tier, is_active   -- added is_active at end
FROM customers WHERE is_active = true;

-- NOT ALLOWED: change existing column type or name
-- NOT ALLOWED: remove a column
-- NOT ALLOWED: reorder columns
-- These require DROP VIEW then CREATE VIEW
```

> **Edge Case — Adding a column mid-list fails:**
> ```sql
> -- Original view: id, name, email, country, tier
> -- Trying to insert "phone" before "country" → ERROR
> CREATE OR REPLACE VIEW active_customers AS
> SELECT id, name, email, phone, country, tier    -- phone inserted in middle
> FROM customers WHERE is_active = true;
> -- ERROR: cannot change name of view column "country" to "phone"
>
> -- Fix: drop and recreate (check for dependent objects first)
> DROP VIEW active_customers;
> CREATE VIEW active_customers AS SELECT id, name, email, phone, country, tier
>   FROM customers WHERE is_active = true;
> ```

### ALTER VIEW

```sql
-- Rename a view
ALTER VIEW active_customers RENAME TO active_users;

-- Rename a column in a view (PostgreSQL 9.4+)
ALTER VIEW active_customers RENAME COLUMN tier TO membership_tier;

-- Change owner
ALTER VIEW active_customers OWNER TO reporting_user;

-- Set default for a view column (used for updatable views)
ALTER VIEW active_customers ALTER COLUMN country SET DEFAULT 'US';
```

### Updatable Views

PostgreSQL allows `INSERT`, `UPDATE`, and `DELETE` directly on a view if the view meets certain conditions:

**A view is automatically updatable when:**
- It queries exactly one table (no JOINs)
- No `DISTINCT`, `GROUP BY`, `HAVING`, `LIMIT`, `OFFSET`
- No set operations (`UNION`, `INTERSECT`, `EXCEPT`)
- No window functions or aggregate functions
- No subqueries in the column list

```sql
CREATE VIEW us_customers AS
SELECT id, name, email, country, tier
FROM customers
WHERE country = 'US';

-- These DML operations work directly on the view:
INSERT INTO us_customers (name, email, country, tier)
VALUES ('Frank', 'frank@example.com', 'US', 'standard');

UPDATE us_customers SET tier = 'gold' WHERE name = 'Alice';

DELETE FROM us_customers WHERE name = 'Charlie';
```

### WITH CHECK OPTION

Prevents inserting/updating rows through a view if the row would no longer be visible through the view (violates the view's `WHERE` clause):

```sql
CREATE VIEW us_customers AS
SELECT id, name, email, country, tier
FROM customers
WHERE country = 'US'
WITH CHECK OPTION;

-- This would succeed without CHECK OPTION but the inserted row
-- would be invisible through the view immediately after insert:
INSERT INTO us_customers (name, email, country, tier)
VALUES ('Kai', 'kai@example.com', 'UK', 'standard');
-- ERROR: new row violates check option for view "us_customers"
-- (country 'UK' fails the WHERE country = 'US' condition)
```

> **Edge Case — Complex views are not updatable:** If a view uses JOINs, aggregates, or any of the listed disqualifying features, `INSERT`/`UPDATE`/`DELETE` will fail:
> ```sql
> UPDATE order_summary SET status = 'shipped' WHERE order_id = 1;
> -- ERROR: cannot update view "order_summary"
> -- DETAIL: Views containing joins are not automatically updatable.
> -- Fix: use INSTEAD OF triggers or update the underlying table directly
> ```

---

## 5. Dropping a View

```sql
-- Basic drop (fails if other views depend on it)
DROP VIEW view_name;

-- Safe drop (no error if view doesn't exist)
DROP VIEW IF EXISTS view_name;

-- Drop multiple views at once
DROP VIEW IF EXISTS view1, view2, view3;

-- Drop and remove all dependent views too
DROP VIEW view_name CASCADE;
```

```sql
-- Check what depends on a view before dropping
SELECT dependent.relname AS depends_on_view
FROM pg_depend dep
JOIN pg_rewrite rw  ON dep.objid = rw.oid
JOIN pg_class dependent ON rw.ev_class = dependent.oid
JOIN pg_class source     ON dep.refobjid = source.oid
WHERE source.relname = 'active_customers'
  AND dependent.relname != 'active_customers';
```

> **Edge Case — DROP VIEW vs DROP VIEW CASCADE:**
> ```sql
> -- If reporting.customer_report depends on active_customers:
> DROP VIEW active_customers;
> -- ERROR: cannot drop view active_customers because other objects depend on it
> -- DETAIL: view reporting.customer_report depends on view active_customers
>
> -- CASCADE drops active_customers AND customer_report (and anything that depends on that)
> DROP VIEW active_customers CASCADE;
> -- NOTICE: drop cascades to view reporting.customer_report
> ```

---

## 6. Materialized Views

A **materialized view** is like a regular view but it **physically stores the query result on disk**. Queries read the stored data rather than re-executing the query — making reads very fast. The trade-off: the data can become stale and must be manually (or automatically) refreshed.

```
Regular View:            Materialized View:
─────────────────────    ──────────────────────────────────
Query runs live          Data stored on disk as a snapshot
Always fresh             Stale until REFRESH is called
No storage cost          Uses disk space
No index possible        Can be indexed
Good for simple queries  Good for expensive queries
```

### Creating a Materialized View

```sql
CREATE MATERIALIZED VIEW view_name AS
SELECT ...;

-- With NO DATA: creates the structure but doesn't populate it yet
CREATE MATERIALIZED VIEW view_name AS
SELECT ...
WITH NO DATA;

-- To populate later:
REFRESH MATERIALIZED VIEW view_name;
```

### Examples

```sql
-- Expensive aggregation run on every dashboard load — good candidate for matview
CREATE MATERIALIZED VIEW mv_customer_spend AS
SELECT
  c.id                                          AS customer_id,
  c.name,
  c.country,
  c.tier,
  COUNT(o.id)                                   AS total_orders,
  COUNT(o.id) FILTER (WHERE o.status = 'delivered') AS delivered_orders,
  COALESCE(SUM(o.total) FILTER
    (WHERE o.status = 'delivered'), 0)           AS total_spent,
  ROUND(AVG(o.total)
    FILTER (WHERE o.status = 'delivered'), 2)   AS avg_order_value,
  MAX(o.created_at)                             AS last_order_date
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name, c.country, c.tier;
```

```sql
-- Monthly revenue summary — good for finance dashboards
CREATE MATERIALIZED VIEW mv_monthly_revenue AS
SELECT
  DATE_TRUNC('month', created_at)::DATE  AS month,
  COUNT(*)                               AS order_count,
  SUM(total)                             AS revenue,
  ROUND(AVG(total), 2)                   AS avg_order_value,
  COUNT(DISTINCT customer_id)            AS unique_customers
FROM orders
WHERE status = 'delivered'
GROUP BY DATE_TRUNC('month', created_at)::DATE
ORDER BY month;
```

```sql
-- Product performance summary
CREATE MATERIALIZED VIEW mv_product_performance AS
SELECT
  oi.product_name,
  COUNT(DISTINCT oi.order_id)             AS times_ordered,
  SUM(oi.quantity)                        AS total_units_sold,
  SUM(oi.quantity * oi.unit_price)        AS total_revenue,
  ROUND(AVG(oi.unit_price), 2)            AS avg_unit_price
FROM order_items oi
JOIN orders o ON oi.order_id = o.id
WHERE o.status = 'delivered'
GROUP BY oi.product_name
ORDER BY total_revenue DESC;
```

### Querying a Materialized View

Identical to querying a regular view or table:

```sql
SELECT * FROM mv_customer_spend ORDER BY total_spent DESC;

SELECT * FROM mv_monthly_revenue WHERE month >= '2024-01-01';

SELECT country, SUM(total_spent) AS country_revenue
FROM mv_customer_spend
GROUP BY country
ORDER BY country_revenue DESC;
```

### Indexes on Materialized Views

Unlike regular views, materialized views can have **indexes** — this is their major performance advantage:

```sql
-- Index for fast customer lookups
CREATE INDEX idx_mv_customer_spend_customer_id
  ON mv_customer_spend (customer_id);

-- Index for filtering by country
CREATE INDEX idx_mv_customer_spend_country
  ON mv_customer_spend (country);

-- Index for sorting by spend
CREATE INDEX idx_mv_customer_spend_total_spent
  ON mv_customer_spend (total_spent DESC);

-- Unique index — required for CONCURRENT refresh (see section 7)
CREATE UNIQUE INDEX idx_mv_customer_spend_unique
  ON mv_customer_spend (customer_id);
```

### When to Use Materialized Views

Use a materialized view when:

| Signal | Example |
|---|---|
| The query is slow and read-only results are acceptable with slight staleness | Analytics dashboards, weekly reports |
| Same expensive query is run many times per second | High-traffic reporting endpoints |
| Query involves large aggregations, multi-level joins, or window functions | Revenue rollups, cohort analysis |
| You need to index the result | Filter/sort on aggregated data |
| The underlying data changes infrequently | Reference data, daily batch processing |

Avoid when:

| Signal | Example |
|---|---|
| Data must always be perfectly up to date | Live inventory, real-time balances |
| Underlying data changes constantly | Per-second event streams |
| Refresh cost is as expensive as just re-querying | Tiny tables |

```sql
-- Rule of thumb: if this query takes > 500ms and is called frequently,
-- consider a materialized view:
EXPLAIN ANALYZE
SELECT c.name, SUM(o.total)
FROM customers c JOIN orders o ON c.id = o.customer_id
GROUP BY c.name;
-- Planning Time: 2ms, Execution Time: 3400ms ← good candidate
```

### Listing Materialized Views

```sql
-- In psql
\dm          -- list materialized views
\dm+         -- with size and description

-- Via SQL
SELECT matviewname, matviewowner, ispopulated
FROM pg_matviews
WHERE schemaname = 'public';
```

---

## 7. Refreshing Materialized Views

A materialized view's data is a **snapshot taken at creation time** (or last refresh time). It does not update automatically when underlying tables change. You must explicitly refresh it.

### REFRESH MATERIALIZED VIEW

```sql
-- Full refresh: replaces all data with fresh query results
REFRESH MATERIALIZED VIEW mv_customer_spend;

-- Refresh a view in a specific schema
REFRESH MATERIALIZED VIEW reporting.mv_monthly_revenue;
```

**What happens during a full refresh:**
1. PostgreSQL acquires an **exclusive lock** on the materialized view
2. The stored data is completely replaced
3. All queries to the view are **blocked** during the refresh
4. Lock is released when refresh completes

> **Edge Case — Refreshing a view created WITH NO DATA:**
> ```sql
> CREATE MATERIALIZED VIEW mv_report AS SELECT ... WITH NO DATA;
>
> SELECT * FROM mv_report;
> -- ERROR: materialized view "mv_report" has not been populated
>
> -- Must refresh before first use:
> REFRESH MATERIALIZED VIEW mv_report;
> ```

---

### CONCURRENTLY

`REFRESH MATERIALIZED VIEW CONCURRENTLY` refreshes without locking out readers — the old data remains available while the new data is computed in the background. Once ready, it swaps atomically.

```sql
-- Non-blocking refresh (readers can still query the view)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_customer_spend;
```

**Requirements for CONCURRENTLY:**
1. The materialized view must have at least one **unique index**
2. Takes longer than a regular refresh (does a diff + swap instead of full replace)
3. Cannot be used on a view that has never been populated

```sql
-- Step 1: Create matview
CREATE MATERIALIZED VIEW mv_customer_spend AS
SELECT c.id AS customer_id, c.name, SUM(o.total) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name;

-- Step 2: Add unique index (required for CONCURRENTLY)
CREATE UNIQUE INDEX ON mv_customer_spend (customer_id);

-- Step 3: Now CONCURRENT refresh is available
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_customer_spend;
```

### Full vs CONCURRENTLY Refresh

| | REFRESH | REFRESH CONCURRENTLY |
|---|---|---|
| Blocks readers | Yes (exclusive lock) | No (readers continue) |
| Speed | Faster (full replace) | Slower (computes diff) |
| Requires unique index | No | Yes |
| Works on unpopulated view | Yes | No |
| Use when | Nightly batch jobs, low-traffic | High-traffic, always-on dashboards |

---

### Automating Refresh

PostgreSQL has no built-in scheduler — use one of these approaches:

**Option 1: pg_cron extension (most common)**

```sql
-- Install pg_cron (requires superuser, listed in shared_preload_libraries)
CREATE EXTENSION pg_cron;

-- Refresh every hour
SELECT cron.schedule(
  'refresh-mv-customer-spend',        -- job name
  '0 * * * *',                        -- cron expression: top of every hour
  'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_customer_spend'
);

-- Refresh every night at 2am
SELECT cron.schedule(
  'refresh-mv-monthly-revenue',
  '0 2 * * *',
  'REFRESH MATERIALIZED VIEW mv_monthly_revenue'
);

-- List scheduled jobs
SELECT * FROM cron.job;

-- Remove a scheduled job
SELECT cron.unschedule('refresh-mv-customer-spend');
```

**Option 2: Trigger-based refresh (refresh on each write)**

```sql
-- Refresh the materialized view whenever orders table changes
-- WARNING: this is synchronous and can slow down writes — only for low-write tables
CREATE OR REPLACE FUNCTION refresh_mv_customer_spend()
RETURNS TRIGGER AS $$
BEGIN
  REFRESH MATERIALIZED VIEW CONCURRENTLY mv_customer_spend;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_refresh_mv_customer_spend
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH STATEMENT
EXECUTE FUNCTION refresh_mv_customer_spend();
```

**Option 3: Application-level refresh**

```sql
-- Called from application code after a batch job completes
-- e.g., after nightly ETL pipeline finishes:
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_monthly_revenue;
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_product_performance;
```

**Option 4: OS cron job (psql)**

```bash
# /etc/cron.d/refresh-matviews
0 * * * * postgres psql -d mydb -c "REFRESH MATERIALIZED VIEW CONCURRENTLY mv_customer_spend;"
```

### Check Last Refresh Time

PostgreSQL does not track last refresh time natively. Common workaround: add a refresh log table:

```sql
CREATE TABLE matview_refresh_log (
  view_name    TEXT NOT NULL,
  refreshed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- After each refresh, log it:
REFRESH MATERIALIZED VIEW mv_customer_spend;
INSERT INTO matview_refresh_log (view_name) VALUES ('mv_customer_spend');

-- Check last refresh
SELECT view_name, MAX(refreshed_at) AS last_refreshed
FROM matview_refresh_log
GROUP BY view_name;
```

> **Edge Case — Schema changes break materialized views:**
> ```sql
> -- If a column used in the matview definition is dropped from the base table:
> ALTER TABLE orders DROP COLUMN total;
>
> REFRESH MATERIALIZED VIEW mv_customer_spend;
> -- ERROR: column "total" of relation "orders" does not exist
>
> -- Fix: DROP and recreate the materialized view with the new schema
> DROP MATERIALIZED VIEW mv_customer_spend;
> CREATE MATERIALIZED VIEW mv_customer_spend AS ...;  -- updated definition
> ```

> **Edge Case — CONCURRENTLY on a large view can be slow and I/O intensive:** It computes the full new result set, diffs it against the old one, then applies changes. On very large materialized views, consider scheduling refreshes during low-traffic windows even with CONCURRENTLY.

> **Edge Case — Dropped the only unique index? CONCURRENTLY breaks:**
> ```sql
> DROP INDEX idx_mv_customer_spend_unique;
>
> REFRESH MATERIALIZED VIEW CONCURRENTLY mv_customer_spend;
> -- ERROR: cannot refresh materialized view concurrently without a unique index
>
> -- Fix: recreate the unique index before using CONCURRENTLY again
> CREATE UNIQUE INDEX idx_mv_customer_spend_unique ON mv_customer_spend (customer_id);
> ```

---

## Quick Reference

```sql
-- Create a regular view
CREATE VIEW vw_name AS SELECT ...;
CREATE VIEW schema.vw_name AS SELECT ...;

-- Replace (update) a view — columns can only be added at end, not removed/renamed
CREATE OR REPLACE VIEW vw_name AS SELECT ...;

-- Alter a view
ALTER VIEW vw_name RENAME TO new_name;
ALTER VIEW vw_name RENAME COLUMN old_col TO new_col;
ALTER VIEW vw_name OWNER TO new_owner;

-- Updatable view guard
CREATE VIEW vw_name AS SELECT ... WHERE cond
WITH CHECK OPTION;                              -- reject rows that violate WHERE

-- Drop a view
DROP VIEW IF EXISTS vw_name;
DROP VIEW vw_name CASCADE;                     -- also drops dependent views

-- List views
\dv                                            -- psql: all views in schema
\dv+                                           -- with definition
SELECT table_name FROM information_schema.views WHERE table_schema = 'public';
SELECT pg_get_viewdef('vw_name', true);        -- show SQL definition

-- Create a materialized view
CREATE MATERIALIZED VIEW mv_name AS SELECT ...;
CREATE MATERIALIZED VIEW mv_name AS SELECT ... WITH NO DATA;

-- Refresh
REFRESH MATERIALIZED VIEW mv_name;                     -- blocks readers
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_name;        -- non-blocking (needs unique index)

-- Index on matview (enables fast queries + CONCURRENTLY refresh)
CREATE UNIQUE INDEX ON mv_name (id_col);               -- required for CONCURRENTLY
CREATE INDEX ON mv_name (filter_col);

-- Drop a materialized view
DROP MATERIALIZED VIEW IF EXISTS mv_name;
DROP MATERIALIZED VIEW mv_name CASCADE;

-- List materialized views
\dm                                                    -- psql
\dm+                                                   -- with size
SELECT matviewname, ispopulated FROM pg_matviews WHERE schemaname = 'public';

-- Schedule refresh with pg_cron
SELECT cron.schedule('job-name', '0 * * * *', 'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_name');
SELECT cron.unschedule('job-name');
```
