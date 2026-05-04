# Module 10: Subqueries and CTEs — Composing Complex Queries

---

## Table of Contents

- [1. Subquery in WHERE](#1-subquery-in-where)
  - [IN Subquery](#in-subquery)
  - [NOT IN Subquery](#not-in-subquery)
  - [EXISTS Subquery](#exists-subquery)
  - [NOT EXISTS Subquery](#not-exists-subquery)
  - [Scalar Subquery in WHERE](#scalar-subquery-in-where)
  - [ALL and ANY with Subqueries](#all-and-any-with-subqueries)
- [2. Subquery in FROM (Derived Table)](#2-subquery-in-from-derived-table)
- [3. Subquery in SELECT (Correlated Subquery)](#3-subquery-in-select-correlated-subquery)
- [4. Common Table Expressions (WITH Clause)](#4-common-table-expressions-with-clause)
- [5. Multiple CTEs in One Query](#5-multiple-ctEs-in-one-query)
- [6. Recursive CTEs](#6-recursive-ctes)
  - [Basic Structure](#basic-structure)
  - [Hierarchy Traversal](#hierarchy-traversal)
  - [Generating Series](#generating-series)
  - [Graph Traversal](#graph-traversal)
- [7. CTE vs Subquery — When to Use Which](#7-cte-vs-subquery--when-to-use-which)
- [Quick Reference](#quick-reference)

---

## 1. Subquery in WHERE

A **subquery** is a `SELECT` statement nested inside another SQL statement. When placed in `WHERE`, it filters rows based on the result of the inner query.

### Setup (tables used throughout this module)

```sql
CREATE TABLE customers (
  id      SERIAL PRIMARY KEY,
  name    TEXT NOT NULL,
  country TEXT NOT NULL,
  tier    TEXT NOT NULL DEFAULT 'standard'
);

CREATE TABLE orders (
  id          SERIAL PRIMARY KEY,
  customer_id INTEGER REFERENCES customers(id),
  status      TEXT NOT NULL,
  total       NUMERIC(10,2) NOT NULL,
  created_at  DATE NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE order_items (
  id           SERIAL PRIMARY KEY,
  order_id     INTEGER NOT NULL REFERENCES orders(id),
  product_id   INTEGER NOT NULL,
  product_name TEXT NOT NULL,
  quantity     INTEGER NOT NULL,
  unit_price   NUMERIC(10,2) NOT NULL
);

CREATE TABLE products (
  id       SERIAL PRIMARY KEY,
  name     TEXT NOT NULL,
  category TEXT NOT NULL,
  price    NUMERIC(10,2) NOT NULL
);

INSERT INTO customers VALUES
  (1, 'Alice',   'US', 'gold'),
  (2, 'Bob',     'UK', 'standard'),
  (3, 'Charlie', 'US', 'standard'),
  (4, 'Dana',    'CA', 'gold'),
  (5, 'Eve',     'US', 'standard');

INSERT INTO orders VALUES
  (1,  1, 'delivered', 250.00, '2024-01-10'),
  (2,  1, 'delivered', 430.00, '2024-03-15'),
  (3,  2, 'pending',    95.00, '2024-05-01'),
  (4,  3, 'delivered',  60.00, '2024-02-20'),
  (5,  3, 'cancelled', 120.00, '2024-04-05'),
  (6,  4, 'delivered', 875.00, '2024-06-10'),
  (7,  5, 'pending',    40.00, '2024-07-01');

INSERT INTO products VALUES
  (1, 'Laptop',     'Electronics', 999.99),
  (2, 'Mouse',      'Electronics',  29.99),
  (3, 'Desk',       'Furniture',   349.99),
  (4, 'Chair',      'Furniture',   199.99),
  (5, 'Notebook',   'Stationery',    4.99),
  (6, 'Headphones', 'Electronics', 149.99);
```

---

### IN Subquery

Checks whether a value matches **any value** returned by the subquery.

```sql
-- Customers who have placed at least one delivered order
SELECT id, name, country
FROM customers
WHERE id IN (
  SELECT DISTINCT customer_id
  FROM orders
  WHERE status = 'delivered'
);
```
```
 id | name    | country
----+---------+---------
  1 | Alice   | US
  3 | Charlie | US
  4 | Dana    | CA
```

```sql
-- Products in categories that have at least one product over $200
SELECT name, price
FROM products
WHERE category IN (
  SELECT DISTINCT category
  FROM products
  WHERE price > 200
)
ORDER BY category, price;
```

```sql
-- Orders containing a specific product (by product_id)
SELECT id, customer_id, total
FROM orders
WHERE id IN (
  SELECT DISTINCT order_id
  FROM order_items
  WHERE product_id = 1   -- Laptop
);
```

> **Edge Case — IN vs JOIN:** Both return the same rows, but `IN` with a subquery and `JOIN` have different behaviors when duplicates exist in the subquery result. `IN` automatically deduplicates (no need for `DISTINCT`), while a `JOIN` without deduplication can multiply rows. For existence checks, `EXISTS` is often faster than `IN` on large subquery results.

---

### NOT IN Subquery

```sql
-- Customers who have NEVER placed a delivered order
SELECT id, name
FROM customers
WHERE id NOT IN (
  SELECT customer_id
  FROM orders
  WHERE status = 'delivered'
  AND customer_id IS NOT NULL   -- CRITICAL: filter NULLs
);
```

> **Edge Case — The NULL trap with NOT IN (most dangerous pitfall in SQL):**
> ```sql
> -- If the subquery returns even ONE NULL, NOT IN returns zero rows — always
> SELECT name FROM customers
> WHERE id NOT IN (SELECT customer_id FROM orders);
> -- orders has a row with customer_id = NULL
> -- NOT IN (1, 2, 3, NULL) → x!=1 AND x!=2 AND x!=3 AND x!=NULL
> -- x != NULL is always NULL → entire expression is NULL → row excluded
> -- RESULT: 0 rows, even though there are customers with no orders
>
> -- Safe fix: always filter NULLs from NOT IN subqueries
> WHERE id NOT IN (
>   SELECT customer_id FROM orders WHERE customer_id IS NOT NULL
> );
>
> -- Better fix: use NOT EXISTS instead (NULL-safe by design)
> WHERE NOT EXISTS (
>   SELECT 1 FROM orders WHERE orders.customer_id = customers.id
> );
> ```

---

### EXISTS Subquery

`EXISTS` returns `TRUE` if the subquery returns **at least one row** — it doesn't care about the actual values. The `SELECT 1` (or `SELECT *`) inside is conventional — the values returned are irrelevant.

```sql
-- Customers who have placed at least one order (any status)
SELECT id, name
FROM customers c
WHERE EXISTS (
  SELECT 1
  FROM orders o
  WHERE o.customer_id = c.id   -- correlated: references outer query's c.id
);
```

```sql
-- Orders that contain at least one Electronics product
SELECT o.id, o.total, o.status
FROM orders o
WHERE EXISTS (
  SELECT 1
  FROM order_items oi
  JOIN products p ON oi.product_id = p.id
  WHERE oi.order_id = o.id
    AND p.category = 'Electronics'
);
```

```sql
-- Customers with at least one order over $300
SELECT name, tier
FROM customers c
WHERE EXISTS (
  SELECT 1 FROM orders o
  WHERE o.customer_id = c.id
    AND o.total > 300
);
```

**EXISTS vs IN — which is faster?**

| Scenario | Prefer |
|---|---|
| Subquery is large (many rows) | `EXISTS` — stops at first match |
| Subquery is small | Either — similar performance |
| Subquery has NULLs | `EXISTS` — NULL-safe |
| Need to reference outer row | `EXISTS` — supports correlation |
| Checking membership in a list literal | `IN ('a','b','c')` — cleaner |

> **EXISTS short-circuits:** As soon as the inner query finds one matching row, it stops scanning. `IN` evaluates the full subquery first, then checks membership. On large tables, `EXISTS` is almost always faster for "does any match exist?" checks.

---

### NOT EXISTS Subquery

Find rows in the outer table that have **no matching row** in the subquery — the NULL-safe anti-join.

```sql
-- Customers who have NEVER placed any order
SELECT id, name, country
FROM customers c
WHERE NOT EXISTS (
  SELECT 1 FROM orders o
  WHERE o.customer_id = c.id
);
```
```
 id | name | country
----+------+---------
  5 | Eve  | US
```

```sql
-- Products that appear in no orders
SELECT id, name, category
FROM products p
WHERE NOT EXISTS (
  SELECT 1 FROM order_items oi
  WHERE oi.product_id = p.id
);
```

```sql
-- Customers in 'US' who have no pending orders
SELECT c.name
FROM customers c
WHERE c.country = 'US'
  AND NOT EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = c.id
      AND o.status = 'pending'
  );
```

---

### Scalar Subquery in WHERE

A **scalar subquery** returns exactly one row and one column — a single value. It can be used anywhere a value is expected.

```sql
-- Orders whose total exceeds the overall average order total
SELECT id, customer_id, total
FROM orders
WHERE total > (
  SELECT AVG(total) FROM orders   -- returns a single number
);
```
```
 id | customer_id | total
----+-------------+--------
  2 |           1 | 430.00
  6 |           4 | 875.00
```

```sql
-- Customers whose first order total exceeds the avg first-order total
SELECT c.name
FROM customers c
WHERE (
  SELECT total FROM orders
  WHERE customer_id = c.id
  ORDER BY created_at ASC
  LIMIT 1
) > (
  SELECT AVG(first_total) FROM (
    SELECT DISTINCT ON (customer_id)
      customer_id, total AS first_total
    FROM orders
    ORDER BY customer_id, created_at ASC
  ) ft
);
```

> **Edge Case — Scalar subquery returning more than one row causes an error:**
> ```sql
> SELECT * FROM customers
> WHERE id = (SELECT customer_id FROM orders);
> -- ERROR: more than one row returned by a subquery used as an expression
>
> -- Fix: use IN, EXISTS, or add LIMIT 1 / aggregate
> WHERE id = (SELECT customer_id FROM orders WHERE id = 3)        -- OK if unique
> WHERE id = (SELECT MIN(customer_id) FROM orders)                -- aggregate → 1 row
> WHERE id IN (SELECT customer_id FROM orders)                    -- IN handles multiple
> ```

> **Edge Case — Scalar subquery returning zero rows returns NULL:**
> ```sql
> SELECT name,
>   (SELECT MAX(total) FROM orders WHERE customer_id = c.id) AS max_order
> FROM customers c;
> -- For Eve (no orders), max_order = NULL (not 0)
> -- Wrap with COALESCE if 0 is preferred: COALESCE((SELECT MAX...), 0)
> ```

---

### ALL and ANY with Subqueries

```sql
-- Orders greater than ALL individual orders from customer 2
-- (i.e., greater than the MAX of customer 2's orders)
SELECT id, total
FROM orders
WHERE total > ALL (
  SELECT total FROM orders WHERE customer_id = 2
);
-- Equivalent to: WHERE total > (SELECT MAX(total) FROM orders WHERE customer_id = 2)

-- Orders greater than at least one order from customer 1
-- (i.e., greater than the MIN of customer 1's orders)
SELECT id, total
FROM orders
WHERE total > ANY (
  SELECT total FROM orders WHERE customer_id = 1
);
-- Equivalent to: WHERE total > (SELECT MIN(total) FROM orders WHERE customer_id = 1)
```

> `= ANY(subquery)` is equivalent to `IN (subquery)`.
> `<> ALL(subquery)` is equivalent to `NOT IN (subquery)` — but inherits the NULL trap.

---

## 2. Subquery in FROM (Derived Table)

A subquery in the `FROM` clause acts as a **temporary table** (called a derived table or inline view). It must be given an alias.

```sql
SELECT outer_cols
FROM (
  SELECT inner_cols FROM table WHERE condition
) AS alias_name
WHERE outer_condition;
```

### Basic Derived Table

```sql
-- Average order total per customer, then filter for high spenders
SELECT customer_id, avg_total
FROM (
  SELECT customer_id, ROUND(AVG(total), 2) AS avg_total
  FROM orders
  GROUP BY customer_id
) AS customer_avgs
WHERE avg_total > 200;
```
```
 customer_id | avg_total
-------------+-----------
           1 |    340.00
           4 |    875.00
```

### Avoid Repeating Expressions

```sql
-- Without derived table — must repeat the CASE expression in WHERE
SELECT
  id, total,
  CASE WHEN total > 400 THEN 'high'
       WHEN total > 100 THEN 'medium'
       ELSE 'low' END AS tier
FROM orders
WHERE CASE WHEN total > 400 THEN 'high'     -- repeated
           WHEN total > 100 THEN 'medium'
           ELSE 'low' END = 'high';

-- With derived table — define once, filter by alias
SELECT id, total, tier
FROM (
  SELECT id, total,
    CASE WHEN total > 400 THEN 'high'
         WHEN total > 100 THEN 'medium'
         ELSE 'low' END AS tier
  FROM orders
) AS categorized
WHERE tier = 'high';
```

### Aggregation on Top of Aggregation

SQL does not allow nested aggregate functions directly (`AVG(COUNT(*))` is illegal). Use a derived table to work around this:

```sql
-- Average number of orders per customer
-- (can't do AVG(COUNT(*)) directly — must nest)
SELECT ROUND(AVG(order_count), 2) AS avg_orders_per_customer
FROM (
  SELECT customer_id, COUNT(*) AS order_count
  FROM orders
  GROUP BY customer_id
) AS counts;
```

```sql
-- Distribution of order counts: how many customers have 1 order, 2 orders, etc.
SELECT order_count, COUNT(*) AS customer_count
FROM (
  SELECT customer_id, COUNT(*) AS order_count
  FROM orders
  GROUP BY customer_id
) AS order_counts
GROUP BY order_count
ORDER BY order_count;
```

### Paginating a Complex Query

```sql
-- Paginate a ranked result (can't use LIMIT inside a ranked subquery)
SELECT name, total_spent, rank
FROM (
  SELECT
    c.name,
    SUM(o.total) AS total_spent,
    RANK() OVER (ORDER BY SUM(o.total) DESC) AS rank
  FROM customers c
  JOIN orders o ON c.id = o.customer_id
  GROUP BY c.id, c.name
) AS ranked
WHERE rank <= 3;
```

> **Edge Case — Derived table alias is mandatory:**
> ```sql
> SELECT * FROM (SELECT id FROM orders);
> -- ERROR: subquery in FROM must have an alias
>
> -- Fix: always add AS alias
> SELECT * FROM (SELECT id FROM orders) AS sub;
> ```

> **Edge Case — Column names in the outer query must match aliases defined in the subquery:**
> ```sql
> SELECT total_spent FROM (            -- "total_spent" must be defined inside
>   SELECT SUM(total) AS total_spent   -- ← defined here
>   FROM orders GROUP BY customer_id
> ) AS t;
> ```

---

## 3. Subquery in SELECT (Correlated Subquery)

A **correlated subquery** in the `SELECT` clause runs **once per row** of the outer query, referencing the outer row's values. It must return exactly one value (scalar).

```sql
-- Each customer with their total number of orders
SELECT
  c.name,
  c.tier,
  (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.id) AS order_count,
  (SELECT COALESCE(SUM(total), 0) FROM orders o WHERE o.customer_id = c.id) AS total_spent,
  (SELECT MAX(total) FROM orders o WHERE o.customer_id = c.id) AS largest_order
FROM customers c
ORDER BY total_spent DESC;
```
```
 name    | tier     | order_count | total_spent | largest_order
---------+----------+-------------+-------------+---------------
 Alice   | gold     |           2 |      680.00 |        430.00
 Dana    | gold     |           1 |      875.00 |        875.00
 Charlie | standard |           2 |      180.00 |        120.00
 Bob     | standard |           1 |       95.00 |         95.00
 Eve     | standard |           0 |        0.00 |          NULL
```

```sql
-- Each order with the customer's total lifetime spend (for context)
SELECT
  o.id,
  o.total,
  o.status,
  (SELECT SUM(total) FROM orders o2 WHERE o2.customer_id = o.customer_id) AS customer_lifetime_value
FROM orders o
ORDER BY o.id;
```

```sql
-- Customers with their most recent order date
SELECT
  c.name,
  (SELECT MAX(created_at) FROM orders o WHERE o.customer_id = c.id) AS last_order_date
FROM customers c;
```

> **Edge Case — Performance: correlated subquery runs N times (once per outer row):**
> ```sql
> -- If customers has 100,000 rows, the subquery runs 100,000 times — very slow
> SELECT c.name,
>   (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.id) AS cnt
> FROM customers c;
>
> -- Rewrite as LEFT JOIN + GROUP BY — runs once, much faster
> SELECT c.name, COUNT(o.id) AS cnt
> FROM customers c
> LEFT JOIN orders o ON c.id = o.customer_id
> GROUP BY c.id, c.name;
> ```
> Use correlated subqueries when:
> - The query is simple and the table is small
> - The logic is hard to express as a JOIN (e.g., multiple independent aggregates)
> - Readability matters more than performance in that context

> **Edge Case — Correlated subquery returning more than one row:**
> ```sql
> -- ERROR if more than one row returned
> SELECT c.name,
>   (SELECT total FROM orders o WHERE o.customer_id = c.id)  -- multiple orders
> FROM customers c;
> -- ERROR: more than one row returned
>
> -- Fix: add aggregate or LIMIT 1
> (SELECT MAX(total) FROM orders o WHERE o.customer_id = c.id)
> (SELECT total FROM orders o WHERE o.customer_id = c.id ORDER BY created_at DESC LIMIT 1)
> ```

---

## 4. Common Table Expressions (WITH Clause)

A **CTE** (Common Table Expression) defines a named temporary result set at the top of a query using the `WITH` keyword. It can be referenced like a table in the main query.

```sql
WITH cte_name AS (
  SELECT ...
)
SELECT * FROM cte_name WHERE ...;
```

### Basic CTE

```sql
-- Customers with at least one delivered order
WITH delivered_customers AS (
  SELECT DISTINCT customer_id
  FROM orders
  WHERE status = 'delivered'
)
SELECT c.name, c.tier, c.country
FROM customers c
JOIN delivered_customers dc ON c.id = dc.customer_id;
```

### CTE Replacing a Subquery for Clarity

```sql
-- Without CTE — nested and hard to read
SELECT name, total_spent
FROM (
  SELECT c.name, SUM(o.total) AS total_spent
  FROM customers c
  JOIN orders o ON c.id = o.customer_id
  WHERE o.status = 'delivered'
  GROUP BY c.id, c.name
) AS sub
WHERE total_spent > 300
ORDER BY total_spent DESC;

-- With CTE — structured top-down, each step named
WITH delivered_totals AS (
  SELECT c.name, SUM(o.total) AS total_spent
  FROM customers c
  JOIN orders o ON c.id = o.customer_id
  WHERE o.status = 'delivered'
  GROUP BY c.id, c.name
)
SELECT name, total_spent
FROM delivered_totals
WHERE total_spent > 300
ORDER BY total_spent DESC;
```

### CTE Used Multiple Times in One Query

One of the biggest advantages of CTEs over subqueries — define once, reference many times:

```sql
WITH order_stats AS (
  SELECT
    customer_id,
    COUNT(*)        AS order_count,
    SUM(total)      AS total_spent,
    AVG(total)      AS avg_order,
    MAX(total)      AS biggest_order,
    MIN(created_at) AS first_order,
    MAX(created_at) AS last_order
  FROM orders
  WHERE status != 'cancelled'
  GROUP BY customer_id
)
SELECT
  c.name,
  c.tier,
  os.order_count,
  os.total_spent,
  ROUND(os.avg_order, 2)  AS avg_order,
  os.biggest_order,
  os.first_order,
  os.last_order,
  -- reuse the same CTE for a window function
  RANK() OVER (ORDER BY os.total_spent DESC) AS spend_rank
FROM customers c
JOIN order_stats os ON c.id = os.customer_id
ORDER BY spend_rank;
```

> **Edge Case — CTEs in PostgreSQL are optimization fences (before PG 12):**
> Before PostgreSQL 12, CTEs were always materialized (executed once and stored). This could help performance (avoiding repeated subquery execution) or hurt it (preventing the planner from pushing WHERE filters inside the CTE).
> Since **PostgreSQL 12**, the planner can inline CTEs when beneficial. You can override with:
> ```sql
> WITH cte AS MATERIALIZED (     -- force materialization (old behavior)
>   SELECT * FROM big_table
> )
> WITH cte AS NOT MATERIALIZED ( -- force inlining (let planner optimize)
>   SELECT * FROM big_table
> )
> ```

---

## 5. Multiple CTEs in One Query

Chain multiple CTEs separated by commas — each can reference the ones defined before it.

```sql
WITH
cte_one AS (...),
cte_two AS (...),       -- can reference cte_one
cte_three AS (...)      -- can reference cte_one and cte_two
SELECT ...
FROM cte_three;
```

### Full Example: Customer Spending Report

```sql
WITH
-- Step 1: raw order totals per customer, excluding cancellations
order_totals AS (
  SELECT
    customer_id,
    COUNT(*)        AS order_count,
    SUM(total)      AS total_spent,
    MAX(created_at) AS last_order_date
  FROM orders
  WHERE status != 'cancelled'
  GROUP BY customer_id
),

-- Step 2: classify customers by spend tier (references order_totals)
customer_tiers AS (
  SELECT
    customer_id,
    total_spent,
    order_count,
    last_order_date,
    CASE
      WHEN total_spent >= 500 THEN 'platinum'
      WHEN total_spent >= 200 THEN 'gold'
      WHEN total_spent >= 50  THEN 'silver'
      ELSE                         'bronze'
    END AS spend_tier
  FROM order_totals
),

-- Step 3: summary stats per spend tier (references customer_tiers)
tier_summary AS (
  SELECT
    spend_tier,
    COUNT(*)              AS customer_count,
    ROUND(AVG(total_spent), 2) AS avg_spend
  FROM customer_tiers
  GROUP BY spend_tier
)

-- Final output: join everything together
SELECT
  c.name,
  c.country,
  ct.spend_tier,
  ct.total_spent,
  ct.order_count,
  ct.last_order_date,
  ts.customer_count    AS others_in_tier,
  ts.avg_spend         AS tier_avg_spend
FROM customers c
JOIN customer_tiers ct ON c.id = ct.customer_id
JOIN tier_summary   ts ON ct.spend_tier = ts.spend_tier
ORDER BY ct.total_spent DESC;
```

### CTEs with DML (INSERT, UPDATE, DELETE)

CTEs can contain data-modifying statements too — powerful for atomic multi-step operations:

```sql
-- Archive old orders and delete them in one atomic statement
WITH archived AS (
  INSERT INTO orders_archive
  SELECT * FROM orders WHERE created_at < '2023-01-01'
  RETURNING id
)
DELETE FROM orders
WHERE id IN (SELECT id FROM archived);
```

```sql
-- Insert a new customer, then insert a welcome order for them
WITH new_customer AS (
  INSERT INTO customers (name, email, country)
  VALUES ('Frank', 'frank@example.com', 'US')
  RETURNING id
)
INSERT INTO orders (customer_id, status, total)
SELECT id, 'pending', 0.00 FROM new_customer;
```

> **Edge Case — CTEs execute once, even if referenced multiple times:**
> ```sql
> WITH expensive_calc AS (
>   SELECT customer_id, SUM(total) AS total FROM orders GROUP BY customer_id
> )
> SELECT * FROM expensive_calc WHERE total > 300
> UNION ALL
> SELECT * FROM expensive_calc WHERE total < 100;
> -- "expensive_calc" is computed ONCE and reused — not executed twice
> -- (when MATERIALIZED, which is the default for non-trivial CTEs)
> ```

---

## 6. Recursive CTEs

A **recursive CTE** references itself, allowing it to iterate — essential for hierarchies, graphs, and sequences where depth is unknown at write time.

### Basic Structure

```sql
WITH RECURSIVE cte_name AS (
  -- Anchor: the starting point (non-recursive, runs once)
  SELECT ...
  FROM base_table
  WHERE starting_condition

  UNION ALL

  -- Recursive: references cte_name itself, runs until no new rows
  SELECT ...
  FROM base_table
  JOIN cte_name ON join_condition    -- ← references itself
  WHERE stop_condition
)
SELECT * FROM cte_name;
```

- **Anchor member** — executed once, seeds the recursion
- **Recursive member** — executed repeatedly, joins new rows to the current result
- Stops when the recursive member returns **zero rows**
- Always add a **depth limit** or cycle guard to prevent infinite loops

---

### Hierarchy Traversal

```sql
CREATE TABLE departments (
  id        SERIAL PRIMARY KEY,
  name      TEXT NOT NULL,
  parent_id INTEGER REFERENCES departments(id)
);

INSERT INTO departments VALUES
  (1, 'Company',     NULL),
  (2, 'Engineering', 1),
  (3, 'Marketing',   1),
  (4, 'Backend',     2),
  (5, 'Frontend',    2),
  (6, 'SEO',         3),
  (7, 'Ads',         3),
  (8, 'Infrastructure', 4);
```

**Top-down traversal (root → leaves):**

```sql
WITH RECURSIVE dept_tree AS (
  -- Anchor: start at the root (no parent)
  SELECT
    id,
    name,
    parent_id,
    0              AS depth,
    name::TEXT     AS path
  FROM departments
  WHERE parent_id IS NULL

  UNION ALL

  -- Recursive: attach children to current level
  SELECT
    d.id,
    d.name,
    d.parent_id,
    dt.depth + 1,
    dt.path || ' > ' || d.name
  FROM departments d
  JOIN dept_tree dt ON d.parent_id = dt.id
  WHERE dt.depth < 10            -- safety: max 10 levels deep
)
SELECT
  REPEAT('  ', depth) || name AS indented_name,
  depth,
  path
FROM dept_tree
ORDER BY path;
```
```
 indented_name         | depth | path
-----------------------+-------+-------------------------------------
 Company               |     0 | Company
   Engineering         |     1 | Company > Engineering
     Backend           |     2 | Company > Engineering > Backend
       Infrastructure  |     3 | Company > Engineering > Backend > Infrastructure
     Frontend          |     2 | Company > Engineering > Frontend
   Marketing           |     1 | Company > Marketing
     Ads               |     2 | Company > Marketing > Ads
     SEO               |     2 | Company > Marketing > SEO
```

**Bottom-up traversal (leaf → root — find ancestors):**

```sql
-- Find all ancestors of "Infrastructure" (id=8)
WITH RECURSIVE ancestors AS (
  -- Anchor: start at the target node
  SELECT id, name, parent_id, 0 AS depth
  FROM departments
  WHERE id = 8

  UNION ALL

  -- Recursive: walk up to parent
  SELECT d.id, d.name, d.parent_id, a.depth + 1
  FROM departments d
  JOIN ancestors a ON d.id = a.parent_id
)
SELECT name, depth FROM ancestors ORDER BY depth;
```
```
 name            | depth
-----------------+-------
 Infrastructure  |     0
 Backend         |     1
 Engineering     |     2
 Company         |     3
```

---

### Generating Series

```sql
-- Generate numbers 1 to 10
WITH RECURSIVE nums AS (
  SELECT 1 AS n
  UNION ALL
  SELECT n + 1 FROM nums WHERE n < 10
)
SELECT n FROM nums;

-- Generate a date series (every day for 7 days)
WITH RECURSIVE dates AS (
  SELECT CURRENT_DATE AS d
  UNION ALL
  SELECT d + 1 FROM dates WHERE d < CURRENT_DATE + 6
)
SELECT d FROM dates;
```

> **Prefer `generate_series()` for simple sequences** — it's built-in and faster:
> ```sql
> SELECT * FROM generate_series(1, 10);
> SELECT * FROM generate_series(CURRENT_DATE, CURRENT_DATE + 6, '1 day'::INTERVAL);
> ```
> Use recursive CTEs for sequences that depend on data (hierarchy, graph paths).

---

### Graph Traversal

```sql
CREATE TABLE routes (
  from_city TEXT NOT NULL,
  to_city   TEXT NOT NULL,
  distance  INTEGER NOT NULL
);

INSERT INTO routes VALUES
  ('NYC', 'Boston',  215),
  ('NYC', 'Philly',   95),
  ('Boston', 'Portland', 107),
  ('Philly', 'DC',    140);

-- Find all reachable cities from NYC, with total distance
WITH RECURSIVE reachable AS (
  -- Anchor: starting city
  SELECT
    to_city                AS city,
    distance               AS total_distance,
    ARRAY['NYC', to_city]  AS visited        -- track visited nodes
  FROM routes
  WHERE from_city = 'NYC'

  UNION ALL

  -- Recursive: extend the path
  SELECT
    r.to_city,
    rc.total_distance + r.distance,
    rc.visited || r.to_city
  FROM routes r
  JOIN reachable rc ON r.from_city = rc.city
  WHERE NOT r.to_city = ANY(rc.visited)    -- cycle prevention
    AND array_length(rc.visited, 1) < 10  -- depth limit
)
SELECT city, total_distance, visited AS path
FROM reachable
ORDER BY total_distance;
```
```
   city   | total_distance |          path
----------+----------------+-------------------------
 Boston   |            215 | {NYC,Boston}
 Philly   |             95 | {NYC,Philly}
 DC       |            235 | {NYC,Philly,DC}
 Portland |            322 | {NYC,Boston,Portland}
```

> **Edge Case — Infinite loops in recursive CTEs:**
> ```sql
> -- If your data has cycles (A → B → A) and no cycle guard, the query runs forever
> -- Always add one of:
> -- 1. depth limit:        WHERE dt.depth < 20
> -- 2. cycle array guard:  WHERE NOT node_id = ANY(visited_array)
> -- 3. CYCLE clause (PostgreSQL 14+):
> WITH RECURSIVE cte AS (
>   SELECT id, parent_id, name FROM nodes WHERE parent_id IS NULL
>   UNION ALL
>   SELECT n.id, n.parent_id, n.name FROM nodes n JOIN cte ON n.parent_id = cte.id
> ) CYCLE id SET is_cycle USING cycle_path
> SELECT * FROM cte WHERE NOT is_cycle;
> ```

---

## 7. CTE vs Subquery — When to Use Which

Both CTEs and subqueries produce the same results — the choice is about **readability, reusability, and sometimes performance**.

### Decision Table

| Situation | Use |
|---|---|
| Simple one-time filter or derived table | **Subquery** — less overhead, inline |
| Logic referenced more than once in the query | **CTE** — define once, reuse |
| Query has 3+ logical steps | **CTE** — one named block per step |
| Debugging — isolate a step | **CTE** — run the CTE alone to inspect results |
| Recursive logic (hierarchy, graph, sequence) | **Recursive CTE** — only option |
| DML chaining (INSERT → DELETE) | **CTE with RETURNING** |
| Performance-critical, small input | **Subquery** — planner can optimize inline |
| Anti-join ("find rows with no match") | **NOT EXISTS subquery** — NULL-safe, fast |

### Side-by-Side Comparison

```sql
-- SUBQUERY version
SELECT name, total_spent
FROM (
  SELECT c.name, SUM(o.total) AS total_spent
  FROM customers c
  JOIN orders o ON c.id = o.customer_id
  GROUP BY c.id, c.name
) AS t
WHERE total_spent > 300;

-- CTE version (identical result)
WITH customer_spend AS (
  SELECT c.name, SUM(o.total) AS total_spent
  FROM customers c
  JOIN orders o ON c.id = o.customer_id
  GROUP BY c.id, c.name
)
SELECT name, total_spent
FROM customer_spend
WHERE total_spent > 300;
```

### Key Differences

```sql
-- Subqueries can be nested (harder to read as depth grows)
SELECT * FROM (
  SELECT * FROM (
    SELECT * FROM (
      SELECT id, total FROM orders WHERE status = 'delivered'
    ) AS a WHERE total > 100
  ) AS b WHERE id > 2
) AS c;   -- ← three levels deep, hard to follow

-- CTEs are flat (easier to read, each step named)
WITH step1 AS (SELECT id, total FROM orders WHERE status = 'delivered'),
     step2 AS (SELECT * FROM step1 WHERE total > 100),
     step3 AS (SELECT * FROM step2 WHERE id > 2)
SELECT * FROM step3;   -- reads top-to-bottom like a recipe
```

```sql
-- Subquery used twice = executed twice (unless the planner is smart)
SELECT *
FROM orders
WHERE total > (SELECT AVG(total) FROM orders)
  AND total < (SELECT AVG(total) FROM orders) * 2;
-- AVG subquery potentially executed twice

-- CTE used twice = computed once (when materialized)
WITH avg_total AS (
  SELECT AVG(total) AS avg FROM orders
)
SELECT * FROM orders
WHERE total > (SELECT avg FROM avg_total)
  AND total < (SELECT avg FROM avg_total) * 2;
```

### Rules of Thumb

```
1. Default to subqueries for simple, one-off use
2. Reach for a CTE when:
   - You need to name an intermediate result
   - The same subquery appears more than once
   - The query has more than 2–3 logical levels
   - You're writing recursive logic
3. Always use NOT EXISTS over NOT IN when the subquery might return NULLs
4. Prefer LEFT JOIN + GROUP BY over correlated SELECT subqueries for aggregations
5. For very complex reports, chain CTEs like a pipeline — one step per CTE
```

---

## Quick Reference

```sql
-- Subquery in WHERE
WHERE id IN     (SELECT id FROM t WHERE cond)
WHERE id NOT IN (SELECT id FROM t WHERE id IS NOT NULL)  -- NULL-safe NOT IN
WHERE EXISTS    (SELECT 1 FROM t WHERE t.fk = outer.id)  -- preferred
WHERE NOT EXISTS(SELECT 1 FROM t WHERE t.fk = outer.id)  -- NULL-safe anti-join
WHERE col >     (SELECT AVG(col) FROM t)                 -- scalar subquery
WHERE col > ALL (SELECT col FROM t WHERE cond)           -- greater than max
WHERE col > ANY (SELECT col FROM t WHERE cond)           -- greater than min

-- Subquery in FROM (derived table — alias required)
SELECT * FROM (SELECT col, AGG() FROM t GROUP BY col) AS alias
SELECT * FROM (SELECT ...) AS alias WHERE alias.col = 'x'

-- Subquery in SELECT (correlated — runs per row, must return 1 value)
SELECT col, (SELECT COUNT(*) FROM t2 WHERE t2.fk = t1.id) AS cnt FROM t1
SELECT col, (SELECT MAX(x)   FROM t2 WHERE t2.fk = t1.id) AS max_x FROM t1

-- Basic CTE
WITH cte AS (SELECT ...)
SELECT * FROM cte WHERE ...;

-- Multiple CTEs
WITH
  step1 AS (SELECT ...),
  step2 AS (SELECT ... FROM step1),
  step3 AS (SELECT ... FROM step1 JOIN step2 ON ...)
SELECT * FROM step3;

-- CTE with DML
WITH deleted AS (DELETE FROM t WHERE cond RETURNING *)
INSERT INTO archive SELECT * FROM deleted;

-- Forced materialization (PG 12+)
WITH cte AS MATERIALIZED     (...) ...
WITH cte AS NOT MATERIALIZED (...) ...

-- Recursive CTE
WITH RECURSIVE cte AS (
  SELECT ...               -- anchor: runs once
  UNION ALL
  SELECT ... FROM t
  JOIN cte ON cte.id = t.parent_id
  WHERE cte.depth < 10     -- depth limit (always add this)
)
SELECT * FROM cte;

-- Cycle detection (PG 14+)
WITH RECURSIVE cte AS (...)
CYCLE id SET is_cycle USING cycle_path
SELECT * FROM cte WHERE NOT is_cycle;
```
