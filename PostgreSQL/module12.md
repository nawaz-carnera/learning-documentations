# Module 12: Indexes and Query Performance

---

## Table of Contents

- [1. What an Index Is (B-tree)](#1-what-an-index-is-b-tree)
  - [The Problem Without an Index](#the-problem-without-an-index)
  - [How a B-tree Index Works](#how-a-b-tree-index-works)
  - [Other Index Types in PostgreSQL](#other-index-types-in-postgresql)
- [2. Creating an Index (CREATE INDEX)](#2-creating-an-index-create-index)
  - [Basic Syntax](#basic-syntax)
  - [Naming Conventions](#naming-conventions)
  - [Concurrent Index Creation](#concurrent-index-creation)
- [3. When to Add an Index](#3-when-to-add-an-index)
  - [Foreign Key Columns](#foreign-key-columns)
  - [Frequently Filtered Columns](#frequently-filtered-columns)
  - [Frequently Sorted or Joined Columns](#frequently-sorted-or-joined-columns)
- [4. When NOT to Add an Index](#4-when-not-to-add-an-index)
- [5. Composite (Multi-Column) Indexes](#5-composite-multi-column-indexes)
  - [Column Order Matters](#column-order-matters)
  - [Index-Only Scans (Covering Indexes)](#index-only-scans-covering-indexes)
- [6. Unique Indexes](#6-unique-indexes)
- [7. Partial Indexes](#7-partial-indexes)
- [8. Dropping an Index](#8-dropping-an-index)
- [9. EXPLAIN Basics](#9-explain-basics)
  - [Sequential Scan](#sequential-scan)
  - [Index Scan](#index-scan)
  - [Bitmap Index Scan](#bitmap-index-scan)
  - [Reading EXPLAIN Output](#reading-explain-output)
- [10. EXPLAIN ANALYZE](#10-explain-analyze)
  - [Actual vs Estimated Rows](#actual-vs-estimated-rows)
  - [Key Metrics to Read](#key-metrics-to-read)
  - [EXPLAIN Options](#explain-options)
- [Quick Reference](#quick-reference)

---

## 1. What an Index Is (B-tree)

### The Problem Without an Index

Without an index, PostgreSQL performs a **sequential scan** — it reads every single row in the table from start to finish to find the ones matching your query condition.

```sql
-- Table with 1,000,000 orders
SELECT * FROM orders WHERE customer_id = 42;

-- Without index: reads all 1,000,000 rows to find the ~10 that match
-- Like finding a name in a phone book by reading every entry front to back
```

An **index** is a separate data structure that PostgreSQL maintains alongside the table. It maps column values to the physical location of rows on disk — so instead of reading every row, PostgreSQL jumps directly to the relevant rows.

```
Without index:  Read 1,000,000 rows → return 10 matches   (slow)
With index:     Look up 42 in index → jump to 10 rows     (fast)
```

### How a B-tree Index Works

The default (and most common) index type is **B-tree** (Balanced Tree). It organizes the indexed values in a sorted tree structure:

```
                    [500]
                   /     \
           [250]            [750]
          /     \          /     \
      [125]   [375]    [625]   [875]
      / \     / \      / \     / \
   [100][150][300][400][600][650][800][950]
      ↓    ↓    ↓   ↓    ↓   ↓    ↓   ↓
    rows  rows rows rows rows rows rows rows
```

- **Balanced** — every leaf is the same depth, so lookups take the same time regardless of value
- **Sorted** — supports equality (`=`), range (`>`, `<`, `BETWEEN`), `ORDER BY`, and prefix `LIKE`
- **Self-balancing** — automatically rebalances as rows are inserted and deleted
- Typical depth: 3–4 levels even for millions of rows (each lookup = 3–4 disk reads vs millions)

```sql
-- B-tree index supports all of these:
WHERE customer_id = 42           -- equality
WHERE total > 100                -- range
WHERE total BETWEEN 50 AND 500   -- range
WHERE name LIKE 'Ali%'           -- prefix (left-anchored only)
ORDER BY created_at              -- sort (can avoid sort step entirely)
JOIN orders ON orders.customer_id = users.id  -- join
```

```sql
-- B-tree index does NOT help with:
WHERE name LIKE '%alice%'        -- leading wildcard — use GIN/trigram
WHERE data @> '{"key": "val"}'   -- JSON containment — use GIN
WHERE location <-> point         -- nearest-neighbor — use GiST
```

### Other Index Types in PostgreSQL

| Type | Best For | Example Use |
|---|---|---|
| **B-tree** (default) | Equality, range, sort, joins | `id`, `email`, `created_at`, `status` |
| **Hash** | Equality only (`=`) — slightly faster than B-tree for pure equality | `session_token`, `uuid` |
| **GIN** | Contains/overlap — arrays, JSONB, full-text search | `tags @> '{sale}'`, `metadata @> '{}'`, `tsvector` |
| **GiST** | Geometric/spatial data, nearest-neighbor, range types | PostGIS, `tsrange`, `ip4r` |
| **BRIN** | Very large tables with naturally ordered data | Time-series, append-only logs |
| **SP-GiST** | Non-balanced tree structures | IP ranges, phone numbers |

```sql
-- GIN index for JSONB queries
CREATE INDEX idx_products_metadata ON products USING GIN (metadata);

-- GIN index for full-text search
CREATE INDEX idx_articles_fts ON articles USING GIN (to_tsvector('english', body));

-- BRIN for large time-series table (tiny, fast for range scans)
CREATE INDEX idx_events_created_brin ON events USING BRIN (created_at);

-- Hash index (equality-only, PostgreSQL 10+ crash-safe)
CREATE INDEX idx_sessions_token ON sessions USING HASH (token);
```

---

## 2. Creating an Index (CREATE INDEX)

### Basic Syntax

```sql
-- Basic B-tree index (default)
CREATE INDEX ON table_name (column_name);

-- Named index (recommended — easier to manage)
CREATE INDEX index_name ON table_name (column_name);

-- Specific index type
CREATE INDEX index_name ON table_name USING index_type (column_name);

-- With sort order
CREATE INDEX index_name ON table_name (column_name DESC);
CREATE INDEX index_name ON table_name (column_name ASC NULLS LAST);
```

### Setup

```sql
CREATE TABLE customers (
  id         SERIAL PRIMARY KEY,
  name       TEXT NOT NULL,
  email      TEXT UNIQUE NOT NULL,
  country    TEXT NOT NULL DEFAULT 'US',
  tier       TEXT NOT NULL DEFAULT 'standard',
  is_active  BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
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
  product_id   INTEGER NOT NULL,
  product_name TEXT NOT NULL,
  quantity     INTEGER NOT NULL,
  unit_price   NUMERIC(10,2) NOT NULL
);

-- Creating indexes on the tables above
CREATE INDEX idx_orders_customer_id  ON orders (customer_id);
CREATE INDEX idx_orders_status       ON orders (status);
CREATE INDEX idx_orders_created_at   ON orders (created_at DESC);
CREATE INDEX idx_customers_country   ON customers (country);
CREATE INDEX idx_customers_email     ON customers (email);
CREATE INDEX idx_order_items_order_id   ON order_items (order_id);
CREATE INDEX idx_order_items_product_id ON order_items (product_id);
```

### Naming Conventions

A consistent naming convention makes it easy to identify indexes:

```
idx_{table}_{column(s)}[_{type}]

Examples:
  idx_orders_customer_id           -- single column
  idx_orders_status_created_at     -- composite
  idx_orders_customer_id_status    -- composite
  idx_orders_total_partial         -- partial index
  idx_customers_email_unique       -- unique index
  idx_products_metadata_gin        -- GIN index
```

### Concurrent Index Creation

By default, `CREATE INDEX` **locks the table** for writes while the index is built. On large production tables, this can block inserts/updates for minutes.

Use `CREATE INDEX CONCURRENTLY` to build the index without locking:

```sql
-- Non-blocking index creation (safe for production)
CREATE INDEX CONCURRENTLY idx_orders_customer_id ON orders (customer_id);
```

| | CREATE INDEX | CREATE INDEX CONCURRENTLY |
|---|---|---|
| Write lock | Yes — blocks INSERT/UPDATE/DELETE | No — table stays writable |
| Speed | Faster | ~2× slower (multiple passes) |
| Transaction safe | Yes | Cannot run inside a transaction |
| On failure | Index removed cleanly | May leave invalid index (needs DROP) |

```sql
-- If CONCURRENTLY fails, it may leave an invalid index:
SELECT indexname, indisvalid
FROM pg_indexes
JOIN pg_index ON indexrelid = (
  SELECT oid FROM pg_class WHERE relname = indexname
)
WHERE tablename = 'orders';

-- Drop the invalid index and retry:
DROP INDEX CONCURRENTLY idx_orders_customer_id;
CREATE INDEX CONCURRENTLY idx_orders_customer_id ON orders (customer_id);
```

---

## 3. When to Add an Index

### Foreign Key Columns

PostgreSQL does **not** automatically create an index on foreign key columns (only the referenced PK gets one). Always index FK columns — they are used in every JOIN and ON DELETE operation.

```sql
-- FK columns without indexes cause sequential scans on every JOIN
-- and slow ON DELETE CASCADE operations

-- Add indexes on all FK columns:
CREATE INDEX idx_orders_customer_id    ON orders (customer_id);
CREATE INDEX idx_order_items_order_id  ON order_items (order_id);
CREATE INDEX idx_order_items_product_id ON order_items (product_id);
CREATE INDEX idx_employees_manager_id  ON employees (manager_id);
```

> **Edge Case — ON DELETE CASCADE without FK index is especially dangerous:**
> When you `DELETE FROM customers WHERE id = 1`, PostgreSQL must find all child rows in `orders` with `customer_id = 1`. Without an index on `orders.customer_id`, it scans the entire orders table — for every parent row deleted. With 10,000 customers deleted in a batch, that's 10,000 full table scans.

### Frequently Filtered Columns

Index columns that appear often in `WHERE`, `HAVING`, and `JOIN ON`:

```sql
-- Common query patterns → candidate index columns

-- WHERE status = 'pending'
CREATE INDEX idx_orders_status ON orders (status);

-- WHERE created_at >= '2024-01-01'
CREATE INDEX idx_orders_created_at ON orders (created_at);

-- WHERE country = 'US' AND is_active = true
CREATE INDEX idx_customers_country_active ON customers (country, is_active);

-- WHERE email = 'alice@example.com'  (already covered by UNIQUE constraint index)
-- No additional index needed — UNIQUE creates one automatically
```

### Frequently Sorted or Joined Columns

```sql
-- ORDER BY created_at DESC → index with DESC to match sort direction
CREATE INDEX idx_orders_created_at_desc ON orders (created_at DESC);

-- JOIN orders o ON o.customer_id = c.id  → index on the FK side
CREATE INDEX idx_orders_customer_id ON orders (customer_id);

-- ORDER BY last_name, first_name → composite index in same order
CREATE INDEX idx_employees_name ON employees (last_name, first_name);
```

### Functional Indexes

Index the result of an expression — useful when queries use functions on columns:

```sql
-- Query: WHERE LOWER(email) = 'alice@example.com'
-- Normal index on email won't help (function changes the value)
CREATE INDEX idx_customers_email_lower ON customers (LOWER(email));

-- Query: WHERE DATE_TRUNC('month', created_at) = '2024-01-01'
CREATE INDEX idx_orders_month ON orders (DATE_TRUNC('month', created_at));

-- Query: WHERE EXTRACT(YEAR FROM created_at) = 2024
CREATE INDEX idx_orders_year ON orders (EXTRACT(YEAR FROM created_at));

-- Query: WHERE LENGTH(description) > 1000
CREATE INDEX idx_products_desc_length ON products (LENGTH(description));
```

---

## 4. When NOT to Add an Index

More indexes is not always better. Each index has costs:

- **Write overhead:** Every `INSERT`, `UPDATE`, and `DELETE` must update all indexes on that table
- **Storage:** Indexes take disk space (sometimes 20–100% of the table size)
- **Query planner confusion:** Too many indexes can mislead the planner into choosing a bad plan

### Do Not Index Small Tables

```sql
-- Table with 500 rows: sequential scan is FASTER than index scan
-- Reading 500 rows sequentially = 1–2 disk pages
-- Index lookup = 3–4 disk reads for the B-tree + 1 for the row = overhead

-- Threshold: PostgreSQL typically switches to sequential scan
-- below ~1–5% of table rows being returned (depends on table size)
-- For tiny tables, it always uses sequential scan regardless of indexes
```

### Do Not Index Columns with Low Cardinality

**Cardinality** = number of distinct values. Low-cardinality columns return many rows per lookup — the index is rarely selective enough to be useful.

```sql
-- BAD index: is_active (only 2 values: true/false)
-- If 95% of rows are is_active = true, the index returns 95% of the table
-- PostgreSQL will ignore it and do a sequential scan anyway
CREATE INDEX idx_users_is_active ON users (is_active);  -- wasteful

-- BAD index: gender (3–4 values)
CREATE INDEX idx_employees_gender ON employees (gender);  -- wasteful

-- BAD index: status with only 3 values and 80% rows in 'active'
CREATE INDEX idx_orders_status ON orders (status);  -- marginal at best

-- EXCEPTION: use a PARTIAL index to target the rare/selective value
CREATE INDEX idx_orders_pending ON orders (id) WHERE status = 'pending';
-- Only indexes pending orders (e.g., 2% of rows) — very selective, very useful
```

### Do Not Over-Index Write-Heavy Tables

```sql
-- A table receiving 10,000 inserts per second:
-- Each index adds ~10–30% overhead per write
-- 5 indexes = 50–150% write overhead

-- events table: append-only, insert-heavy, rarely queried by single row
-- BAD: indexing every column
CREATE INDEX ON events (user_id);
CREATE INDEX ON events (event_type);
CREATE INDEX ON events (session_id);
CREATE INDEX ON events (created_at);
CREATE INDEX ON events (ip_address);  -- too many, write performance suffers

-- BETTER: index only what's actually queried, use BRIN for time column
CREATE INDEX ON events (user_id);
CREATE INDEX ON events (event_type);
CREATE INDEX ON events USING BRIN (created_at);  -- tiny, fast for time ranges
```

### Summary: Index Decision Checklist

```
ASK BEFORE CREATING AN INDEX:
✓ Is the table large enough to benefit? (> 1,000–10,000 rows)
✓ Is the column selective enough? (many distinct values)
✓ Is this query run frequently? (not a one-off report)
✓ Does EXPLAIN show a sequential scan you want to eliminate?
✗ Is the table write-heavy with tolerable read latency?
✗ Does the query return > 10–20% of the table? (index won't help)
✗ Is the column rarely used in WHERE/JOIN/ORDER BY?
```

---

## 5. Composite (Multi-Column) Indexes

A composite index covers **two or more columns** — useful when queries filter or sort on multiple columns together.

```sql
CREATE INDEX index_name ON table_name (col1, col2, col3);
```

### Column Order Matters

The **leftmost column rule**: a composite index can be used for queries that filter on:
- The first column alone
- The first + second columns together
- The first + second + third columns together
- etc.

It **cannot** be used efficiently for queries that skip the first column.

```sql
CREATE INDEX idx_orders_customer_status ON orders (customer_id, status);

-- USES the index (filters on leading column):
WHERE customer_id = 5                        -- ✓ first column
WHERE customer_id = 5 AND status = 'pending' -- ✓ first + second column

-- Does NOT use the index efficiently:
WHERE status = 'pending'                     -- ✗ skips first column (full scan or bitmap)
```

```sql
-- Practical example: most common query patterns drive index design
-- Query 1: WHERE country = 'US' AND tier = 'gold'
-- Query 2: WHERE country = 'US'
-- Query 3: WHERE tier = 'gold'  ← less common

-- One composite index handles Query 1 and Query 2:
CREATE INDEX idx_customers_country_tier ON customers (country, tier);
-- Query 1: ✓ uses both columns
-- Query 2: ✓ uses first column only
-- Query 3: ✗ can't use — tier is not the leading column
--          → add a separate index if Query 3 is also frequent:
CREATE INDEX idx_customers_tier ON customers (tier);
```

```sql
-- Sort optimization: index column order should match ORDER BY order
-- Query: ORDER BY last_name ASC, first_name ASC
CREATE INDEX idx_employees_name ON employees (last_name ASC, first_name ASC);

-- Query: ORDER BY created_at DESC, id DESC (latest first, tiebreak by id)
CREATE INDEX idx_orders_recent ON orders (created_at DESC, id DESC);
```

### Index-Only Scans (Covering Indexes)

If a query only needs columns that are all in the index, PostgreSQL can satisfy it entirely from the index without reading the table at all — called an **index-only scan**.

```sql
-- Query needs only customer_id and status — both in the index
SELECT customer_id, status FROM orders WHERE customer_id = 5;
-- With idx_orders_customer_status (customer_id, status):
-- → Index-Only Scan: reads just the index, never touches the table rows

-- "Covering index" — deliberately include extra columns to enable index-only scans
CREATE INDEX idx_orders_customer_status_total
  ON orders (customer_id, status, total);
-- Now this query is index-only too:
SELECT customer_id, status, total FROM orders WHERE customer_id = 5;
```

> **Edge Case — Index-only scans require a clean visibility map:** PostgreSQL's MVCC means even index-only scans occasionally need to check the heap for row visibility. Run `VACUUM` regularly to keep the visibility map current and maximize index-only scan usage.

> **Edge Case — INCLUDE columns (PostgreSQL 11+):** A cleaner way to add "payload" columns to an index without affecting the sort order:
> ```sql
> CREATE INDEX idx_orders_customer_covering
>   ON orders (customer_id)
>   INCLUDE (status, total, created_at);
> -- customer_id is the search key; status/total/created_at are just carried along
> -- Queries filtering on customer_id that also SELECT status/total/created_at
> -- get index-only scans
> ```

---

## 6. Unique Indexes

A unique index enforces that all values in the indexed column(s) are distinct. It's the mechanism behind `UNIQUE` constraints.

```sql
-- Standalone unique index (same effect as UNIQUE constraint)
CREATE UNIQUE INDEX idx_customers_email_unique ON customers (email);

-- Composite unique index — combination must be unique
CREATE UNIQUE INDEX idx_user_roles_unique
  ON user_roles (user_id, role_id);

-- With schema
CREATE UNIQUE INDEX ON hr.employees (employee_number);
```

### UNIQUE Constraint vs UNIQUE Index

```sql
-- These two are equivalent — both create a unique B-tree index:
ALTER TABLE customers ADD CONSTRAINT customers_email_key UNIQUE (email);
CREATE UNIQUE INDEX customers_email_key ON customers (email);

-- The constraint form is preferred for:
-- - Self-documenting intent ("this is a data rule")
-- - Being referenceable by FOREIGN KEY constraints
-- The index form is preferred for:
-- - Partial unique indexes (constraints can't be partial)
-- - CONCURRENTLY creation
```

### Partial Unique Index

```sql
-- Only one active record per user (allow multiple inactive/deleted)
CREATE UNIQUE INDEX idx_subscriptions_active_user
  ON subscriptions (user_id)
  WHERE status = 'active';

-- INSERT INTO subscriptions (user_id, status) VALUES (1, 'active');  → OK first time
-- INSERT INTO subscriptions (user_id, status) VALUES (1, 'active');  → ERROR: duplicate
-- INSERT INTO subscriptions (user_id, status) VALUES (1, 'cancelled'); → OK (not 'active')

-- One pending job per queue slot — cancelled/done jobs don't count
CREATE UNIQUE INDEX idx_jobs_pending_slot
  ON jobs (queue_name, slot_id)
  WHERE status = 'pending';
```

---

## 7. Partial Indexes

A **partial index** only indexes rows that satisfy a `WHERE` condition. This makes the index smaller, faster, and more selective.

```sql
CREATE INDEX index_name ON table_name (column) WHERE condition;
```

```sql
-- Index only pending orders (not the millions of historical delivered/cancelled ones)
CREATE INDEX idx_orders_pending ON orders (customer_id) WHERE status = 'pending';

-- For the query: WHERE customer_id = 5 AND status = 'pending'
-- → Uses the partial index (tiny — only pending orders)
-- vs a full index on (customer_id, status) which includes all historical orders

-- Index only active users (skip the 90% inactive accounts)
CREATE INDEX idx_users_active_email ON users (email) WHERE is_active = true;

-- Index only non-NULL optional columns
CREATE INDEX idx_orders_coupon ON orders (coupon_code) WHERE coupon_code IS NOT NULL;
-- Without partial: index stores NULL for 95% of rows (wasteful)
-- With partial: index only stores the 5% of rows that have a coupon
```

> **Edge Case — The query must include the partial index condition for the planner to use it:**
> ```sql
> CREATE INDEX idx_orders_pending ON orders (customer_id) WHERE status = 'pending';
>
> -- Uses the index:
> WHERE customer_id = 5 AND status = 'pending'
>
> -- Does NOT use the index (condition not present):
> WHERE customer_id = 5
> -- PostgreSQL doesn't know status = 'pending' unless you tell it
> ```

---

## 8. Dropping an Index

```sql
-- Basic drop
DROP INDEX index_name;

-- Safe drop (no error if not exists)
DROP INDEX IF EXISTS index_name;

-- Non-blocking drop (safe for production)
DROP INDEX CONCURRENTLY index_name;
DROP INDEX CONCURRENTLY IF EXISTS index_name;

-- Cannot drop the index backing a constraint directly
-- Must drop the constraint instead:
ALTER TABLE customers DROP CONSTRAINT customers_email_key;
-- This also drops the underlying unique index
```

### Find Unused Indexes

PostgreSQL tracks index usage statistics. Unused indexes waste disk space and slow writes with no read benefit:

```sql
-- Indexes that have NEVER been used since last stats reset
SELECT
  schemaname,
  tablename,
  indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
  idx_scan    AS times_used
FROM pg_stat_user_indexes
JOIN pg_index USING (indexrelid)
WHERE idx_scan = 0
  AND NOT indisprimary
  AND NOT indisunique
ORDER BY pg_relation_size(indexrelid) DESC;
```

```sql
-- All indexes with usage stats — identify rarely used ones
SELECT
  schemaname,
  tablename,
  indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) AS size,
  idx_scan      AS scans,
  idx_tup_read  AS tuples_read
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC, pg_relation_size(indexrelid) DESC;
```

> **Edge Case — Stats reset on server restart:** Index scan counts (`idx_scan`) reset to 0 when PostgreSQL restarts or when `pg_stat_reset()` is called. "0 scans" after a restart doesn't mean the index was never useful — wait for a representative period of normal traffic before dropping "unused" indexes.

> **Edge Case — Bloated indexes:** Over time, B-tree indexes accumulate dead entries from updates/deletes. `REINDEX` rebuilds the index and reclaims space:
> ```sql
> REINDEX INDEX idx_orders_customer_id;       -- blocks reads + writes
> REINDEX INDEX CONCURRENTLY idx_orders_customer_id;  -- non-blocking (PG 12+)
> REINDEX TABLE orders;                       -- rebuilds all indexes on the table
> ```

---

## 9. EXPLAIN Basics

`EXPLAIN` shows PostgreSQL's **query execution plan** — how it intends to retrieve the data. No data is actually fetched; it's purely a planning preview.

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 5;
```

### Sequential Scan

A **Seq Scan** reads every row in the table from start to finish. Used when:
- No useful index exists
- The query returns a large fraction of the table
- The table is very small

```sql
EXPLAIN SELECT * FROM orders WHERE status = 'pending';
```
```
QUERY PLAN
──────────────────────────────────────────────────────────────
Seq Scan on orders  (cost=0.00..24.50 rows=12 width=52)
  Filter: (status = 'pending'::text)
```

- `cost=0.00..24.50` — estimated startup cost `..` estimated total cost (in arbitrary planner units)
- `rows=12` — estimated number of rows returned
- `width=52` — estimated average row size in bytes
- `Filter` — condition applied to each row during the scan

### Index Scan

An **Index Scan** uses the B-tree to find matching rows directly. Used when:
- A suitable index exists
- The query returns a small fraction of rows (selective)

```sql
-- First create the index
CREATE INDEX idx_orders_customer_id ON orders (customer_id);

EXPLAIN SELECT * FROM orders WHERE customer_id = 5;
```
```
QUERY PLAN
──────────────────────────────────────────────────────────────
Index Scan using idx_orders_customer_id on orders
  (cost=0.28..8.30 rows=3 width=52)
  Index Cond: (customer_id = 5)
```

- `Index Scan using idx_orders_customer_id` — identified which index is used
- `Index Cond` — the condition matched via the index (vs `Filter` which is post-scan)

### Bitmap Index Scan

A **Bitmap Index Scan** is a middle ground — it scans the index to build a bitmap of matching pages, then fetches those pages from the table in one pass. Used when:
- Multiple rows match (more than a few, less than a full scan)
- Combining multiple indexes (`BitmapAnd`, `BitmapOr`)

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 5 AND status = 'pending';
```
```
QUERY PLAN
──────────────────────────────────────────────────────────────
Bitmap Heap Scan on orders  (cost=4.58..16.73 rows=1 width=52)
  Recheck Cond: (customer_id = 5)
  Filter: (status = 'pending'::text)
  ->  Bitmap Index Scan on idx_orders_customer_id
        (cost=0.00..4.58 rows=3 width=0)
        Index Cond: (customer_id = 5)
```

### Reading EXPLAIN Output

```sql
EXPLAIN
SELECT c.name, o.total
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE c.country = 'US'
ORDER BY o.total DESC
LIMIT 5;
```
```
QUERY PLAN
──────────────────────────────────────────────────────────────────────────────
Limit  (cost=58.32..58.33 rows=5 width=36)
  ->  Sort  (cost=58.32..58.60 rows=112 width=36)
        Sort Key: o.total DESC
        ->  Hash Join  (cost=14.50..54.70 rows=112 width=36)
              Hash Cond: (o.customer_id = c.id)
              ->  Seq Scan on orders  (cost=0.00..30.40 rows=2040 width=16)
              ->  Hash  (cost=13.00..13.00 rows=120 width=24)
                    ->  Seq Scan on customers  (cost=0.00..13.00 rows=120 width=24)
                          Filter: (country = 'US'::text)
```

**Reading the plan tree — bottom up:**
1. `Seq Scan on customers` — scan customers, filter by `country = 'US'`
2. `Hash` — build a hash table from the filtered customers
3. `Seq Scan on orders` — scan all orders
4. `Hash Join` — probe the hash table to match `o.customer_id = c.id`
5. `Sort` — sort results by `o.total DESC`
6. `Limit` — return first 5 rows

**Plan node types:**

| Node | Meaning |
|---|---|
| `Seq Scan` | Full table scan |
| `Index Scan` | B-tree lookup + heap fetch |
| `Index Only Scan` | B-tree lookup, no heap fetch |
| `Bitmap Heap Scan` | Batch heap fetch from bitmap |
| `Hash Join` | Build hash table, probe with other table |
| `Nested Loop` | For each row in outer, scan inner |
| `Merge Join` | Merge two pre-sorted inputs |
| `Sort` | Sort rows (may use disk if large) |
| `Limit` | Stop after N rows |
| `Aggregate` | GROUP BY, COUNT, SUM etc. |
| `Hash Aggregate` | Aggregate using a hash table |

> **Edge Case — Cost numbers are estimates, not milliseconds:** The `cost=X..Y` values are in planner-internal units (roughly: sequential page reads = 1.0, random page reads = 4.0, CPU per row = 0.01). They are useful for comparing plans, not for predicting actual time.

> **Edge Case — EXPLAIN without ANALYZE may show a plan that's not what actually runs:** `EXPLAIN` shows the plan chosen by the planner based on statistics. If the statistics are stale (table changed a lot without `ANALYZE`), the plan shown may differ from what a real execution would choose.

---

## 10. EXPLAIN ANALYZE

`EXPLAIN ANALYZE` **actually executes the query** and returns both the estimated plan and the **real measured execution statistics**.

```sql
-- WARNING: EXPLAIN ANALYZE actually runs the query
-- For INSERT/UPDATE/DELETE, wrap in a transaction and rollback:
BEGIN;
EXPLAIN ANALYZE DELETE FROM orders WHERE status = 'cancelled';
ROLLBACK;
```

### Basic EXPLAIN ANALYZE

```sql
EXPLAIN ANALYZE
SELECT c.name, COUNT(o.id) AS order_count
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name
ORDER BY order_count DESC;
```
```
QUERY PLAN
──────────────────────────────────────────────────────────────────────────────────────
Sort  (cost=78.41..78.66 rows=100 width=40)
      (actual time=1.823..1.826 rows=5 loops=1)
  Sort Key: (count(o.id)) DESC
  Sort Method: quicksort  Memory: 25kB
  ->  HashAggregate  (cost=74.00..75.00 rows=100 width=40)
                     (actual time=1.804..1.808 rows=5 loops=1)
        Group Key: c.id, c.name
        ->  Hash Left Join  (cost=17.50..71.50 rows=500 width=36)
                             (actual time=0.314..1.775 rows=12 loops=1)
              Hash Cond: (o.customer_id = c.id)
              ->  Seq Scan on orders  (cost=0.00..30.40 rows=2040 width=8)
                                      (actual time=0.018..0.040 rows=7 loops=1)
              ->  Hash  (cost=15.00..15.00 rows=200 width=36)
                         (actual time=0.276..0.277 rows=5 loops=1)
                    ->  Seq Scan on customers  (cost=0.00..15.00 rows=200 width=36)
                                               (actual time=0.010..0.013 rows=5 loops=1)
Planning Time: 0.312 ms
Execution Time: 1.921 ms
```

### Actual vs Estimated Rows

The most important thing to look for — big differences between `rows=estimate` and `actual rows=real`:

```
(cost=0.00..30.40 rows=2040 ...)    ← planner estimated 2040 rows
(actual time=0.018..0.040 rows=7 ...)  ← actually returned 7 rows
```

A large mismatch means the planner's statistics are stale. Fix with:

```sql
ANALYZE orders;          -- update statistics for one table
ANALYZE;                 -- update statistics for all tables
VACUUM ANALYZE orders;   -- reclaim dead rows + update statistics
```

### Key Metrics to Read

```
(actual time=0.018..1.921 rows=7 loops=3)
              ↑       ↑      ↑       ↑
         startup  total  actual   how many
           time    time   rows   times node
                          out     ran
```

| Metric | What It Means | What to Look For |
|---|---|---|
| `actual time=X..Y` | Start time .. total time (ms) for this node | High time nodes are bottlenecks |
| `rows=N` | Actual rows output by this node | Compare to estimated `rows=` |
| `loops=N` | How many times this node ran | High loops on inner side of Nested Loop = problem |
| `Planning Time` | Time spent choosing the plan | Usually < 5ms; high = complex query |
| `Execution Time` | Total wall-clock time | The number you care about most |

```sql
-- Common red flags in EXPLAIN ANALYZE output:
-- 1. "Seq Scan" on a large table → missing index
-- 2. "rows=10000" estimated but "rows=3" actual → stale stats, run ANALYZE
-- 3. "loops=50000" on inner Nested Loop → JOIN on unindexed column
-- 4. "Sort Method: external merge  Disk: 42MB" → sort spilling to disk, increase work_mem
-- 5. High node time with low row count → filter not pushed down efficiently
```

### EXPLAIN Options

```sql
-- EXPLAIN with output format options

-- Default: text (human readable)
EXPLAIN SELECT * FROM orders;

-- JSON format (good for tooling/parsing)
EXPLAIN (FORMAT JSON) SELECT * FROM orders;

-- ANALYZE: actually run the query and show real timings
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 5;

-- BUFFERS: show cache hit/miss statistics (use with ANALYZE)
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE customer_id = 5;

-- VERBOSE: show output column list for each node
EXPLAIN (ANALYZE, VERBOSE) SELECT * FROM orders;

-- All options together — most detailed output
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT)
SELECT c.name, SUM(o.total)
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.id;
```

**Understanding BUFFERS output:**

```
Seq Scan on orders (actual time=0.01..1.23 rows=1000 loops=1)
  Buffers: shared hit=45 read=12
                    ↑         ↑
            pages from      pages read
            memory cache    from disk
```

- `shared hit` — data was in PostgreSQL's shared buffer cache (fast)
- `read` — data had to be read from disk (slow)
- Goal: maximize cache hits, minimize disk reads

### Full Diagnostic Workflow

```sql
-- Step 1: identify a slow query (from pg_stat_statements or application logs)
-- Step 2: run EXPLAIN ANALYZE to see the plan
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.name, o.total, o.status
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE c.country = 'US' AND o.status = 'pending'
ORDER BY o.total DESC;

-- Step 3: look for:
--   - Seq Scan on large tables
--   - Large estimate vs actual row count mismatch
--   - High loops count on nested loop inner side
--   - Disk reads instead of buffer hits

-- Step 4: act
--   Seq Scan → add index
--   Row count mismatch → ANALYZE table
--   Slow sort → CREATE INDEX with matching ORDER BY, or increase work_mem
--   Disk reads → increase shared_buffers or add more RAM

-- Step 5: re-run EXPLAIN ANALYZE and compare
```

---

## Quick Reference

```sql
-- Create indexes
CREATE INDEX idx_name ON table (col);
CREATE INDEX idx_name ON table (col DESC);
CREATE INDEX idx_name ON table (col1, col2);           -- composite
CREATE INDEX idx_name ON table (LOWER(col));           -- functional
CREATE INDEX idx_name ON table (col) WHERE cond;       -- partial
CREATE UNIQUE INDEX idx_name ON table (col);           -- unique
CREATE INDEX idx_name ON table USING GIN (jsonb_col);  -- GIN
CREATE INDEX idx_name ON table USING BRIN (ts_col);    -- BRIN
CREATE INDEX CONCURRENTLY idx_name ON table (col);     -- non-blocking

-- Covering index (INCLUDE, PG 11+)
CREATE INDEX idx_name ON table (search_col) INCLUDE (col1, col2);

-- Drop indexes
DROP INDEX IF EXISTS idx_name;
DROP INDEX CONCURRENTLY IF EXISTS idx_name;            -- non-blocking
REINDEX INDEX idx_name;                                -- rebuild bloated index
REINDEX INDEX CONCURRENTLY idx_name;                   -- non-blocking rebuild (PG 12+)

-- List indexes
\di                                                    -- psql: all indexes
\di+ tablename                                         -- indexes for one table
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'orders';

-- Find unused indexes
SELECT indexname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND NOT indisprimary
ORDER BY pg_relation_size(indexrelid) DESC;

-- EXPLAIN
EXPLAIN SELECT ...;                                    -- show plan (no execution)
EXPLAIN ANALYZE SELECT ...;                            -- run + show real timings
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;                 -- + cache hit/miss stats
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ...;    -- JSON output for tools

-- Key plan nodes to recognize
-- Seq Scan     → full table scan (no index used)
-- Index Scan   → index lookup + heap fetch
-- Index Only Scan → index lookup, no heap (all columns in index)
-- Bitmap Heap Scan → batch heap fetch (medium selectivity)
-- Hash Join    → fast join when one side fits in memory
-- Nested Loop  → good for small outer + indexed inner

-- When to add an index
-- ✓ FK columns (not auto-indexed)
-- ✓ Frequent WHERE / JOIN / ORDER BY columns
-- ✓ Large tables (> ~10,000 rows)
-- ✓ High-cardinality columns (many distinct values)

-- When NOT to add an index
-- ✗ Small tables
-- ✗ Low-cardinality columns (boolean, status with few values)
-- ✗ Columns rarely used in queries
-- ✗ Write-heavy tables that can tolerate slower reads
```
