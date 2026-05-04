# Module 9: JOINs — Combining Data Across Tables

---

## Table of Contents

- [1. What a JOIN Is and Why](#1-what-a-join-is-and-why)
- [2. INNER JOIN](#2-inner-join)
- [3. LEFT JOIN](#3-left-join)
- [4. RIGHT JOIN](#4-right-join)
- [5. FULL OUTER JOIN](#5-full-outer-join)
- [6. CROSS JOIN](#6-cross-join)
- [7. SELF JOIN](#7-self-join)
- [8. Multi-Table JOINs (3+ Tables)](#8-multi-table-joins-3-tables)
- [9. Common Pitfalls](#9-common-pitfalls)
  - [Cartesian Explosion](#cartesian-explosion)
  - [NULL on Outer JOINs](#null-on-outer-joins)
  - [Duplicate Rows from One-to-Many JOINs](#duplicate-rows-from-one-to-many-joins)
  - [JOIN on Non-Indexed Columns](#join-on-non-indexed-columns)
- [Quick Reference](#quick-reference)

---

## 1. What a JOIN Is and Why

A **JOIN** combines rows from two or more tables based on a related column. Relational databases intentionally split data into separate tables to avoid redundancy — JOINs are how you reassemble that data at query time.

### Setup (tables used throughout this module)

```sql
CREATE TABLE customers (
  id         SERIAL PRIMARY KEY,
  name       TEXT NOT NULL,
  email      TEXT UNIQUE NOT NULL,
  country    TEXT NOT NULL DEFAULT 'US'
);

CREATE TABLE orders (
  id          SERIAL PRIMARY KEY,
  customer_id INTEGER REFERENCES customers(id) ON DELETE SET NULL,
  status      TEXT NOT NULL DEFAULT 'pending',
  total       NUMERIC(10,2) NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE order_items (
  id         SERIAL PRIMARY KEY,
  order_id   INTEGER NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id INTEGER NOT NULL,
  product_name TEXT NOT NULL,
  quantity   INTEGER NOT NULL,
  unit_price NUMERIC(10,2) NOT NULL
);

CREATE TABLE products (
  id       SERIAL PRIMARY KEY,
  name     TEXT NOT NULL,
  category TEXT NOT NULL,
  price    NUMERIC(10,2) NOT NULL,
  stock    INTEGER NOT NULL DEFAULT 0
);

-- Customers: some have orders, some don't
INSERT INTO customers (name, email, country) VALUES
  ('Alice',   'alice@example.com',   'US'),
  ('Bob',     'bob@example.com',     'UK'),
  ('Charlie', 'charlie@example.com', 'US'),
  ('Dana',    'dana@example.com',    'CA'),
  ('Eve',     'eve@example.com',     'US');   -- no orders

-- Orders: some have a customer, one is orphaned (customer deleted)
INSERT INTO orders (customer_id, status, total) VALUES
  (1, 'delivered', 250.00),
  (1, 'delivered', 180.00),
  (2, 'pending',    95.00),
  (3, 'cancelled',  60.00),
  (NULL, 'pending', 120.00);   -- orphaned order (customer deleted)

-- Products
INSERT INTO products (name, category, price, stock) VALUES
  ('Laptop',   'Electronics', 999.99, 10),
  ('Mouse',    'Electronics',  29.99, 50),
  ('Desk',     'Furniture',   349.99,  5),
  ('Chair',    'Furniture',   199.99,  8),
  ('Notebook', 'Stationery',    4.99, 200);
```

### Why Not Just Use One Big Table?

```
Without normalization — everything in one table:
order_id | customer_name | customer_email       | product_name | quantity | price
---------|---------------|----------------------|--------------|----------|------
1        | Alice         | alice@example.com    | Laptop       | 1        | 999.99
1        | Alice         | alice@example.com    | Mouse        | 2        |  29.99
2        | Alice         | alice@example.com    | Desk         | 1        | 349.99

Problems:
- alice@example.com is stored 3 times — update one, others go stale (update anomaly)
- Deleting all orders deletes Alice's contact info too (delete anomaly)
- Can't store Alice without an order (insert anomaly)

With normalization + JOINs: each fact stored once, reassembled on demand.
```

### JOIN Anatomy

```sql
SELECT t1.col, t2.col
FROM   t1
JOIN   t2 ON t1.key = t2.foreign_key;
--           └── join condition
```

- `FROM t1` — the **left** table
- `JOIN t2` — the **right** table
- `ON` — the condition that links them (usually FK = PK)

---

## 2. INNER JOIN

Returns **only rows that have a match in both tables**. Rows with no match on either side are excluded.

```
Left  ──┐
         ├── matched rows only
Right ──┘
```

```sql
-- Syntax: INNER JOIN and JOIN are identical
SELECT ...
FROM   left_table
[INNER] JOIN right_table ON left_table.col = right_table.col;
```

### Examples

```sql
-- All orders with their customer names (unmatched rows excluded)
SELECT
  o.id         AS order_id,
  c.name       AS customer_name,
  c.country,
  o.status,
  o.total
FROM orders o
INNER JOIN customers c ON o.customer_id = c.id;
```
```
 order_id | customer_name | country | status    | total
----------+---------------+---------+-----------+--------
        1 | Alice         | US      | delivered | 250.00
        2 | Alice         | US      | delivered | 180.00
        3 | Bob           | UK      | pending   |  95.00
        4 | Charlie       | US      | cancelled |  60.00
```
> Order 5 (orphaned, `customer_id = NULL`) is excluded — no matching customer.
> Eve (id=5) is excluded — she has no orders.

```sql
-- Total spent per customer (only customers with at least one order)
SELECT
  c.name,
  COUNT(o.id)       AS order_count,
  SUM(o.total)      AS total_spent
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name
ORDER BY total_spent DESC;
```
```
 name    | order_count | total_spent
---------+-------------+------------
 Alice   |           2 |     430.00
 Bob     |           1 |      95.00
 Charlie |           1 |      60.00
```

```sql
-- Order items with product details
SELECT
  oi.order_id,
  oi.product_name,
  oi.quantity,
  oi.unit_price,
  oi.quantity * oi.unit_price AS line_total
FROM order_items oi
INNER JOIN orders o ON oi.order_id = o.id
WHERE o.status = 'delivered';
```

> **Edge Case — INNER JOIN with NULL FK returns no row:**
> ```sql
> -- Order 5 has customer_id = NULL
> -- NULL = anything is NULL (not TRUE) → row excluded from INNER JOIN
> SELECT * FROM orders o
> INNER JOIN customers c ON o.customer_id = c.id
> WHERE o.id = 5;
> -- Returns 0 rows — NULL FK never matches
> ```

> **Edge Case — Multiple matches multiply rows:**
> ```sql
> -- Alice has 2 orders → appears twice in the result
> -- This is correct and expected — one row per match
> SELECT c.name, o.id AS order_id
> FROM customers c
> INNER JOIN orders o ON c.id = o.customer_id;
> -- Alice | 1
> -- Alice | 2
> -- Bob   | 3
> -- Charlie | 4
> ```

---

## 3. LEFT JOIN

Returns **all rows from the left table**, plus matched rows from the right table. Where there is no match, right-table columns are filled with `NULL`.

```
Left  ──── all rows
Right ──── matched rows only (NULL for non-matches)
```

```sql
SELECT ...
FROM   left_table
LEFT [OUTER] JOIN right_table ON left_table.col = right_table.col;
```

### Examples

```sql
-- All customers, with their orders (Eve appears with NULLs — no orders)
SELECT
  c.name,
  c.country,
  o.id      AS order_id,
  o.status,
  o.total
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
ORDER BY c.name;
```
```
 name    | country | order_id | status    | total
---------+---------+----------+-----------+--------
 Alice   | US      |        1 | delivered | 250.00
 Alice   | US      |        2 | delivered | 180.00
 Bob     | UK      |        3 | pending   |  95.00
 Charlie | US      |        4 | cancelled |  60.00
 Dana    | CA      |   NULL   | NULL      | NULL     ← no orders
 Eve     | US      |   NULL   | NULL      | NULL     ← no orders
```

### Find rows with NO match (anti-join pattern)

```sql
-- Customers who have never placed an order
SELECT c.id, c.name, c.email
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.id IS NULL;
```
```
 id | name | email
----+------+------------------
  4 | Dana | dana@example.com
  5 | Eve  | eve@example.com
```

```sql
-- Products that have never been ordered
SELECT p.id, p.name, p.category
FROM products p
LEFT JOIN order_items oi ON p.id = oi.product_id
WHERE oi.id IS NULL;
```

### LEFT JOIN with aggregation (include zero-count groups)

```sql
-- All customers with order count (including those with 0 orders)
SELECT
  c.name,
  COUNT(o.id)       AS order_count,   -- COUNT(col) = 0 when all NULLs
  COALESCE(SUM(o.total), 0) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name
ORDER BY total_spent DESC;
```
```
 name    | order_count | total_spent
---------+-------------+------------
 Alice   |           2 |     430.00
 Bob     |           1 |      95.00
 Charlie |           1 |      60.00
 Dana    |           0 |       0.00
 Eve     |           0 |       0.00
```

> **When to use LEFT JOIN:**
> - You want all rows from the left table regardless of matches
> - Finding "unmatched" rows (anti-join: `WHERE right.id IS NULL`)
> - Including zero-count groups in aggregations
> - Reporting where missing data should show as NULL or 0

> **Edge Case — Filtering on the right table in WHERE turns LEFT JOIN into INNER JOIN:**
> ```sql
> -- LOOKS like LEFT JOIN but behaves like INNER JOIN:
> SELECT c.name, o.status
> FROM customers c
> LEFT JOIN orders o ON c.id = o.customer_id
> WHERE o.status = 'pending';      -- ← filters out NULL rows (Dana, Eve excluded)
>
> -- Fix: move the filter into the ON clause to preserve left-table rows
> SELECT c.name, o.status
> FROM customers c
> LEFT JOIN orders o ON c.id = o.customer_id AND o.status = 'pending';
> -- Dana and Eve still appear, but their o.status is NULL (no pending orders)
> ```

---

## 4. RIGHT JOIN

Returns **all rows from the right table**, plus matched rows from the left table. Where there is no match, left-table columns are `NULL`.

```
Left  ──── matched rows only (NULL for non-matches)
Right ──── all rows
```

```sql
SELECT ...
FROM   left_table
RIGHT [OUTER] JOIN right_table ON left_table.col = right_table.col;
```

```sql
-- All orders, with customer info where available
-- (orphaned order with NULL customer_id will appear with NULL customer columns)
SELECT
  o.id        AS order_id,
  o.total,
  o.status,
  c.name      AS customer_name
FROM customers c
RIGHT JOIN orders o ON c.id = o.customer_id
ORDER BY o.id;
```
```
 order_id | total  | status    | customer_name
----------+--------+-----------+---------------
        1 | 250.00 | delivered | Alice
        2 | 180.00 | delivered | Alice
        3 |  95.00 | pending   | Bob
        4 |  60.00 | cancelled | Charlie
        5 | 120.00 | pending   | NULL          ← orphaned order
```

> **RIGHT JOIN is rarely needed in practice.**
> Any `RIGHT JOIN` can be rewritten as a `LEFT JOIN` by swapping the table order:
> ```sql
> -- These two queries return the same result:
> SELECT * FROM customers c RIGHT JOIN orders o ON c.id = o.customer_id;
> SELECT * FROM orders o   LEFT  JOIN customers c ON o.customer_id = c.id;
> ```
> Most developers stick to `LEFT JOIN` for consistency — it reads naturally as "start with this table, optionally attach that one."

---

## 5. FULL OUTER JOIN

Returns **all rows from both tables**. Matched rows are combined; unmatched rows from either side get `NULL` for the other table's columns.

```
Left  ──── all rows (NULL for unmatched right columns)
Right ──── all rows (NULL for unmatched left columns)
```

```sql
SELECT ...
FROM   left_table
FULL [OUTER] JOIN right_table ON left_table.col = right_table.col;
```

```sql
-- All customers AND all orders — unmatched rows on both sides visible
SELECT
  c.name        AS customer_name,
  c.country,
  o.id          AS order_id,
  o.status,
  o.total
FROM customers c
FULL OUTER JOIN orders o ON c.id = o.customer_id
ORDER BY c.name NULLS LAST, o.id;
```
```
 customer_name | country | order_id | status    | total
---------------+---------+----------+-----------+--------
 Alice         | US      |        1 | delivered | 250.00
 Alice         | US      |        2 | delivered | 180.00
 Bob           | UK      |        3 | pending   |  95.00
 Charlie       | US      |        4 | cancelled |  60.00
 Dana          | CA      |   NULL   | NULL      | NULL     ← customer with no order
 Eve           | US      |   NULL   | NULL      | NULL     ← customer with no order
 NULL          | NULL    |        5 | pending   | 120.00   ← orphaned order
```

### Find rows unmatched on EITHER side

```sql
-- All unmatched rows from both tables (full anti-join)
SELECT
  c.name    AS customer_name,
  o.id      AS order_id
FROM customers c
FULL OUTER JOIN orders o ON c.id = o.customer_id
WHERE c.id IS NULL      -- unmatched orders (no customer)
   OR o.id IS NULL;     -- unmatched customers (no orders)
```
```
 customer_name | order_id
---------------+----------
 Dana          | NULL
 Eve           | NULL
 NULL          |        5
```

### Common use cases

```sql
-- Data reconciliation: compare two sources for the same entity
-- Find IDs present in one table but missing in the other
SELECT
  a.id AS source_a_id,
  b.id AS source_b_id
FROM source_a a
FULL OUTER JOIN source_b b ON a.external_id = b.external_id
WHERE a.id IS NULL OR b.id IS NULL;

-- Merge two lists and coalesce matching columns
SELECT
  COALESCE(a.product_id, b.product_id) AS product_id,
  a.stock_warehouse                    AS warehouse_stock,
  b.stock_store                        AS store_stock
FROM warehouse_inventory a
FULL OUTER JOIN store_inventory b ON a.product_id = b.product_id;
```

---

## 6. CROSS JOIN

Returns the **Cartesian product** of two tables — every row from the left paired with every row from the right. No `ON` clause.

```
Left rows × Right rows = result rows
3 rows × 4 rows = 12 rows
```

```sql
SELECT ...
FROM left_table
CROSS JOIN right_table;

-- Implicit syntax (comma-separated in FROM — same result)
SELECT * FROM left_table, right_table;
```

```sql
-- Every customer paired with every product (all possible combinations)
SELECT
  c.name        AS customer,
  p.name        AS product,
  p.price
FROM customers c
CROSS JOIN products p
ORDER BY c.name, p.name;
```
```
 customer | product   | price
----------+-----------+--------
 Alice    | Chair     | 199.99
 Alice    | Desk      | 349.99
 Alice    | Laptop    | 999.99
 Alice    | Mouse     |  29.99
 Alice    | Notebook  |   4.99
 Bob      | Chair     | 199.99
 ... (5 customers × 5 products = 25 rows)
```

### Practical Uses

```sql
-- Generate a date series for a report (all months × all departments)
SELECT
  d.month,
  dept.department
FROM generate_series('2024-01-01'::DATE, '2024-12-01', '1 month') AS d(month)
CROSS JOIN (SELECT DISTINCT department FROM employees) AS dept
ORDER BY dept.department, d.month;
-- Ensures every department appears for every month, even with no data

-- Generate a multiplication table
SELECT
  a.n AS multiplier,
  b.n AS multiplicand,
  a.n * b.n AS product
FROM generate_series(1, 5) AS a(n)
CROSS JOIN generate_series(1, 5) AS b(n)
ORDER BY a.n, b.n;

-- All possible size + color combinations for a product
SELECT s.size, c.color
FROM (VALUES ('S'), ('M'), ('L'), ('XL')) AS s(size)
CROSS JOIN (VALUES ('Red'), ('Blue'), ('Green')) AS c(color);
```

> **Edge Case — Accidental CROSS JOIN (Cartesian explosion):** The most dangerous JOIN mistake. See [Section 9](#9-common-pitfalls) for details.

---

## 7. SELF JOIN

A **self join** joins a table to **itself**. The table appears twice in the query, so you must use aliases to distinguish the two instances.

```sql
SELECT ...
FROM table AS a
JOIN table AS b ON a.col = b.other_col;
```

### Hierarchy / Org Chart (Manager → Employee)

```sql
CREATE TABLE employees (
  id         SERIAL PRIMARY KEY,
  name       TEXT NOT NULL,
  role       TEXT NOT NULL,
  manager_id INTEGER REFERENCES employees(id)
);

INSERT INTO employees (name, role, manager_id) VALUES
  ('Carol',   'CEO',      NULL),   -- id=1
  ('Dave',    'VP Eng',   1),      -- id=2, reports to Carol
  ('Eve',     'VP Sales', 1),      -- id=3, reports to Carol
  ('Frank',   'Engineer', 2),      -- id=4, reports to Dave
  ('Grace',   'Engineer', 2),      -- id=5, reports to Dave
  ('Hank',    'Sales Rep',3);      -- id=6, reports to Eve

-- Self JOIN to get employee + their manager's name
SELECT
  e.name        AS employee,
  e.role        AS employee_role,
  m.name        AS manager,
  m.role        AS manager_role
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id
ORDER BY m.name NULLS FIRST;
```
```
 employee | employee_role | manager | manager_role
----------+---------------+---------+--------------
 Carol    | CEO           | NULL    | NULL          ← top of hierarchy
 Dave     | VP Eng        | Carol   | CEO
 Eve      | VP Sales      | Carol   | CEO
 Frank    | Engineer      | Dave    | VP Eng
 Grace    | Engineer      | Dave    | VP Eng
 Hank     | Sales Rep     | Eve     | VP Sales
```

### Find Pairs with Something in Common

```sql
-- All pairs of customers from the same country (self-join on country)
SELECT
  a.name AS customer_1,
  b.name AS customer_2,
  a.country
FROM customers a
JOIN customers b ON a.country = b.country AND a.id < b.id
-- a.id < b.id prevents (Alice,Charlie) and (Charlie,Alice) both appearing
-- and prevents (Alice,Alice) self-pairing
ORDER BY a.country, a.name;
```
```
 customer_1 | customer_2 | country
------------+------------+---------
 Alice      | Charlie    | US
 Alice      | Eve        | US
 Charlie    | Eve        | US
```

```sql
-- Find employees earning more than their manager
SELECT
  e.name     AS employee,
  e.salary   AS emp_salary,
  m.name     AS manager,
  m.salary   AS mgr_salary
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

> **Edge Case — Self JOIN vs recursive CTE:** A self join works for one level of hierarchy (employee → manager). For arbitrary depth (full org tree), use a recursive CTE:
> ```sql
> WITH RECURSIVE org_tree AS (
>   SELECT id, name, manager_id, 0 AS depth, name::TEXT AS path
>   FROM employees WHERE manager_id IS NULL     -- root(s)
>   UNION ALL
>   SELECT e.id, e.name, e.manager_id, ot.depth + 1,
>          ot.path || ' > ' || e.name
>   FROM employees e
>   JOIN org_tree ot ON e.manager_id = ot.id
>   WHERE ot.depth < 10                         -- safety limit
> )
> SELECT depth, path FROM org_tree ORDER BY path;
> ```
> ```
>  depth | path
> -------+-------------------------------
>      0 | Carol
>      1 | Carol > Dave
>      2 | Carol > Dave > Frank
>      2 | Carol > Dave > Grace
>      1 | Carol > Eve
>      2 | Carol > Eve > Hank
> ```

---

## 8. Multi-Table JOINs (3+ Tables)

Join as many tables as needed — PostgreSQL executes them left to right, building up the result set incrementally.

```sql
SELECT ...
FROM   table_a a
JOIN   table_b b ON a.id = b.a_id
JOIN   table_c c ON b.id = c.b_id
LEFT JOIN table_d d ON c.id = d.c_id;
```

### Three-Table JOIN

```sql
-- Customers → Orders → Order Items (full detail)
SELECT
  c.name                                        AS customer,
  o.id                                          AS order_id,
  o.status,
  oi.product_name,
  oi.quantity,
  oi.unit_price,
  oi.quantity * oi.unit_price                   AS line_total
FROM customers c
JOIN orders     o  ON c.id       = o.customer_id
JOIN order_items oi ON o.id      = oi.order_id
ORDER BY c.name, o.id, oi.product_name;
```

```sql
-- Revenue by product category
SELECT
  p.category,
  SUM(oi.quantity * oi.unit_price)              AS revenue,
  COUNT(DISTINCT o.id)                          AS order_count,
  SUM(oi.quantity)                              AS units_sold
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
JOIN products    p  ON oi.product_id = p.id
WHERE o.status = 'delivered'
GROUP BY p.category
ORDER BY revenue DESC;
```

### Four-Table JOIN with a Mix of LEFT and INNER

```sql
-- Full report: all customers, their orders, items, and product details
-- Use LEFT JOIN to include customers with no orders
SELECT
  c.name                                        AS customer,
  c.country,
  o.id                                          AS order_id,
  o.status,
  oi.product_name,
  p.category,
  oi.quantity,
  oi.unit_price
FROM customers c
LEFT JOIN orders      o  ON c.id       = o.customer_id
LEFT JOIN order_items oi ON o.id       = oi.order_id
LEFT JOIN products    p  ON oi.product_id = p.id
ORDER BY c.name, o.id;
```

### Using CTEs to Break Up Complex Multi-Joins

```sql
-- Complex query split into readable steps
WITH delivered_orders AS (
  SELECT id, customer_id, total
  FROM orders
  WHERE status = 'delivered'
),
order_totals AS (
  SELECT customer_id, COUNT(*) AS orders, SUM(total) AS revenue
  FROM delivered_orders
  GROUP BY customer_id
)
SELECT
  c.name,
  c.country,
  COALESCE(ot.orders,  0) AS delivered_orders,
  COALESCE(ot.revenue, 0) AS total_revenue
FROM customers c
LEFT JOIN order_totals ot ON c.id = ot.customer_id
ORDER BY total_revenue DESC;
```

> **Edge Case — JOIN order affects which rows survive when mixing LEFT and INNER JOIN:**
> ```sql
> -- WRONG: the INNER JOIN on order_items cancels the LEFT JOIN on orders
> SELECT c.name, o.id, oi.product_name
> FROM customers c
> LEFT JOIN orders o       ON c.id = o.customer_id
> INNER JOIN order_items oi ON o.id = oi.order_id;
> -- Customers with no orders have o.id = NULL
> -- INNER JOIN drops rows where o.id IS NULL → Eve/Dana disappear
>
> -- FIX: use LEFT JOIN consistently down the chain
> FROM customers c
> LEFT JOIN orders      o  ON c.id = o.customer_id
> LEFT JOIN order_items oi ON o.id = oi.order_id;
> ```

> **Edge Case — Alias every table in multi-joins:** Without aliases, column references become ambiguous when multiple tables share column names like `id`, `name`, `created_at`. Always alias and always qualify:
> ```sql
> -- Ambiguous — which "id" and "name"?
> SELECT id, name, status FROM customers JOIN orders ON customers.id = orders.customer_id;
>
> -- Clear and correct
> SELECT c.id, c.name, o.status FROM customers c JOIN orders o ON c.id = o.customer_id;
> ```

---

## 9. Common Pitfalls

### Cartesian Explosion

A Cartesian product occurs when a JOIN condition is **missing or wrong**, causing every row in one table to match every row in another.

```sql
-- MISSING ON clause — accidental CROSS JOIN (PostgreSQL will warn but execute it)
SELECT * FROM customers, orders;
-- 5 customers × 5 orders = 25 rows  ← probably not what you wanted

-- WRONG condition — joining on the wrong columns
SELECT *
FROM orders o
JOIN customers c ON o.id = c.id;   -- joins order.id to customer.id (wrong!)
-- order 1 matches customer 1, order 2 matches customer 2, etc.
-- Not a full cartesian product, but semantically wrong

-- CORRECT
SELECT *
FROM orders o
JOIN customers c ON o.customer_id = c.id;
```

```sql
-- Real-world Cartesian explosion in aggregation
-- Suppose orders has 1M rows and order_items has 10M rows
-- An accidental cross join produces 10 trillion rows — crashes the server

-- Always verify JOIN conditions with small sample data first:
SELECT COUNT(*) FROM table_a;            -- e.g. 1000
SELECT COUNT(*) FROM table_b;            -- e.g. 500
SELECT COUNT(*) FROM table_a JOIN table_b ON a.id = b.a_id;
-- Expected: between 0 and max(1000,500) for 1:1 or 1:many
-- If result is 500,000 → something is wrong (1000 × 500 = 500,000)
```

> **Rule of thumb:** After writing a multi-table query, check the row count with `SELECT COUNT(*)` before `SELECT *`. If it's unexpectedly large, you have a JOIN problem.

---

### NULL on Outer JOINs

The most common logic error with `LEFT JOIN` is filtering on right-table columns in `WHERE`, which silently converts it to an `INNER JOIN`.

```sql
-- TRAP: WHERE on right-table column eliminates NULL rows
SELECT c.name, o.status
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.status = 'delivered';
-- Dana and Eve (no orders) are excluded — o.status IS NULL, not 'delivered'
-- This behaves identically to INNER JOIN

-- FIX A: Move the filter to the ON clause
SELECT c.name, o.status
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id AND o.status = 'delivered';
-- Dana and Eve still appear, but o.status = NULL

-- FIX B: Also allow NULLs in WHERE
SELECT c.name, o.status
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.status = 'delivered' OR o.status IS NULL;
```

```sql
-- TRAP: counting right-table rows with COUNT(*) includes the NULL row
SELECT c.name, COUNT(*) AS order_count
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.name;
-- Dana and Eve each show order_count = 1 (the NULL row is counted!)

-- FIX: use COUNT(o.id) — counts only non-NULL values
SELECT c.name, COUNT(o.id) AS order_count
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.name;
-- Dana and Eve correctly show order_count = 0
```

```sql
-- TRAP: SUM on NULL rows inflates total
SELECT c.name, SUM(o.total) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.name;
-- SUM(NULL) = NULL for Dana/Eve — ok for SUM, but use COALESCE for display

SELECT c.name, COALESCE(SUM(o.total), 0) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.name;
```

---

### Duplicate Rows from One-to-Many JOINs

When the right table has **multiple rows per left-table row**, the left row is repeated — this is correct behavior but surprises many beginners.

```sql
-- Alice has 2 orders → appears twice
SELECT c.name, o.id AS order_id
FROM customers c
JOIN orders o ON c.id = o.customer_id;
-- Alice | 1
-- Alice | 2   ← duplicate customer row, not a bug

-- Problem: aggregating the parent table after joining can double-count
SELECT c.name, COUNT(*) AS something
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.name;
-- Alice | 2  ← 2 matches (correct for order count)

-- Double-counting bug: summing customer-level data through a join
-- Suppose customers had a "credit_limit" column:
SELECT SUM(c.credit_limit)           -- WRONG: Alice's credit_limit counted twice
FROM customers c
JOIN orders o ON c.id = o.customer_id;

-- FIX A: aggregate before joining
WITH order_counts AS (
  SELECT customer_id, COUNT(*) AS cnt FROM orders GROUP BY customer_id
)
SELECT SUM(c.credit_limit) FROM customers c JOIN order_counts oc ON c.id = oc.customer_id;

-- FIX B: use DISTINCT in the SUM
SELECT SUM(DISTINCT c.credit_limit) FROM customers c JOIN orders o ON c.id = o.customer_id;
-- Caution: DISTINCT SUM only works if credit_limit values are unique per customer

-- FIX C (cleanest): don't join when you don't need to
SELECT SUM(credit_limit) FROM customers
WHERE id IN (SELECT DISTINCT customer_id FROM orders);
```

---

### JOIN on Non-Indexed Columns

```sql
-- Fast: joining on indexed FK column
SELECT * FROM orders o JOIN customers c ON o.customer_id = c.id;
-- o.customer_id is a FK → often has an index
-- c.id is PK → always has an index

-- Slow: joining on a non-indexed column (full table scan per row)
SELECT * FROM orders o JOIN customers c ON o.status = c.country;
-- Neither orders.status nor customers.country has an index by default

-- Fix: create indexes on columns used in JOIN conditions
CREATE INDEX idx_orders_customer_id ON orders (customer_id);
CREATE INDEX idx_orders_status      ON orders (status);
```

> **Check which JOINs are slow using EXPLAIN ANALYZE:**
> ```sql
> EXPLAIN ANALYZE
> SELECT * FROM orders o
> JOIN customers c ON o.customer_id = c.id;
> -- Look for "Seq Scan" on large tables — that's a sign you need an index
> -- "Index Scan" or "Index Only Scan" means the index is being used
> ```

---

## Quick Reference

```sql
-- INNER JOIN: only matching rows
SELECT * FROM a JOIN b ON a.id = b.a_id;
SELECT * FROM a INNER JOIN b ON a.id = b.a_id;   -- same

-- LEFT JOIN: all left rows, NULL for unmatched right
SELECT * FROM a LEFT JOIN b ON a.id = b.a_id;
SELECT * FROM a LEFT OUTER JOIN b ON a.id = b.a_id;   -- same

-- RIGHT JOIN: all right rows, NULL for unmatched left
SELECT * FROM a RIGHT JOIN b ON a.id = b.a_id;

-- FULL OUTER JOIN: all rows from both, NULL where unmatched
SELECT * FROM a FULL OUTER JOIN b ON a.id = b.a_id;

-- CROSS JOIN: every row × every row (no ON clause)
SELECT * FROM a CROSS JOIN b;
SELECT * FROM a, b;   -- implicit cross join (avoid — error-prone)

-- SELF JOIN: table joined to itself
SELECT e.name, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- Anti-join: rows in left with NO match in right
SELECT * FROM a LEFT JOIN b ON a.id = b.a_id WHERE b.id IS NULL;

-- Multi-table
SELECT * FROM a
JOIN  b ON a.id = b.a_id
JOIN  c ON b.id = c.b_id
LEFT JOIN d ON c.id = d.c_id;

-- Key rules
-- 1. INNER JOIN excludes NULLs and unmatched rows on both sides
-- 2. LEFT JOIN: use COUNT(right.id) not COUNT(*) to avoid counting NULL rows
-- 3. LEFT JOIN: filter right-table columns in ON clause, not WHERE
-- 4. Always alias tables and qualify column names in multi-joins
-- 5. Verify row count after writing a new JOIN (check for Cartesian explosion)
-- 6. RIGHT JOIN = LEFT JOIN with tables swapped — prefer LEFT JOIN for consistency
```
