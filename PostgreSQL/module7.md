# Module 7: Querying Data — SELECT, Filter, Sort, and Paginate

---

## Table of Contents

- [1. SELECT and Column Selection](#1-select-and-column-selection)
  - [Basic SELECT](#basic-select)
  - [Column Aliases](#column-aliases)
  - [Computed Columns](#computed-columns)
  - [SELECT Without FROM](#select-without-from)
- [2. WHERE Clause](#2-where-clause)
- [3. AND, OR, NOT](#3-and-or-not)
  - [AND](#and)
  - [OR](#or)
  - [NOT](#not)
  - [Operator Precedence](#operator-precedence)
- [4. Comparison Operators](#4-comparison-operators)
  - [Basic Comparisons (=, !=, <, >, <=, >=)](#basic-comparisons-----)
  - [IN and NOT IN](#in-and-not-in)
  - [BETWEEN and NOT BETWEEN](#between-and-not-between)
  - [LIKE and NOT LIKE](#like-and-not-like)
  - [ILIKE (Case-Insensitive LIKE)](#ilike-case-insensitive-like)
- [5. NULL Handling](#5-null-handling)
  - [IS NULL and IS NOT NULL](#is-null-and-is-not-null)
  - [COALESCE](#coalesce)
  - [NULLIF](#nullif)
  - [NULL in Expressions and Aggregates](#null-in-expressions-and-aggregates)
- [6. ORDER BY](#6-order-by)
  - [ASC and DESC](#asc-and-desc)
  - [Multiple Columns](#multiple-columns)
  - [NULL Ordering](#null-ordering)
  - [ORDER BY with Expressions](#order-by-with-expressions)
- [7. LIMIT and OFFSET](#7-limit-and-offset)
  - [LIMIT](#limit)
  - [OFFSET](#offset)
  - [Pagination Pattern](#pagination-pattern)
  - [FETCH (SQL Standard Alternative)](#fetch-sql-standard-alternative)
- [8. DISTINCT](#8-distinct)
  - [DISTINCT on a Single Column](#distinct-on-a-single-column)
  - [DISTINCT on Multiple Columns](#distinct-on-multiple-columns)
  - [DISTINCT ON (PostgreSQL-specific)](#distinct-on-postgresql-specific)
- [Quick Reference](#quick-reference)

---

## 1. SELECT and Column Selection

`SELECT` retrieves rows from a table or expression. It is the most used statement in SQL.

### Basic SELECT

```sql
-- Select all columns
SELECT * FROM users;

-- Select specific columns
SELECT id, name, email FROM users;

-- Select from a schema-qualified table
SELECT id, name FROM hr.employees;
```

> **Avoid `SELECT *` in production code.** It returns every column — including ones you don't need — wastes bandwidth, breaks if columns are added/reordered, and prevents index-only scans. Name your columns explicitly.

### Column Aliases

Rename a column in the output using `AS` (the keyword is optional but recommended for clarity):

```sql
SELECT
  id          AS user_id,
  name        AS full_name,
  email       AS contact_email,
  created_at  AS registered_on
FROM users;
```

```sql
-- AS keyword is optional but clearer
SELECT name full_name FROM users;   -- same as AS full_name

-- Alias with spaces: use double quotes
SELECT name AS "Full Name" FROM users;
```

> **Edge Case — You cannot use a column alias in the WHERE clause** of the same query. WHERE is evaluated before SELECT:
> ```sql
> -- ERROR: column "total_amount" does not exist
> SELECT total * quantity AS total_amount
> FROM order_items
> WHERE total_amount > 100;
>
> -- Fix option 1: repeat the expression
> WHERE total * quantity > 100;
>
> -- Fix option 2: wrap in a subquery or CTE
> WITH item_totals AS (
>   SELECT total * quantity AS total_amount FROM order_items
> )
> SELECT * FROM item_totals WHERE total_amount > 100;
> ```

### Computed Columns

```sql
SELECT
  name,
  salary,
  salary * 0.10          AS bonus,
  salary + salary * 0.10 AS total_compensation,
  UPPER(email)           AS email_upper,
  LENGTH(name)           AS name_length,
  NOW() - created_at     AS account_age
FROM users;
```

```sql
-- String concatenation
SELECT first_name || ' ' || last_name AS full_name FROM employees;

-- Conditional column using CASE
SELECT
  name,
  salary,
  CASE
    WHEN salary >= 100000 THEN 'Senior'
    WHEN salary >= 60000  THEN 'Mid'
    ELSE                       'Junior'
  END AS seniority_band
FROM employees;
```

### SELECT Without FROM

PostgreSQL allows `SELECT` without a table — useful for quick calculations and testing:

```sql
SELECT 1 + 1;                          -- 2
SELECT NOW();                          -- current timestamp
SELECT UPPER('hello');                 -- HELLO
SELECT 10 / 3;                         -- 3 (integer division)
SELECT 10.0 / 3;                       -- 3.3333...
SELECT gen_random_uuid();              -- a fresh UUID
SELECT 'hello' || ' ' || 'world';     -- hello world
```

---

## 2. WHERE Clause

`WHERE` filters rows — only rows where the condition evaluates to `TRUE` are returned. Rows where the condition is `FALSE` or `NULL` are excluded.

```sql
SELECT columns FROM table WHERE condition;
```

```sql
-- Filter by exact value
SELECT * FROM users WHERE role = 'admin';

-- Filter by number
SELECT * FROM orders WHERE total > 100;

-- Filter by date
SELECT * FROM orders WHERE created_at >= '2024-01-01';

-- Filter by boolean
SELECT * FROM users WHERE is_active = true;
SELECT * FROM users WHERE is_active;          -- shorthand for = true
SELECT * FROM users WHERE NOT is_active;      -- shorthand for = false
```

> **Edge Case — WHERE evaluates before SELECT aliases:** As shown above, column aliases defined in `SELECT` are not available in `WHERE`. The SQL execution order is: `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT`.

> **Edge Case — Case sensitivity in string comparisons:**
> ```sql
> SELECT * FROM users WHERE role = 'Admin';   -- no match if stored as 'admin'
> SELECT * FROM users WHERE role = 'admin';   -- matches
>
> -- Case-insensitive match:
> SELECT * FROM users WHERE LOWER(role) = 'admin';
> SELECT * FROM users WHERE role ILIKE 'admin';
> ```

---

## 3. AND, OR, NOT

Combine multiple conditions with logical operators.

### AND

All conditions must be true:

```sql
SELECT * FROM users
WHERE is_active = true
  AND role = 'admin';

SELECT * FROM orders
WHERE status = 'pending'
  AND total > 500
  AND created_at >= '2024-01-01';
```

### OR

At least one condition must be true:

```sql
SELECT * FROM users
WHERE role = 'admin'
   OR role = 'staff';

-- Cleaner with IN for multiple OR on same column:
SELECT * FROM users
WHERE role IN ('admin', 'staff');
```

### NOT

Negates a condition:

```sql
SELECT * FROM users WHERE NOT is_active;

SELECT * FROM users WHERE NOT (role = 'admin' OR role = 'staff');

SELECT * FROM orders WHERE NOT status = 'cancelled';
SELECT * FROM orders WHERE status != 'cancelled';   -- equivalent
```

### Operator Precedence

`NOT` binds tightest, then `AND`, then `OR`. Use parentheses to be explicit and avoid bugs:

```sql
-- Ambiguous — AND binds before OR:
SELECT * FROM users
WHERE role = 'admin' OR role = 'staff' AND is_active = true;
-- Evaluated as: role = 'admin' OR (role = 'staff' AND is_active = true)
-- Returns all admins (active or not) + active staff

-- Explicit with parentheses (probably what you meant):
SELECT * FROM users
WHERE (role = 'admin' OR role = 'staff') AND is_active = true;
-- Returns only active admins and active staff
```

> **Always use parentheses when mixing AND and OR.** The implicit precedence rules are a common source of bugs.

---

## 4. Comparison Operators

### Basic Comparisons (=, !=, <, >, <=, >=)

| Operator | Meaning | Example |
|---|---|---|
| `=` | Equal | `status = 'active'` |
| `!=` or `<>` | Not equal | `status != 'cancelled'` |
| `<` | Less than | `age < 18` |
| `>` | Greater than | `salary > 50000` |
| `<=` | Less than or equal | `quantity <= 10` |
| `>=` | Greater than or equal | `created_at >= '2024-01-01'` |

```sql
SELECT * FROM orders WHERE total >= 100 AND total <= 500;
SELECT * FROM employees WHERE salary <> 0;
SELECT * FROM products WHERE stock_qty < 5;
```

> **Edge Case — Comparing different types:**
> PostgreSQL is strict about types. Comparing a `TEXT` column with an integer without casting will error:
> ```sql
> SELECT * FROM orders WHERE id = '5';    -- OK: Postgres casts '5' to integer implicitly
> SELECT * FROM orders WHERE id = 5.0;   -- OK: implicit cast
> SELECT * FROM users  WHERE phone = 5;  -- may error or produce unexpected results
>                                        -- phone is TEXT, 5 is INTEGER
>
> -- Explicit cast to be safe:
> SELECT * FROM orders WHERE id = '5'::INTEGER;
> ```

---

### IN and NOT IN

Test if a value matches any value in a list:

```sql
-- IN: match any of these values
SELECT * FROM users WHERE role IN ('admin', 'staff', 'moderator');

-- NOT IN: match none of these values
SELECT * FROM orders WHERE status NOT IN ('cancelled', 'refunded');

-- IN with a subquery
SELECT * FROM users
WHERE id IN (
  SELECT DISTINCT user_id FROM orders WHERE total > 1000
);

-- NOT IN with a subquery
SELECT * FROM users
WHERE id NOT IN (
  SELECT user_id FROM orders WHERE status = 'completed'
);
```

> **Edge Case — NOT IN with NULL values is a silent trap:**
> ```sql
> -- If the subquery returns even one NULL, NOT IN returns no rows at all:
> SELECT * FROM users
> WHERE id NOT IN (SELECT user_id FROM orders);
> -- If any order has user_id = NULL, this returns EMPTY — every user is excluded
>
> -- Why: x NOT IN (1, 2, NULL) → x!=1 AND x!=2 AND x!=NULL → always NULL → never TRUE
>
> -- Fix: use NOT EXISTS instead (NULL-safe)
> SELECT * FROM users u
> WHERE NOT EXISTS (
>   SELECT 1 FROM orders o WHERE o.user_id = u.id
> );
>
> -- Or filter out NULLs explicitly
> WHERE id NOT IN (SELECT user_id FROM orders WHERE user_id IS NOT NULL);
> ```

---

### BETWEEN and NOT BETWEEN

Inclusive range check — equivalent to `col >= low AND col <= high`:

```sql
-- Numbers
SELECT * FROM orders WHERE total BETWEEN 100 AND 500;

-- Dates
SELECT * FROM orders
WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';

-- NOT BETWEEN
SELECT * FROM products WHERE price NOT BETWEEN 10 AND 50;
```

> **Edge Case — BETWEEN is inclusive on both ends:**
> ```sql
> SELECT * FROM orders WHERE total BETWEEN 100 AND 500;
> -- Equivalent to: total >= 100 AND total <= 500
> -- 100 and 500 are both included
> ```

> **Edge Case — BETWEEN with TIMESTAMPTZ misses end-of-day rows:**
> ```sql
> -- This misses all rows on 2024-12-31 after midnight (00:00:00):
> WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';
> -- '2024-12-31' is treated as '2024-12-31 00:00:00' — only midnight is included
>
> -- Correct: use < next day
> WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
> ```

---

### LIKE and NOT LIKE

Pattern matching with wildcards:

| Wildcard | Meaning | Example Pattern | Matches |
|---|---|---|---|
| `%` | Zero or more characters | `'Al%'` | `Alice`, `Al`, `Alfred` |
| `_` | Exactly one character | `'Al_ce'` | `Alice`, `Al1ce` |

```sql
-- Starts with
SELECT * FROM users WHERE name LIKE 'Al%';

-- Ends with
SELECT * FROM users WHERE email LIKE '%@gmail.com';

-- Contains
SELECT * FROM products WHERE name LIKE '%widget%';

-- Exactly 5 characters
SELECT * FROM codes WHERE code LIKE '_____';

-- NOT LIKE
SELECT * FROM users WHERE email NOT LIKE '%@temp.com';
```

> **Edge Case — LIKE is case-sensitive in PostgreSQL:**
> ```sql
> SELECT * FROM users WHERE name LIKE 'alice%';  -- won't match 'Alice'
>
> -- Use ILIKE for case-insensitive matching (Postgres-specific):
> SELECT * FROM users WHERE name ILIKE 'alice%';  -- matches 'Alice', 'ALICE', 'alice'
> ```

> **Edge Case — Searching for literal `%` or `_`:**
> ```sql
> -- Find products with "50%" in their name
> SELECT * FROM products WHERE name LIKE '%50\%%' ESCAPE '\';
> -- Or use: LIKE '%50!%%' ESCAPE '!'
> ```

> **Edge Case — LIKE with leading `%` cannot use a B-tree index:**
> ```sql
> WHERE name LIKE 'Al%'      -- CAN use index (prefix search)
> WHERE name LIKE '%ice'     -- CANNOT use B-tree index (full scan)
> WHERE name LIKE '%lic%'    -- CANNOT use B-tree index (full scan)
>
> -- For contains/suffix searches, use a trigram index (pg_trgm extension):
> CREATE EXTENSION pg_trgm;
> CREATE INDEX idx_users_name_trgm ON users USING GIN (name gin_trgm_ops);
> -- Now LIKE '%lic%' and ILIKE can use this index
> ```

---

### ILIKE (Case-Insensitive LIKE)

PostgreSQL-specific operator — same as `LIKE` but ignores case:

```sql
SELECT * FROM users WHERE name ILIKE 'alice%';     -- matches Alice, ALICE, alice
SELECT * FROM products WHERE name ILIKE '%laptop%'; -- matches Laptop, LAPTOP, laptop
SELECT * FROM users WHERE email ILIKE '%@GMAIL.COM';

-- NOT ILIKE
SELECT * FROM users WHERE name NOT ILIKE '%test%';
```

> **Edge Case — ILIKE and indexes:** Standard B-tree indexes don't support `ILIKE`. Use a functional index on `LOWER()` or a trigram index:
> ```sql
> -- Functional index for ILIKE prefix searches
> CREATE INDEX idx_users_name_lower ON users (LOWER(name));
> SELECT * FROM users WHERE LOWER(name) LIKE 'alice%';
>
> -- Trigram index for all ILIKE patterns
> CREATE INDEX idx_users_name_trgm ON users USING GIN (name gin_trgm_ops);
> SELECT * FROM users WHERE name ILIKE '%alice%';
> ```

---

## 5. NULL Handling

`NULL` means unknown or absent. It is not a value — it is the **absence of a value**. Any comparison with NULL returns NULL (not TRUE or FALSE).

### IS NULL and IS NOT NULL

```sql
-- Find rows where phone is not set
SELECT * FROM users WHERE phone IS NULL;

-- Find rows where phone is set
SELECT * FROM users WHERE phone IS NOT NULL;

-- Combined
SELECT * FROM orders
WHERE shipped_at IS NULL AND status = 'confirmed';
```

> **Edge Case — Never use `= NULL`:**
> ```sql
> SELECT * FROM users WHERE phone = NULL;     -- always returns 0 rows — WRONG
> SELECT * FROM users WHERE phone != NULL;    -- always returns 0 rows — WRONG
>
> -- NULL = NULL evaluates to NULL, not TRUE
> -- Always use IS NULL / IS NOT NULL
> ```

> **Edge Case — NULL in AND/OR logic (three-valued logic):**
> ```sql
> SELECT NULL AND TRUE;   -- NULL
> SELECT NULL AND FALSE;  -- FALSE  (false short-circuits)
> SELECT NULL OR  TRUE;   -- TRUE   (true short-circuits)
> SELECT NULL OR  FALSE;  -- NULL
> SELECT NOT NULL;        -- NULL
> ```

---

### COALESCE

Returns the **first non-NULL value** from a list of arguments. Essential for replacing NULLs with defaults in output.

```sql
COALESCE(value1, value2, value3, ...)
```

```sql
-- Replace NULL phone with a default string
SELECT name, COALESCE(phone, 'No phone') AS phone FROM users;

-- Use the first available contact method
SELECT
  name,
  COALESCE(mobile, home_phone, work_phone, 'No contact') AS best_contact
FROM contacts;

-- Replace NULL numeric values with 0 for calculations
SELECT
  name,
  COALESCE(salary, 0)  AS salary,
  COALESCE(bonus, 0)   AS bonus,
  COALESCE(salary, 0) + COALESCE(bonus, 0) AS total
FROM employees;

-- In ORDER BY: treat NULLs as a specific value for sorting
SELECT * FROM employees ORDER BY COALESCE(salary, 0) DESC;
```

> **Edge Case — COALESCE is lazy (short-circuits):** It stops evaluating arguments as soon as it finds a non-NULL value. This matters if arguments have side effects (like function calls).

> **Edge Case — COALESCE vs CASE:**
> `COALESCE(a, b)` is exactly equivalent to `CASE WHEN a IS NOT NULL THEN a ELSE b END`. Use whichever reads more clearly.

---

### NULLIF

Returns `NULL` if two values are equal, otherwise returns the first value. The inverse of `COALESCE` in a sense — turns a specific value into NULL.

```sql
NULLIF(value, compare_value)
```

```sql
-- Prevent division by zero: return NULL instead of error when divisor = 0
SELECT total / NULLIF(quantity, 0) AS unit_price FROM order_items;
-- If quantity = 0 → NULLIF returns NULL → total / NULL = NULL (no error)
-- If quantity > 0 → NULLIF returns quantity → normal division

-- Treat empty string same as NULL
SELECT NULLIF(TRIM(notes), '') AS notes FROM orders;
-- '' or '   ' → NULL
-- 'actual note' → 'actual note'

-- Combine with COALESCE for full null/empty handling
SELECT COALESCE(NULLIF(TRIM(notes), ''), 'No notes') AS notes FROM orders;
```

---

### NULL in Expressions and Aggregates

```sql
-- Arithmetic: NULL propagates
SELECT 5 + NULL;       -- NULL
SELECT NULL * 100;     -- NULL
SELECT NULL || 'str';  -- NULL (string concat also propagates NULL in Postgres)

-- Aggregate functions ignore NULL
SELECT AVG(salary)   FROM employees;   -- ignores NULL salaries
SELECT SUM(bonus)    FROM employees;   -- ignores NULL bonuses
SELECT COUNT(phone)  FROM users;       -- counts only non-NULL phone values
SELECT COUNT(*)      FROM users;       -- counts all rows including NULLs

-- COUNT(*) vs COUNT(col)
SELECT
  COUNT(*)      AS total_rows,
  COUNT(phone)  AS rows_with_phone,
  COUNT(*) - COUNT(phone) AS rows_without_phone
FROM users;
```

---

## 6. ORDER BY

`ORDER BY` sorts the result set. Without it, row order is **not guaranteed** — PostgreSQL may return rows in any order.

### ASC and DESC

```sql
-- Default is ASC (ascending, smallest first)
SELECT * FROM products ORDER BY price;
SELECT * FROM products ORDER BY price ASC;   -- same

-- DESC: largest/latest first
SELECT * FROM orders ORDER BY created_at DESC;
SELECT * FROM employees ORDER BY salary DESC;
```

### Multiple Columns

Sort by first column, break ties with second column, and so on:

```sql
-- Sort by role (A→Z), then by name (A→Z) within each role
SELECT * FROM users ORDER BY role ASC, name ASC;

-- Sort by status, then by most expensive order first
SELECT * FROM orders ORDER BY status ASC, total DESC;

-- Three-level sort
SELECT * FROM employees
ORDER BY department ASC, seniority DESC, name ASC;
```

### NULL Ordering

By default in PostgreSQL: NULLs sort **last in ASC**, **first in DESC**.

```sql
-- Default: NULLs last in ASC
SELECT name, salary FROM employees ORDER BY salary ASC;
-- rows: 30000, 50000, 80000, NULL, NULL

-- Default: NULLs first in DESC
SELECT name, salary FROM employees ORDER BY salary DESC;
-- rows: NULL, NULL, 80000, 50000, 30000

-- Override: put NULLs last regardless of direction
SELECT name, salary FROM employees ORDER BY salary DESC NULLS LAST;
-- rows: 80000, 50000, 30000, NULL, NULL

-- Override: put NULLs first regardless of direction
SELECT name, salary FROM employees ORDER BY salary ASC NULLS FIRST;
-- rows: NULL, NULL, 30000, 50000, 80000
```

> **Edge Case — NULL ordering differs across databases:** MySQL and SQL Server sort NULLs first in ASC (opposite of Postgres). Always be explicit with `NULLS FIRST` / `NULLS LAST` in portable SQL.

### ORDER BY with Expressions

```sql
-- Sort by computed value
SELECT name, salary, bonus
FROM employees
ORDER BY (salary + COALESCE(bonus, 0)) DESC;

-- Sort by column position (1-based) — works but fragile, avoid in production
SELECT name, email, created_at FROM users ORDER BY 3 DESC;
-- Same as ORDER BY created_at DESC

-- Sort by alias
SELECT name, salary * 1.1 AS adjusted_salary
FROM employees
ORDER BY adjusted_salary DESC;   -- alias is allowed in ORDER BY

-- Sort by CASE expression
SELECT name, role FROM users
ORDER BY
  CASE role
    WHEN 'admin'    THEN 1
    WHEN 'staff'    THEN 2
    WHEN 'customer' THEN 3
    ELSE                 4
  END;
-- Custom sort order: admins first, then staff, then customers
```

---

## 7. LIMIT and OFFSET

### LIMIT

Restrict the number of rows returned:

```sql
-- Return only the first 10 rows
SELECT * FROM products LIMIT 10;

-- Top 5 most expensive products
SELECT name, price FROM products ORDER BY price DESC LIMIT 5;

-- Check if any row exists (faster than COUNT)
SELECT 1 FROM orders WHERE user_id = 42 LIMIT 1;
```

> **Edge Case — LIMIT without ORDER BY is non-deterministic:**
> ```sql
> SELECT * FROM users LIMIT 10;
> -- Returns 10 rows, but which 10? No guarantee across executions.
> -- Always pair LIMIT with ORDER BY for consistent results:
> SELECT * FROM users ORDER BY id LIMIT 10;
> ```

### OFFSET

Skip a number of rows before returning results:

```sql
-- Skip first 10 rows, return next 10
SELECT * FROM products ORDER BY id LIMIT 10 OFFSET 10;

-- Page 3 (0-indexed pages of 10 rows each)
SELECT * FROM products ORDER BY id LIMIT 10 OFFSET 20;
```

### Pagination Pattern

```sql
-- Page formula: OFFSET = (page_number - 1) * page_size
-- Page 1: OFFSET 0
SELECT * FROM products ORDER BY id LIMIT 10 OFFSET 0;

-- Page 2: OFFSET 10
SELECT * FROM products ORDER BY id LIMIT 10 OFFSET 10;

-- Page 3: OFFSET 20
SELECT * FROM products ORDER BY id LIMIT 10 OFFSET 20;

-- Page N (parameterized):
-- LIMIT 10 OFFSET (N - 1) * 10
```

```sql
-- Count total records for pagination UI
SELECT COUNT(*) AS total FROM products;
-- Use with: total_pages = CEIL(total / page_size)
```

> **Edge Case — OFFSET performance degrades on large tables:**
> ```sql
> SELECT * FROM orders ORDER BY id LIMIT 10 OFFSET 100000;
> -- PostgreSQL must scan and discard 100,000 rows before returning 10
> -- Gets slower as OFFSET increases — bad for deep pages
>
> -- Fix: Keyset pagination (cursor-based) — much faster
> -- Instead of OFFSET, remember the last seen id:
> SELECT * FROM orders
> WHERE id > :last_seen_id   -- "cursor" from previous page
> ORDER BY id
> LIMIT 10;
> -- Always O(log n) via index, regardless of page depth
> ```

> **Edge Case — OFFSET + concurrent inserts/deletes cause row skips or duplicates:**
> ```sql
> -- Session A reads page 1 (rows 1-10)
> -- Session B deletes row 5
> -- Session A reads page 2 with OFFSET 10 → row 11 is now row 10 → row 10 appears again
> -- Keyset pagination is immune to this problem
> ```

### FETCH (SQL Standard Alternative)

`FETCH` is the SQL:2008 standard equivalent of `LIMIT/OFFSET`. Both work in PostgreSQL:

```sql
-- LIMIT/OFFSET (PostgreSQL style)
SELECT * FROM products ORDER BY id LIMIT 10 OFFSET 20;

-- FETCH NEXT (SQL standard)
SELECT * FROM products ORDER BY id
OFFSET 20 ROWS
FETCH NEXT 10 ROWS ONLY;

-- First row only
SELECT * FROM products ORDER BY price DESC
FETCH FIRST ROW ONLY;

-- First N rows
SELECT * FROM products ORDER BY price DESC
FETCH FIRST 5 ROWS ONLY;
```

---

## 8. DISTINCT

`DISTINCT` eliminates duplicate rows from the result set.

### DISTINCT on a Single Column

```sql
-- All unique roles in the users table
SELECT DISTINCT role FROM users;

-- All unique statuses
SELECT DISTINCT status FROM orders;
```

### DISTINCT on Multiple Columns

When applied to multiple columns, deduplication is based on the **combination** of all selected columns:

```sql
-- Unique (department, role) combinations
SELECT DISTINCT department, role FROM employees ORDER BY department, role;
```

```sql
-- Example data and result:
-- department | role
-- -----------+--------
-- Engineering | senior    ← unique combo
-- Engineering | junior    ← unique combo
-- Engineering | senior    ← duplicate — removed
-- Marketing   | senior    ← unique combo
--
-- Result:
-- Engineering | junior
-- Engineering | senior
-- Marketing   | senior
```

> **Edge Case — DISTINCT applies to the entire row, not individual columns:**
> ```sql
> SELECT DISTINCT department, name FROM employees;
> -- Deduplicates on (department, name) together
> -- NOT "unique departments" — if two employees have the same dept but different names,
> -- both rows appear
> ```

> **Edge Case — DISTINCT and ORDER BY:**
> Columns in `ORDER BY` must appear in the `SELECT` list when using `DISTINCT`:
> ```sql
> -- ERROR: for SELECT DISTINCT, ORDER BY expressions must appear in select list
> SELECT DISTINCT department FROM employees ORDER BY name;
>
> -- Fix: include the ORDER BY column in SELECT
> SELECT DISTINCT department, name FROM employees ORDER BY name;
> ```

> **Edge Case — DISTINCT is expensive:** It performs an implicit sort or hash to find duplicates. On large tables, prefer alternatives when possible:
> - Use `GROUP BY` instead — it's often faster and more explicit
> - Use proper constraints (UNIQUE) to prevent duplicates at the source
> ```sql
> -- DISTINCT
> SELECT DISTINCT user_id FROM orders;
>
> -- Equivalent GROUP BY (often faster)
> SELECT user_id FROM orders GROUP BY user_id;
> ```

---

### DISTINCT ON (PostgreSQL-specific)

`DISTINCT ON` keeps only the **first row** for each unique value of the specified expression. It must be paired with `ORDER BY` to control which row is "first."

```sql
DISTINCT ON (expression) → keep the first row per unique expression value
```

```sql
-- Get the most recent order per user
SELECT DISTINCT ON (user_id)
  user_id,
  id       AS order_id,
  total,
  created_at
FROM orders
ORDER BY user_id, created_at DESC;
-- For each user_id, keeps the row with the latest created_at
```

```sql
-- Get the cheapest product per category
SELECT DISTINCT ON (category)
  category,
  name,
  price
FROM products
ORDER BY category, price ASC;
-- For each category, keeps the row with the lowest price
```

```sql
-- Get the latest status per order (from an order_events log table)
SELECT DISTINCT ON (order_id)
  order_id,
  status,
  changed_at
FROM order_events
ORDER BY order_id, changed_at DESC;
```

> **Edge Case — DISTINCT ON column must be leftmost in ORDER BY:**
> ```sql
> -- ERROR: SELECT DISTINCT ON expressions must match initial ORDER BY expressions
> SELECT DISTINCT ON (user_id) user_id, total
> FROM orders
> ORDER BY total DESC;   -- must start with user_id, not total
>
> -- Fix: ORDER BY must start with the DISTINCT ON column(s)
> SELECT DISTINCT ON (user_id) user_id, total
> FROM orders
> ORDER BY user_id, total DESC;
> ```

> **Edge Case — DISTINCT ON vs GROUP BY:** `DISTINCT ON` returns full rows (not aggregates), while `GROUP BY` collapses rows into summaries. Use `DISTINCT ON` when you want the actual "winner" row, not an aggregate over the group.

---

## Quick Reference

```sql
-- SELECT
SELECT col1, col2 FROM t;
SELECT col AS alias FROM t;
SELECT * FROM t;                         -- avoid in production
SELECT 1 + 1, NOW(), UPPER('hello');     -- no FROM needed

-- WHERE
SELECT * FROM t WHERE col = 'val';
SELECT * FROM t WHERE col > 100 AND other = 'x';
SELECT * FROM t WHERE col = 'a' OR col = 'b';
SELECT * FROM t WHERE NOT col = 'x';

-- Comparisons
col = 'x'  |  col != 'x'  |  col <> 'x'
col > n    |  col >= n     |  col < n  |  col <= n
col IN ('a', 'b', 'c')
col NOT IN ('a', 'b')
col BETWEEN 10 AND 50           -- inclusive both ends
col NOT BETWEEN 10 AND 50
col LIKE 'Al%'                  -- case-sensitive, % = any chars
col LIKE '_l%'                  -- _ = exactly one char
col ILIKE '%alice%'             -- case-insensitive (Postgres)
col NOT LIKE '%tmp%'

-- NULL
col IS NULL
col IS NOT NULL
COALESCE(col, 'default')        -- first non-NULL value
NULLIF(col, 0)                  -- return NULL if col = 0
NULLIF(TRIM(col), '')           -- treat empty string as NULL

-- ORDER BY
ORDER BY col ASC
ORDER BY col DESC
ORDER BY col1 ASC, col2 DESC
ORDER BY col DESC NULLS LAST
ORDER BY col ASC  NULLS FIRST
ORDER BY CASE WHEN col='a' THEN 1 ELSE 2 END  -- custom order

-- LIMIT / OFFSET
LIMIT 10
LIMIT 10 OFFSET 20              -- page 3 (0-indexed, 10 per page)
FETCH FIRST 10 ROWS ONLY        -- SQL standard equivalent

-- DISTINCT
SELECT DISTINCT col FROM t
SELECT DISTINCT col1, col2 FROM t

-- DISTINCT ON (Postgres-specific): first row per group
SELECT DISTINCT ON (user_id) user_id, total, created_at
FROM orders
ORDER BY user_id, created_at DESC;
```
