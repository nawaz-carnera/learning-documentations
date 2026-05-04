# PostgreSQL Overview — Concept Picker & Cheatsheet

A reverse lookup: given a **requirement / question**, find the **concept** to apply, the **module** to refer back to, and a **query template**.

---

## Table of Contents

- [Module Map (What's Where)](#module-map-whats-where)
- [Part 1: Requirement → Concept Picker](#part-1-requirement--concept-picker)
  - [A. Setup & Structure](#a-setup--structure)
  - [B. Inserting / Modifying Data](#b-inserting--modifying-data)
  - [C. Reading Data — Filters & Single Table](#c-reading-data--filters--single-table)
  - [D. Aggregations & Summaries](#d-aggregations--summaries)
  - [E. Combining Tables (JOINs)](#e-combining-tables-joins)
  - [F. Subqueries & CTEs](#f-subqueries--ctes)
  - [G. Ranking & Row-by-Row Comparisons](#g-ranking--row-by-row-comparisons)
  - [H. Reusable Queries (Views)](#h-reusable-queries-views)
  - [I. Performance (Indexes)](#i-performance-indexes)
  - [J. Safe Multi-Step Operations (Transactions)](#j-safe-multi-step-operations-transactions)
  - [K. Strings, Dates, JSON, Casting](#k-strings-dates-json-casting)
  - [L. Users & Permissions](#l-users--permissions)
- [Part 2: Decision Tree](#part-2-decision-tree)
- [Part 3: Cheatsheets](#part-3-cheatsheets)
  - [psql Meta-Commands](#psql-meta-commands)
  - [Query Execution Order](#query-execution-order)
  - [JOIN Types At-a-Glance](#join-types-at-a-glance)
  - [Aggregate Functions](#aggregate-functions)
  - [Window Functions](#window-functions)
  - [String Functions](#string-functions)
  - [Date/Time Functions](#datetime-functions)
  - [JSON / JSONB Operators](#json--jsonb-operators)
  - [Constraints](#constraints)
  - [Index Types](#index-types)
  - [Transaction Isolation Levels](#transaction-isolation-levels)
  - [Permissions](#permissions)
  - [NULL Handling](#null-handling)

---

## Module Map (What's Where)

| Module | Topic | Use When |
|---|---|---|
| [1](module1.md) | PostgreSQL Fundamentals | Installing, connecting, psql basics |
| [2](module2.md) | Databases | `CREATE DATABASE`, ownership, cluster vs DB vs schema |
| [3](module3.md) | Schemas | Organizing tables, `search_path`, `schema.table` |
| [4](module4.md) | Tables | `CREATE TABLE`, data types, `ALTER TABLE` |
| [5](module5.md) | Constraints | PK, FK, UNIQUE, CHECK, NOT NULL, DEFAULT |
| [6](module6.md) | DML | INSERT, UPDATE, DELETE, UPSERT, RETURNING |
| [7](module7.md) | SELECT / Filter / Sort | WHERE, ORDER BY, LIMIT, NULL, LIKE |
| [8](module8.md) | Aggregates | COUNT, SUM, AVG, GROUP BY, HAVING |
| [9](module9.md) | JOINs | INNER / LEFT / RIGHT / FULL / CROSS / SELF |
| [10](module10.md) | Subqueries & CTEs | `IN (SELECT...)`, `EXISTS`, `WITH`, recursive |
| [11](module11.md) | Views & Materialized Views | Reusable queries, cached aggregates |
| [12](module12.md) | Indexes & Performance | Speed up queries, `EXPLAIN` |
| [13](module13.md) | Window Functions | RANK, ROW_NUMBER, LAG/LEAD, running totals |
| [14](module14.md) | Transactions | BEGIN/COMMIT/ROLLBACK, ACID, isolation |
| [15](module15.md) | Built-in Functions | Strings, dates, JSON, casting |
| [16](module16.md) | Roles & Permissions | GRANT, REVOKE, role membership |

---

## Part 1: Requirement → Concept Picker

> **How to read this section:** Each row gives you a real-world question, the concept(s) you need, the module, and a query skeleton.

### A. Setup & Structure

| Requirement / Question | Concept | Module | Skeleton |
|---|---|---|---|
| "Set up a database for the shop application." | `CREATE DATABASE` | [2](module2.md) | `CREATE DATABASE shop OWNER alice;` |
| "Group HR tables separately from public ones." | Schemas | [3](module3.md) | `CREATE SCHEMA hr; CREATE TABLE hr.employees (...);` |
| "Store products with id, name, price." | `CREATE TABLE` + types | [4](module4.md) | `CREATE TABLE products (id SERIAL PRIMARY KEY, name TEXT NOT NULL, price NUMERIC(10,2));` |
| "Add a `created_at` column to an existing table." | `ALTER TABLE ADD COLUMN` | [4](module4.md) | `ALTER TABLE products ADD COLUMN created_at TIMESTAMPTZ DEFAULT NOW();` |
| "Each email must be unique." | `UNIQUE` constraint | [5](module5.md) | `email TEXT UNIQUE` |
| "Price must be positive." | `CHECK` constraint | [5](module5.md) | `CHECK (price > 0)` |
| "Order must reference an existing customer; delete orders if customer is deleted." | `FOREIGN KEY ... ON DELETE CASCADE` | [5](module5.md) | `customer_id INT REFERENCES customers(id) ON DELETE CASCADE` |
| "Auto-generate IDs." | `SERIAL` / `IDENTITY` / `UUID` | [4](module4.md) | `id SERIAL PRIMARY KEY` or `id UUID DEFAULT gen_random_uuid()` |

### B. Inserting / Modifying Data

| Requirement / Question | Concept | Module | Skeleton |
|---|---|---|---|
| "Add a new product." | `INSERT INTO ... VALUES` | [6](module6.md) | `INSERT INTO products (name, price) VALUES ('Widget', 9.99);` |
| "Add 100 rows in one shot." | Multi-row INSERT | [6](module6.md) | `INSERT INTO products (name, price) VALUES ('A',1),('B',2),...;` |
| "Copy active users into archive table." | `INSERT INTO ... SELECT` | [6](module6.md) | `INSERT INTO archive (id, name) SELECT id, name FROM users WHERE active=false;` |
| "Bump prices of all electronics by 10%." | `UPDATE` + WHERE | [6](module6.md) | `UPDATE products SET price = price * 1.1 WHERE category='electronics';` |
| "Delete cancelled orders." | `DELETE` + WHERE | [6](module6.md) | `DELETE FROM orders WHERE status='cancelled';` |
| "Insert if new, update if exists (upsert)." | `INSERT ... ON CONFLICT DO UPDATE` | [6](module6.md) | `INSERT INTO products(id,price) VALUES(1,9.99) ON CONFLICT (id) DO UPDATE SET price = EXCLUDED.price;` |
| "Return the auto-generated ID after insert." | `RETURNING` | [6](module6.md) | `INSERT INTO products(name) VALUES('X') RETURNING id;` |
| "Empty an entire table fast." | `TRUNCATE` | [6](module6.md) | `TRUNCATE TABLE logs;` |

### C. Reading Data — Filters & Single Table

| Requirement / Question | Concept | Module | Skeleton |
|---|---|---|---|
| "List all products." | `SELECT *` | [7](module7.md) | `SELECT * FROM products;` |
| "Show product name and price only." | Column projection | [7](module7.md) | `SELECT name, price FROM products;` |
| "Show products costing more than $50." | `WHERE` | [7](module7.md) | `SELECT * FROM products WHERE price > 50;` |
| "Find users with gmail addresses." | `LIKE` / `ILIKE` | [7](module7.md) | `SELECT * FROM users WHERE email ILIKE '%@gmail.com';` |
| "Products in categories A, B, or C." | `IN` | [7](module7.md) | `WHERE category IN ('A','B','C')` |
| "Orders between Jan and March." | `BETWEEN` | [7](module7.md) | `WHERE order_date BETWEEN '2024-01-01' AND '2024-03-31'` |
| "Users without a phone number." | `IS NULL` | [7](module7.md) | `WHERE phone IS NULL` |
| "Use 'N/A' when phone is missing." | `COALESCE` | [7](module7.md) | `SELECT COALESCE(phone, 'N/A') FROM users;` |
| "Sort by price descending, then name." | `ORDER BY` | [7](module7.md) | `ORDER BY price DESC, name ASC` |
| "Show the top 10 most expensive products." | `ORDER BY ... LIMIT` | [7](module7.md) | `ORDER BY price DESC LIMIT 10` |
| "Page 3 of results, 20 per page." | `LIMIT ... OFFSET` | [7](module7.md) | `LIMIT 20 OFFSET 40` |
| "Unique categories." | `DISTINCT` | [7](module7.md) | `SELECT DISTINCT category FROM products;` |

### D. Aggregations & Summaries

| Requirement / Question | Concept | Module | Skeleton |
|---|---|---|---|
| "How many users do we have?" | `COUNT(*)` | [8](module8.md) | `SELECT COUNT(*) FROM users;` |
| "Total revenue?" | `SUM` | [8](module8.md) | `SELECT SUM(amount) FROM orders;` |
| "Average order value?" | `AVG` | [8](module8.md) | `SELECT AVG(amount) FROM orders;` |
| "Highest / lowest price?" | `MAX` / `MIN` | [8](module8.md) | `SELECT MAX(price), MIN(price) FROM products;` |
| "Revenue per category." | `GROUP BY` | [8](module8.md) | `SELECT category, SUM(amount) FROM orders GROUP BY category;` |
| "Categories with revenue > $10k." | `HAVING` (filter on aggregate) | [8](module8.md) | `GROUP BY category HAVING SUM(amount) > 10000` |
| "Comma-separated list of product names per category." | `STRING_AGG` | [8](module8.md) | `SELECT category, STRING_AGG(name, ', ') FROM products GROUP BY category;` |
| "Array of order IDs per customer." | `ARRAY_AGG` | [8](module8.md) | `SELECT customer_id, ARRAY_AGG(order_id) FROM orders GROUP BY customer_id;` |

> ⚠️ **WHERE vs HAVING:** `WHERE` filters rows **before** grouping, `HAVING` filters groups **after** aggregation.

### E. Combining Tables (JOINs)

| Requirement / Question | Concept | Module | Skeleton |
|---|---|---|---|
| **"Top 5 best-selling products by revenue."** | `JOIN` + `GROUP BY` + `ORDER BY` + `LIMIT` | [8](module8.md) + [9](module9.md) | `SELECT p.name, SUM(oi.qty * oi.price) AS rev FROM products p JOIN order_items oi ON oi.product_id=p.id GROUP BY p.name ORDER BY rev DESC LIMIT 5;` |
| "Orders with their customer's name." | `INNER JOIN` (only matched rows) | [9](module9.md) | `SELECT o.id, c.name FROM orders o JOIN customers c ON c.id=o.customer_id;` |
| "All customers, even those without orders." | `LEFT JOIN` (keep left side) | [9](module9.md) | `SELECT c.name, o.id FROM customers c LEFT JOIN orders o ON o.customer_id=c.id;` |
| "Customers who never placed an order." | `LEFT JOIN ... WHERE right IS NULL` | [9](module9.md) | `... LEFT JOIN orders o ON ... WHERE o.id IS NULL` |
| "Every product paired with every category (combinations)." | `CROSS JOIN` | [9](module9.md) | `SELECT * FROM sizes CROSS JOIN colors;` |
| "Employees and their managers (same table)." | `SELF JOIN` | [9](module9.md) | `SELECT e.name, m.name AS manager FROM employees e LEFT JOIN employees m ON m.id=e.manager_id;` |
| "Customer + order + product info." | Multi-table JOIN (3+) | [9](module9.md) | `FROM customers c JOIN orders o ON ... JOIN order_items oi ON ... JOIN products p ON ...` |

> ⚠️ Always JOIN on indexed columns (typically the FK). See [Module 12](module12.md).

### F. Subqueries & CTEs

| Requirement / Question | Concept | Module | Skeleton |
|---|---|---|---|
| "Products priced above the average." | Scalar subquery in WHERE | [10](module10.md) | `WHERE price > (SELECT AVG(price) FROM products)` |
| "Customers who placed at least one order." | `EXISTS` | [10](module10.md) | `WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id=c.id)` |
| "Customers who never placed an order." | `NOT EXISTS` | [10](module10.md) | `WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id=c.id)` |
| "Departments with above-average salary spend." | Subquery in `FROM` (derived table) | [10](module10.md) | `SELECT * FROM (SELECT dept, AVG(sal) a FROM emp GROUP BY dept) x WHERE x.a > 5000;` |
| "Step-by-step transformation, readable query." | CTE (`WITH`) | [10](module10.md) | `WITH active_users AS (SELECT * FROM users WHERE active) SELECT * FROM active_users WHERE ...;` |
| "Manager hierarchy / org chart / category tree." | **Recursive CTE** | [10](module10.md) | `WITH RECURSIVE tree AS (SELECT id,parent FROM cats WHERE parent IS NULL UNION ALL SELECT c.id,c.parent FROM cats c JOIN tree t ON c.parent=t.id) SELECT * FROM tree;` |
| "Generate dates from Jan 1 to Jan 31." | `generate_series` (recursive use case) | [10](module10.md) | `SELECT generate_series('2024-01-01'::date, '2024-01-31', '1 day');` |

> 💡 **CTE vs Subquery:** Use CTE when the query is referenced multiple times or is multi-step; subqueries are fine for one-off filtering.

### G. Ranking & Row-by-Row Comparisons

| Requirement / Question | Concept | Module | Skeleton |
|---|---|---|---|
| "Number rows 1, 2, 3 ... within each group." | `ROW_NUMBER() OVER (PARTITION BY ...)` | [13](module13.md) | `ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC)` |
| "Rank products by revenue (ties share rank)." | `RANK()` / `DENSE_RANK()` | [13](module13.md) | `RANK() OVER (ORDER BY revenue DESC)` |
| "Top 3 employees per department." | Window + filter via CTE | [13](module13.md) | `WITH r AS (SELECT *, ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) rn FROM emp) SELECT * FROM r WHERE rn <= 3;` |
| "Compare each row to the previous one (e.g. month-over-month)." | `LAG()` | [13](module13.md) | `LAG(revenue) OVER (ORDER BY month)` |
| "Compare to the next row." | `LEAD()` | [13](module13.md) | `LEAD(revenue) OVER (ORDER BY month)` |
| "Running total of sales by date." | `SUM() OVER (ORDER BY ...)` | [13](module13.md) | `SUM(amount) OVER (ORDER BY date)` |
| "7-day moving average." | Window with frame | [13](module13.md) | `AVG(amount) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)` |
| "Each row + its group's total (without losing rows)." | Window aggregate | [13](module13.md) | `SUM(amount) OVER (PARTITION BY category)` |

> 💡 **GROUP BY vs Window:** `GROUP BY` collapses rows into one per group. Window functions **keep all rows** and add a calculated column.

### H. Reusable Queries (Views)

| Requirement / Question | Concept | Module | Skeleton |
|---|---|---|---|
| "Save this complex query so I can reuse it." | `CREATE VIEW` | [11](module11.md) | `CREATE VIEW active_orders AS SELECT * FROM orders WHERE status='active';` |
| "Pre-compute an expensive dashboard query, refreshed nightly." | **Materialized View** | [11](module11.md) | `CREATE MATERIALIZED VIEW daily_revenue AS SELECT date, SUM(amount) FROM orders GROUP BY date;` |
| "Refresh the cached data." | `REFRESH MATERIALIZED VIEW` | [11](module11.md) | `REFRESH MATERIALIZED VIEW CONCURRENTLY daily_revenue;` |

### I. Performance (Indexes)

| Requirement / Question | Concept | Module | Skeleton |
|---|---|---|---|
| "Query on `email` is slow." | B-tree index | [12](module12.md) | `CREATE INDEX idx_users_email ON users(email);` |
| "Why is this query slow?" | `EXPLAIN ANALYZE` | [12](module12.md) | `EXPLAIN ANALYZE SELECT ...;` |
| "Speed up `WHERE status='active' AND created_at > X`." | Composite index (column order matters!) | [12](module12.md) | `CREATE INDEX ON orders(status, created_at);` |
| "Index only the active rows." | Partial index | [12](module12.md) | `CREATE INDEX ON orders(customer_id) WHERE status='active';` |
| "Avoid slow query without locking the table." | `CREATE INDEX CONCURRENTLY` | [12](module12.md) | `CREATE INDEX CONCURRENTLY ...;` |

> ⚠️ Indexes **slow down** writes. Don't index every column — only those frequently filtered/joined/sorted.

### J. Safe Multi-Step Operations (Transactions)

| Requirement / Question | Concept | Module | Skeleton |
|---|---|---|---|
| "Transfer money: subtract from A, add to B — both or neither." | `BEGIN ... COMMIT` | [14](module14.md) | `BEGIN; UPDATE acc SET bal=bal-100 WHERE id=1; UPDATE acc SET bal=bal+100 WHERE id=2; COMMIT;` |
| "Undo if something goes wrong." | `ROLLBACK` | [14](module14.md) | `BEGIN; ...; ROLLBACK;` |
| "Roll back part of a transaction, not all." | `SAVEPOINT` | [14](module14.md) | `SAVEPOINT s1; ...; ROLLBACK TO s1;` |
| "Run a report without seeing in-flight changes." | Isolation level `REPEATABLE READ` | [14](module14.md) | `BEGIN ISOLATION LEVEL REPEATABLE READ;` |
| "Prevent any concurrency anomalies." | `SERIALIZABLE` | [14](module14.md) | `BEGIN ISOLATION LEVEL SERIALIZABLE;` |

### K. Strings, Dates, JSON, Casting

| Requirement / Question | Concept | Module | Skeleton |
|---|---|---|---|
| "Combine first and last name." | `CONCAT` / `\|\|` | [15](module15.md) | `SELECT first_name \|\| ' ' \|\| last_name FROM users;` |
| "Extract domain from email." | `SPLIT_PART` | [15](module15.md) | `SPLIT_PART(email, '@', 2)` |
| "Replace text." | `REPLACE` / `REGEXP_REPLACE` | [15](module15.md) | `REPLACE(phone, '-', '')` |
| "Convert text to uppercase." | `UPPER` / `LOWER` | [15](module15.md) | `UPPER(name)` |
| "Get current timestamp." | `NOW()` / `CURRENT_DATE` | [15](module15.md) | `SELECT NOW();` |
| "Group orders by month." | `DATE_TRUNC` | [15](module15.md) | `GROUP BY DATE_TRUNC('month', order_date)` |
| "Get year/month/day from a date." | `EXTRACT` | [15](module15.md) | `EXTRACT(YEAR FROM order_date)` |
| "Order from 7 days ago." | `INTERVAL` arithmetic | [15](module15.md) | `WHERE order_date >= NOW() - INTERVAL '7 days'` |
| "How old is a user?" | `AGE` | [15](module15.md) | `AGE(birthday)` |
| "Convert a string to integer." | `::` cast / `CAST` | [15](module15.md) | `'42'::INT` or `CAST('42' AS INT)` |
| "Read a field from a JSON column." | `->` / `->>` operators | [15](module15.md) | `SELECT data->>'name' FROM events;` |
| "Filter on a JSON field." | `@>` / `->>` in WHERE | [15](module15.md) | `WHERE data @> '{"status":"active"}'` or `WHERE data->>'status'='active'` |

### L. Users & Permissions

| Requirement / Question | Concept | Module | Skeleton |
|---|---|---|---|
| "Create a read-only analyst." | `CREATE ROLE` + `GRANT SELECT` | [16](module16.md) | `CREATE ROLE analyst LOGIN PASSWORD 'x'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO analyst;` |
| "Group multiple users under one permission set." | Group role + membership | [16](module16.md) | `CREATE ROLE analysts; GRANT analysts TO alice, bob;` |
| "Allow user to read and write a specific table." | `GRANT` privileges | [16](module16.md) | `GRANT SELECT, INSERT, UPDATE ON orders TO alice;` |
| "Take away a permission." | `REVOKE` | [16](module16.md) | `REVOKE INSERT ON orders FROM alice;` |
| "User can't access a schema." | `GRANT USAGE ON SCHEMA` | [16](module16.md) | `GRANT USAGE ON SCHEMA hr TO alice;` |

---

## Part 2: Decision Tree

When you read a question, walk this tree:

```
Need to read data?
├─ One table only?
│  ├─ With filters?         → WHERE                    [Mod 7]
│  ├─ Need totals/counts?   → GROUP BY + aggregates    [Mod 8]
│  ├─ Need ranking/running? → Window functions         [Mod 13]
│  └─ Just rows?            → SELECT ... ORDER BY      [Mod 7]
│
├─ Multiple tables?
│  ├─ Match required?       → INNER JOIN               [Mod 9]
│  ├─ Keep all of one side? → LEFT/RIGHT JOIN          [Mod 9]
│  └─ Find missing match?   → LEFT JOIN ... IS NULL    [Mod 9]
│                              or NOT EXISTS           [Mod 10]
│
├─ Complex / multi-step?
│  ├─ Reusable building blocks? → CTE (WITH)           [Mod 10]
│  ├─ Hierarchy/recursion?      → Recursive CTE        [Mod 10]
│  └─ Save permanently?         → VIEW / MATVIEW       [Mod 11]
│
Need to write/modify data?
├─ Single row / rows         → INSERT / UPDATE / DELETE  [Mod 6]
├─ Insert OR update          → ON CONFLICT (UPSERT)      [Mod 6]
├─ Multi-step, all-or-none   → BEGIN ... COMMIT          [Mod 14]
│
Performance issue?
├─ Slow SELECT               → EXPLAIN + add index       [Mod 12]
└─ Aggregate dashboard slow  → Materialized View         [Mod 11]
```

---

## Part 3: Cheatsheets

### psql Meta-Commands

| Command | Description |
|---|---|
| `\l` | List databases |
| `\c dbname` | Connect to database |
| `\dt` | List tables in current schema |
| `\dt schema.*` | List tables in a schema |
| `\d table` | Describe a table |
| `\dn` | List schemas |
| `\du` | List roles/users |
| `\df` | List functions |
| `\dv` | List views |
| `\di` | List indexes |
| `\x` | Toggle expanded display |
| `\q` | Quit |

### Query Execution Order

SQL is **written** in this order:

```
SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY → LIMIT
```

But **executed** in this order:

```
1. FROM       (gather rows from tables)
2. WHERE      (filter rows)
3. GROUP BY   (group rows)
4. HAVING     (filter groups)
5. SELECT     (pick columns / compute)
6. ORDER BY   (sort)
7. LIMIT      (cut off)
```

> 💡 This is why column aliases in `SELECT` can't be used in `WHERE` — `WHERE` runs first.

### JOIN Types At-a-Glance

```
A INNER JOIN B  → only rows matched in BOTH
A LEFT JOIN B   → all of A, matched-or-NULL from B
A RIGHT JOIN B  → all of B, matched-or-NULL from A
A FULL JOIN B   → all of A and B, NULLs where no match
A CROSS JOIN B  → every row of A × every row of B
A SELF JOIN A   → join table to itself (alias required)
```

| Goal | Use |
|---|---|
| Only matched rows | `INNER JOIN` |
| All from left + matched right | `LEFT JOIN` |
| Find rows in A missing from B | `LEFT JOIN B WHERE B.id IS NULL` |
| Combinations | `CROSS JOIN` |

### Aggregate Functions

| Function | Returns |
|---|---|
| `COUNT(*)` | Number of rows (incl. NULLs) |
| `COUNT(col)` | Non-NULL values in col |
| `COUNT(DISTINCT col)` | Distinct non-NULL values |
| `SUM(col)` | Total |
| `AVG(col)` | Mean |
| `MIN(col)` / `MAX(col)` | Smallest / largest |
| `STRING_AGG(col, ', ')` | Concatenate as string |
| `ARRAY_AGG(col)` | Concatenate as array |

### Window Functions

| Function | Use |
|---|---|
| `ROW_NUMBER()` | Unique sequence per partition (no ties) |
| `RANK()` | Rank with gaps on ties (1, 2, 2, 4) |
| `DENSE_RANK()` | Rank without gaps (1, 2, 2, 3) |
| `LAG(col, n)` | Value from n rows before |
| `LEAD(col, n)` | Value from n rows after |
| `FIRST_VALUE(col)` | First value in window |
| `LAST_VALUE(col)` | Last value in window (mind the frame!) |
| `SUM/AVG/COUNT(...) OVER (...)` | Running / windowed aggregate |

```sql
-- Anatomy of OVER
func() OVER (
  PARTITION BY group_col   -- optional: split into groups
  ORDER BY sort_col        -- optional: order within group
  ROWS BETWEEN ... AND ... -- optional: frame for running calcs
)
```

### String Functions

| Function | Example | Result |
|---|---|---|
| `\|\|` / `CONCAT` | `'hi' \|\| ' there'` | `'hi there'` |
| `LENGTH(s)` | `LENGTH('abc')` | `3` |
| `UPPER` / `LOWER` | `UPPER('abc')` | `'ABC'` |
| `SUBSTRING(s FROM 1 FOR 3)` | `'hello' → 'hel'` | |
| `REPLACE(s, old, new)` | `REPLACE('abc','b','X')` | `'aXc'` |
| `SPLIT_PART(s, delim, n)` | `SPLIT_PART('a-b-c','-',2)` | `'b'` |
| `TRIM(s)` | `TRIM('  hi  ')` | `'hi'` |
| `POSITION(sub IN s)` | `POSITION('b' IN 'abc')` | `2` |
| `REGEXP_REPLACE(s, pat, rep)` | regex replace | |

### Date/Time Functions

| Function | Example |
|---|---|
| `NOW()` / `CURRENT_TIMESTAMP` | Current timestamp w/ TZ |
| `CURRENT_DATE` | Today's date |
| `EXTRACT(YEAR FROM d)` | Year as integer |
| `DATE_TRUNC('month', d)` | First of month |
| `AGE(d)` | Interval since `d` |
| `d + INTERVAL '7 days'` | Date arithmetic |
| `d::DATE` | Cast timestamp to date |

Common groupings:

```sql
-- By month
SELECT DATE_TRUNC('month', order_date) AS month, SUM(amount)
FROM orders GROUP BY month ORDER BY month;

-- Last 30 days
WHERE order_date >= NOW() - INTERVAL '30 days'
```

### JSON / JSONB Operators

| Operator | Returns | Example |
|---|---|---|
| `->` | JSON value | `data->'name'` → `"Alice"` (json) |
| `->>` | Text value | `data->>'name'` → `Alice` (text) |
| `#>` | JSON at path | `data#>'{a,b}'` |
| `#>>` | Text at path | `data#>>'{a,b}'` |
| `@>` | Contains | `data @> '{"active":true}'` |
| `?` | Key exists | `data ? 'email'` |

> 💡 Use **JSONB** (binary, indexable) over **JSON** for almost all use cases.

### Constraints

| Constraint | Purpose |
|---|---|
| `PRIMARY KEY` | Unique + NOT NULL row identifier |
| `UNIQUE` | No duplicates (NULLs allowed) |
| `NOT NULL` | Forbids NULL |
| `CHECK (expr)` | Custom rule (e.g. `price > 0`) |
| `DEFAULT val` | Auto-fill on INSERT |
| `FOREIGN KEY` | Link to another table |

`ON DELETE` behaviors: `CASCADE`, `SET NULL`, `SET DEFAULT`, `RESTRICT`, `NO ACTION`.

### Index Types

| Type | Best For |
|---|---|
| **B-tree** (default) | Equality, range, sorting |
| **Hash** | Equality only (rarely better than B-tree) |
| **GIN** | Full-text, JSONB, arrays |
| **GiST** | Geometric, full-text, range |
| **BRIN** | Very large tables with naturally ordered data |

```sql
-- Common patterns
CREATE INDEX ON orders(customer_id);              -- FK
CREATE INDEX ON orders(status, created_at);       -- composite
CREATE UNIQUE INDEX ON users(email);              -- unique
CREATE INDEX ON orders(customer_id) WHERE active; -- partial
CREATE INDEX CONCURRENTLY ...                     -- no table lock
```

### Transaction Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| READ UNCOMMITTED* | Possible | Possible | Possible |
| **READ COMMITTED** (default) | No | Possible | Possible |
| REPEATABLE READ | No | No | Possible |
| SERIALIZABLE | No | No | No |

*PostgreSQL treats READ UNCOMMITTED as READ COMMITTED.

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
-- ... statements
COMMIT;  -- or ROLLBACK
```

### Permissions

| Privilege | On |
|---|---|
| `SELECT` | Read rows |
| `INSERT` | Add rows |
| `UPDATE` | Modify rows |
| `DELETE` | Remove rows |
| `TRUNCATE` | Empty table |
| `REFERENCES` | Create FK |
| `USAGE` | Use schema/sequence |
| `CREATE` | Create objects |
| `ALL PRIVILEGES` | Everything |

```sql
GRANT SELECT, INSERT ON orders TO alice;
REVOKE INSERT ON orders FROM alice;
GRANT USAGE ON SCHEMA public TO alice;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO analyst;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT ON TABLES TO analyst;  -- future tables too
```

### NULL Handling

| Behavior | Note |
|---|---|
| `NULL = NULL` | Returns `NULL` (not true!) |
| `NULL = anything` | Returns `NULL` |
| Use `IS NULL` / `IS NOT NULL` | Correct way to check |
| Aggregates skip NULLs | Except `COUNT(*)` |
| `COALESCE(a, b, c)` | First non-NULL |
| `NULLIF(a, b)` | NULL if `a = b`, else `a` |

```sql
-- Wrong:  WHERE phone = NULL          ← never matches
-- Right:  WHERE phone IS NULL

-- Default value when missing:
SELECT COALESCE(phone, email, 'no contact') FROM users;
```

---

## Final Tips

1. **Start with the question** — what do you want as output? Single row? Aggregate? Per-group ranking?
2. **Identify the tables** involved → that tells you JOIN vs single-table.
3. **Look for keywords**:
   - "per", "by", "for each" → `GROUP BY` or `PARTITION BY`
   - "top N", "rank" → `ORDER BY ... LIMIT` or window function
   - "running", "cumulative", "previous", "next" → window function
   - "without", "missing", "never" → `LEFT JOIN ... IS NULL` or `NOT EXISTS`
   - "if exists update else insert" → `ON CONFLICT`
   - "all or nothing" → transaction
4. **Filter rows with `WHERE`, filter groups with `HAVING`.**
5. **When a query is slow, run `EXPLAIN ANALYZE` first** — don't guess.
6. **Test destructive queries with `SELECT` first**, then turn into `DELETE` / `UPDATE`.
