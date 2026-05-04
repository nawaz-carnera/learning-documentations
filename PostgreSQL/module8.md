# Module 8: Aggregates — GROUP BY, HAVING, and Aggregation Functions

---

## Table of Contents

- [1. Aggregate Functions](#1-aggregate-functions)
  - [COUNT](#count)
  - [SUM](#sum)
  - [AVG](#avg)
  - [MIN and MAX](#min-and-max)
  - [Combining Aggregates](#combining-aggregates)
- [2. GROUP BY One Column](#2-group-by-one-column)
- [3. GROUP BY Multiple Columns](#3-group-by-multiple-columns)
- [4. HAVING Clause](#4-having-clause)
- [5. Difference Between WHERE and HAVING](#5-difference-between-where-and-having)
- [6. STRING_AGG](#6-string_agg)
- [7. ARRAY_AGG](#7-array_agg)
- [Quick Reference](#quick-reference)

---

## 1. Aggregate Functions

Aggregate functions **collapse multiple rows into a single value**. They work across a set of rows and return one result per group (or one result for the whole table if there is no `GROUP BY`).

### Setup (tables used throughout this module)

```sql
CREATE TABLE employees (
  id         SERIAL PRIMARY KEY,
  name       TEXT           NOT NULL,
  department TEXT           NOT NULL,
  role       TEXT           NOT NULL,
  salary     NUMERIC(10,2),
  bonus      NUMERIC(10,2),
  joined_at  DATE           NOT NULL DEFAULT CURRENT_DATE
);

INSERT INTO employees (name, department, role, salary, bonus) VALUES
  ('Alice',   'Engineering', 'senior',  120000, 15000),
  ('Bob',     'Engineering', 'junior',   65000,  5000),
  ('Charlie', 'Engineering', 'senior',  115000, 12000),
  ('Dana',    'Marketing',   'manager',  90000, 10000),
  ('Eve',     'Marketing',   'junior',   55000,  3000),
  ('Frank',   'HR',          'manager',  80000,  8000),
  ('Grace',   'HR',          'junior',   52000,  NULL),
  ('Hank',    'Engineering', 'junior',   68000,  4500),
  ('Ivy',     'Marketing',   'senior',   95000, 11000),
  ('Jay',     'HR',          'junior',   50000,  NULL);
```

---

### COUNT

Counts rows or non-NULL values.

```sql
-- COUNT(*): count all rows including NULLs
SELECT COUNT(*) FROM employees;
-- 10

-- COUNT(col): count only non-NULL values in a column
SELECT COUNT(bonus) FROM employees;
-- 8  (Grace and Jay have NULL bonus — excluded)

-- COUNT(DISTINCT col): count unique non-NULL values
SELECT COUNT(DISTINCT department) FROM employees;
-- 3  (Engineering, Marketing, HR)

-- Count with a condition (using FILTER — Postgres 9.4+)
SELECT
  COUNT(*)                                    AS total,
  COUNT(*) FILTER (WHERE department = 'Engineering') AS eng_count,
  COUNT(*) FILTER (WHERE bonus IS NOT NULL)          AS with_bonus
FROM employees;
```

```
 total | eng_count | with_bonus
-------+-----------+------------
    10 |         4 |          8
```

> **Edge Case — COUNT(*) vs COUNT(col) vs COUNT(1):**
> ```sql
> COUNT(*)    -- counts all rows regardless of NULLs — use this for row counts
> COUNT(col)  -- counts rows where col IS NOT NULL
> COUNT(1)    -- same as COUNT(*) — counts all rows (the 1 is never NULL)
>
> -- Common mistake: using COUNT(col) expecting a row count
> SELECT COUNT(bonus) FROM employees;  -- returns 8, not 10
> -- Grace and Jay with NULL bonus are not counted
> ```

> **Edge Case — COUNT on an empty table returns 0, not NULL:**
> ```sql
> SELECT COUNT(*) FROM employees WHERE department = 'Finance';
> -- Returns 0 — not NULL (unlike other aggregates)
> ```

---

### SUM

Adds up numeric values, ignoring NULLs.

```sql
-- Total salary bill
SELECT SUM(salary) FROM employees;
-- 790000.00

-- Total bonus paid (NULLs ignored)
SELECT SUM(bonus) FROM employees;
-- 68500.00  (Grace and Jay's NULL not included)

-- SUM with FILTER
SELECT
  SUM(salary)                                        AS total_salary,
  SUM(salary) FILTER (WHERE department = 'Engineering') AS eng_salary,
  SUM(bonus)  FILTER (WHERE bonus IS NOT NULL)           AS total_bonus
FROM employees;

-- SUM with expression
SELECT SUM(salary + COALESCE(bonus, 0)) AS total_compensation
FROM employees;
```

> **Edge Case — SUM of all NULLs returns NULL, not 0:**
> ```sql
> CREATE TABLE test (val INTEGER);
> INSERT INTO test VALUES (NULL), (NULL);
>
> SELECT SUM(val) FROM test;   -- NULL, not 0
>
> -- Fix: wrap in COALESCE
> SELECT COALESCE(SUM(val), 0) FROM test;   -- 0
> ```

> **Edge Case — SUM on an empty result set returns NULL:**
> ```sql
> SELECT SUM(salary) FROM employees WHERE department = 'Finance';
> -- NULL (no rows match) — use COALESCE(SUM(...), 0) if you need 0
> ```

---

### AVG

Returns the arithmetic mean, ignoring NULLs.

```sql
-- Average salary across all employees
SELECT AVG(salary) FROM employees;
-- 79000.000000000000

-- Round the result
SELECT ROUND(AVG(salary), 2) AS avg_salary FROM employees;
-- 79000.00

-- Average only non-NULL bonuses (Grace and Jay excluded from average)
SELECT ROUND(AVG(bonus), 2) AS avg_bonus FROM employees;
-- 8562.50  (68500 / 8, not / 10)

-- Average treating NULL bonus as 0 (include everyone in denominator)
SELECT ROUND(AVG(COALESCE(bonus, 0)), 2) AS avg_bonus_incl_zero FROM employees;
-- 6850.00  (68500 / 10)
```

> **Edge Case — AVG excludes NULLs from the denominator:**
> This is often the correct behavior (average bonus of people who received one).
> But if you want to include non-recipients in the average, use `COALESCE(col, 0)`.
> Be intentional — the two answers have very different meanings.

> **Edge Case — AVG on integers returns numeric, not integer:**
> ```sql
> SELECT AVG(salary) FROM employees;
> -- Returns: 79000.000000000000 (high-precision numeric)
> -- Use ROUND() or ::NUMERIC(10,2) to control decimal places
> SELECT ROUND(AVG(salary)::NUMERIC, 2);
> ```

---

### MIN and MAX

Return the smallest and largest value in the group, ignoring NULLs.

```sql
-- Salary range
SELECT MIN(salary) AS lowest, MAX(salary) AS highest FROM employees;
-- 50000.00 | 120000.00

-- Date range
SELECT MIN(joined_at) AS earliest, MAX(joined_at) AS latest FROM employees;

-- Works on TEXT too (alphabetical order)
SELECT MIN(name) AS first_alpha, MAX(name) AS last_alpha FROM employees;
-- Alice | Jay

-- Salary spread
SELECT MAX(salary) - MIN(salary) AS salary_range FROM employees;
-- 70000.00
```

> **Edge Case — MIN/MAX on an empty set returns NULL:**
> ```sql
> SELECT MIN(salary) FROM employees WHERE department = 'Finance';
> -- NULL — use COALESCE if you need a fallback value
> ```

> **Edge Case — MIN/MAX ignore NULLs:**
> ```sql
> -- Grace and Jay have NULL bonus
> SELECT MIN(bonus) FROM employees;  -- 3000 (NULL not compared)
> SELECT MAX(bonus) FROM employees;  -- 15000
> -- NULLs are silently ignored in MIN/MAX comparisons
> ```

---

### Combining Aggregates

Multiple aggregates can be computed in one pass — no need for separate queries:

```sql
SELECT
  COUNT(*)                          AS headcount,
  COUNT(bonus)                      AS with_bonus,
  ROUND(AVG(salary), 2)             AS avg_salary,
  SUM(salary)                       AS total_salary,
  MIN(salary)                       AS min_salary,
  MAX(salary)                       AS max_salary,
  MAX(salary) - MIN(salary)         AS salary_spread,
  SUM(COALESCE(bonus, 0))           AS total_bonus_paid
FROM employees;
```

```
 headcount | with_bonus | avg_salary | total_salary | min_salary | max_salary | salary_spread | total_bonus_paid
-----------+------------+------------+--------------+------------+------------+---------------+-----------------
        10 |          8 |   79000.00 |    790000.00 |   50000.00 |  120000.00 |      70000.00 |         68500.00
```

---

## 2. GROUP BY One Column

`GROUP BY` splits rows into groups by a column's unique values, then applies aggregate functions **per group** instead of across the whole table.

```sql
SELECT column, AGGREGATE(other_col)
FROM table
GROUP BY column;
```

**Rule:** Every column in `SELECT` must either be:
1. Listed in `GROUP BY`, or
2. Wrapped in an aggregate function

```sql
-- Headcount per department
SELECT department, COUNT(*) AS headcount
FROM employees
GROUP BY department;
```
```
 department  | headcount
-------------+-----------
 Engineering |         4
 HR          |         3
 Marketing   |         3
```

```sql
-- Total salary cost per department
SELECT department, SUM(salary) AS total_salary
FROM employees
GROUP BY department
ORDER BY total_salary DESC;
```
```
 department  | total_salary
-------------+--------------
 Engineering |    368000.00
 Marketing   |    240000.00
 HR          |    182000.00
```

```sql
-- Full summary per department
SELECT
  department,
  COUNT(*)                  AS headcount,
  ROUND(AVG(salary), 2)     AS avg_salary,
  MIN(salary)               AS min_salary,
  MAX(salary)               AS max_salary,
  SUM(salary)               AS total_salary,
  COUNT(bonus)              AS received_bonus,
  ROUND(AVG(bonus), 2)      AS avg_bonus
FROM employees
GROUP BY department
ORDER BY department;
```

> **Edge Case — Selecting a non-aggregated, non-grouped column:**
> ```sql
> -- ERROR: column "employees.name" must appear in the GROUP BY clause
> --        or be used in an aggregate function
> SELECT department, name, COUNT(*)
> FROM employees
> GROUP BY department;
>
> -- Fix: either group by both columns, or aggregate the name column
> SELECT department, COUNT(*), STRING_AGG(name, ', ') AS members
> FROM employees
> GROUP BY department;
> ```

> **Edge Case — GROUP BY column position (use with caution):**
> ```sql
> SELECT department, COUNT(*) FROM employees GROUP BY 1;
> -- 1 refers to the first SELECT column (department)
> -- Works but fragile — breaks if column order changes
> ```

---

## 3. GROUP BY Multiple Columns

Group by the **combination** of multiple columns — one result row per unique combination.

```sql
-- Headcount per department AND role
SELECT department, role, COUNT(*) AS headcount
FROM employees
GROUP BY department, role
ORDER BY department, role;
```
```
 department  |  role   | headcount
-------------+---------+-----------
 Engineering | junior  |         2
 Engineering | senior  |         2
 HR          | junior  |         2
 HR          | manager |         1
 Marketing   | junior  |         1
 Marketing   | manager |         1
 Marketing   | senior  |         1
```

```sql
-- Average salary per department and role
SELECT
  department,
  role,
  COUNT(*)               AS headcount,
  ROUND(AVG(salary), 2)  AS avg_salary,
  SUM(salary)            AS total_salary
FROM employees
GROUP BY department, role
ORDER BY department, avg_salary DESC;
```

```sql
-- Group by a computed expression
SELECT
  DATE_TRUNC('month', joined_at) AS join_month,
  department,
  COUNT(*) AS new_hires
FROM employees
GROUP BY DATE_TRUNC('month', joined_at), department
ORDER BY join_month, department;
```

> **Edge Case — Grouping by expression must match the SELECT expression exactly:**
> ```sql
> -- If you GROUP BY an expression, use it consistently
> SELECT EXTRACT(YEAR FROM joined_at) AS year, COUNT(*)
> FROM employees
> GROUP BY EXTRACT(YEAR FROM joined_at);  -- must match the expression, not the alias
>
> -- Postgres allows grouping by alias in some cases but it's unreliable across versions
> -- Safest: repeat the expression in GROUP BY
> ```

> **Edge Case — NULL values in GROUP BY columns:**
> ```sql
> -- NULLs in the grouping column form their own group
> SELECT bonus, COUNT(*) FROM employees GROUP BY bonus;
> -- All rows where bonus IS NULL become one group
> -- NULL is treated as equal to NULL for grouping purposes
> --   (opposite of WHERE comparisons where NULL != NULL)
> ```

---

## 4. HAVING Clause

`HAVING` filters **groups** after aggregation — the equivalent of `WHERE` but for aggregate results.

```sql
SELECT column, AGGREGATE(col)
FROM table
GROUP BY column
HAVING AGGREGATE(col) condition;
```

```sql
-- Departments with more than 2 employees
SELECT department, COUNT(*) AS headcount
FROM employees
GROUP BY department
HAVING COUNT(*) > 2;
```
```
 department  | headcount
-------------+-----------
 Engineering |         4
```

```sql
-- Departments where average salary exceeds 75,000
SELECT department, ROUND(AVG(salary), 2) AS avg_salary
FROM employees
GROUP BY department
HAVING AVG(salary) > 75000
ORDER BY avg_salary DESC;
```
```
 department  | avg_salary
-------------+------------
 Engineering |   92000.00
 Marketing   |   80000.00
```

```sql
-- Departments with total salary bill over 200,000
SELECT department, SUM(salary) AS total_salary
FROM employees
GROUP BY department
HAVING SUM(salary) > 200000;
```

```sql
-- Roles that appear in more than one department
SELECT role, COUNT(DISTINCT department) AS dept_count
FROM employees
GROUP BY role
HAVING COUNT(DISTINCT department) > 1;
```

```sql
-- Groups where ALL members have a bonus (no NULLs)
SELECT department, COUNT(*) AS headcount, COUNT(bonus) AS with_bonus
FROM employees
GROUP BY department
HAVING COUNT(*) = COUNT(bonus);   -- headcount = bonus count → no NULLs
```

> **Edge Case — HAVING without GROUP BY:**
> ```sql
> -- HAVING can be used without GROUP BY — treats the whole table as one group
> SELECT COUNT(*), AVG(salary) FROM employees
> HAVING AVG(salary) > 70000;
> -- Returns the row only if the overall average exceeds 70000, otherwise returns nothing
> ```

> **Edge Case — Cannot use SELECT aliases in HAVING:**
> ```sql
> SELECT department, AVG(salary) AS avg_sal
> FROM employees
> GROUP BY department
> HAVING avg_sal > 75000;   -- ERROR: column "avg_sal" does not exist
>
> -- Must repeat the expression:
> HAVING AVG(salary) > 75000;
> ```

---

## 5. Difference Between WHERE and HAVING

This is one of the most commonly confused topics in SQL.

| | WHERE | HAVING |
|---|---|---|
| Filters | **Rows** (before grouping) | **Groups** (after grouping) |
| Evaluated | Before `GROUP BY` | After `GROUP BY` |
| Can use aggregates | No | Yes |
| Can use raw columns | Yes | Yes (if in GROUP BY) |
| Performance | Faster — reduces rows before aggregation | Slower — aggregation happens first |

### SQL Execution Order

```
FROM
  → WHERE       ← filters individual rows (no aggregates yet)
  → GROUP BY    ← groups the filtered rows
  → HAVING      ← filters groups (aggregates available here)
  → SELECT      ← computes output columns
  → ORDER BY    ← sorts the output
  → LIMIT       ← restricts row count
```

### Side-by-side Example

```sql
-- WHERE: filter rows BEFORE grouping
-- Only consider Engineering employees, then group
SELECT department, COUNT(*) AS headcount, AVG(salary) AS avg_salary
FROM employees
WHERE department = 'Engineering'       -- removes non-Engineering rows first
GROUP BY department;

-- HAVING: filter groups AFTER grouping
-- Group all employees, then keep only groups with avg > 75000
SELECT department, COUNT(*) AS headcount, AVG(salary) AS avg_salary
FROM employees
GROUP BY department
HAVING AVG(salary) > 75000;           -- filters groups, not rows

-- Combined: WHERE filters rows first, HAVING filters groups after
SELECT department, COUNT(*) AS headcount, ROUND(AVG(salary), 2) AS avg_salary
FROM employees
WHERE role != 'junior'                 -- exclude juniors from consideration
GROUP BY department
HAVING AVG(salary) > 85000            -- keep only high-avg departments
ORDER BY avg_salary DESC;
```

### The key question to ask yourself:

> **"Am I filtering on a raw column value, or on the result of an aggregation?"**
> - Raw column value → `WHERE`
> - Aggregate result (COUNT, SUM, AVG…) → `HAVING`

```sql
-- WRONG: cannot use aggregate in WHERE
SELECT department, COUNT(*)
FROM employees
WHERE COUNT(*) > 2          -- ERROR: aggregate functions not allowed in WHERE
GROUP BY department;

-- RIGHT: use HAVING for aggregate filter
SELECT department, COUNT(*)
FROM employees
GROUP BY department
HAVING COUNT(*) > 2;

-- WRONG: using HAVING for a simple row filter (works but wasteful)
SELECT department, COUNT(*)
FROM employees
GROUP BY department
HAVING department = 'Engineering';    -- works, but WHERE is better here

-- RIGHT: use WHERE for simple row filter (faster — fewer rows to group)
SELECT department, COUNT(*)
FROM employees
WHERE department = 'Engineering'
GROUP BY department;
```

---

## 6. STRING_AGG

`STRING_AGG` concatenates string values from multiple rows into a single string, with a specified separator. NULLs are silently ignored.

```sql
STRING_AGG(expression, delimiter)
STRING_AGG(expression, delimiter ORDER BY sort_expression)
```

### Basic Usage

```sql
-- List all employee names per department as a comma-separated string
SELECT
  department,
  STRING_AGG(name, ', ') AS members
FROM employees
GROUP BY department
ORDER BY department;
```
```
 department  |              members
-------------+-------------------------------------
 Engineering | Alice, Bob, Charlie, Hank
 HR          | Frank, Grace, Jay
 Marketing   | Dana, Eve, Ivy
```

### With ORDER BY inside STRING_AGG

```sql
-- Names sorted alphabetically within each department
SELECT
  department,
  STRING_AGG(name, ', ' ORDER BY name) AS members_sorted
FROM employees
GROUP BY department;
```
```
 department  |          members_sorted
-------------+----------------------------------
 Engineering | Alice, Bob, Charlie, Hank
 HR          | Frank, Grace, Jay
 Marketing   | Dana, Eve, Ivy
```

```sql
-- Names sorted by salary descending (highest earner first)
SELECT
  department,
  STRING_AGG(name, ' > ' ORDER BY salary DESC) AS salary_ranking
FROM employees
GROUP BY department;
```
```
 department  |        salary_ranking
-------------+--------------------------------
 Engineering | Alice > Charlie > Hank > Bob
 HR          | Frank > Grace > Jay
 Marketing   | Ivy > Dana > Eve
```

### Practical Patterns

```sql
-- Build a comma-separated tag list per product
SELECT
  product_id,
  STRING_AGG(tag, ', ' ORDER BY tag) AS tags
FROM product_tags
GROUP BY product_id;

-- Build pipe-delimited list of roles per user
SELECT
  user_id,
  STRING_AGG(role_name, ' | ' ORDER BY role_name) AS roles
FROM user_roles
GROUP BY user_id;

-- Combine with DISTINCT to deduplicate before aggregating
SELECT
  department,
  STRING_AGG(DISTINCT role, ', ' ORDER BY role) AS unique_roles
FROM employees
GROUP BY department;
```
```
 department  |     unique_roles
-------------+----------------------
 Engineering | junior, senior
 HR          | junior, manager
 Marketing   | junior, manager, senior
```

### With FILTER

```sql
-- Separate lists of seniors vs juniors per department
SELECT
  department,
  STRING_AGG(name, ', ') FILTER (WHERE role = 'senior') AS seniors,
  STRING_AGG(name, ', ') FILTER (WHERE role = 'junior') AS juniors
FROM employees
GROUP BY department
ORDER BY department;
```
```
 department  |      seniors       |    juniors
-------------+--------------------+---------------
 Engineering | Alice, Charlie     | Bob, Hank
 HR          | NULL               | Grace, Jay
 Marketing   | Ivy                | Eve
```

> **Edge Case — STRING_AGG returns NULL when all input values are NULL:**
> ```sql
> SELECT STRING_AGG(bonus::TEXT, ', ') FROM employees WHERE bonus IS NULL;
> -- Returns NULL, not ''
> -- Use COALESCE to handle: COALESCE(STRING_AGG(...), 'none')
> ```

> **Edge Case — Order inside STRING_AGG is independent of query ORDER BY:**
> ```sql
> -- The outer ORDER BY sorts the rows of the result
> -- The inner ORDER BY (inside STRING_AGG) sorts values within the string
> SELECT department,
>   STRING_AGG(name, ', ' ORDER BY salary DESC) AS ranked_names
> FROM employees
> GROUP BY department
> ORDER BY department;   -- outer: sorts result rows by department name
>                        -- inner ORDER BY salary: sorts names within each string
> ```

> **Edge Case — Large concatenations:** `STRING_AGG` with no row limit can produce very large strings. If you have thousands of rows per group, consider whether you really need the full string or just the first N values.

---

## 7. ARRAY_AGG

`ARRAY_AGG` collects values from multiple rows into a PostgreSQL **array**. Unlike `STRING_AGG`, it preserves the original data type and produces a typed array rather than a text string.

```sql
ARRAY_AGG(expression)
ARRAY_AGG(expression ORDER BY sort_expression)
```

### Basic Usage

```sql
-- Collect employee names per department into an array
SELECT
  department,
  ARRAY_AGG(name) AS members
FROM employees
GROUP BY department
ORDER BY department;
```
```
 department  |          members
-------------+-----------------------------
 Engineering | {Alice,Bob,Charlie,Hank}
 HR          | {Frank,Grace,Jay}
 Marketing   | {Dana,Eve,Ivy}
```

### With ORDER BY

```sql
-- Array of salaries sorted highest first
SELECT
  department,
  ARRAY_AGG(salary ORDER BY salary DESC) AS salaries
FROM employees
GROUP BY department;
```
```
 department  |              salaries
-------------+--------------------------------------
 Engineering | {120000,115000,68000,65000}
 HR          | {80000,52000,50000}
 Marketing   | {95000,90000,55000}
```

### With DISTINCT

```sql
-- Unique roles per department as an array
SELECT
  department,
  ARRAY_AGG(DISTINCT role ORDER BY role) AS roles
FROM employees
GROUP BY department;
```
```
 department  |        roles
-------------+---------------------
 Engineering | {junior,senior}
 HR          | {junior,manager}
 Marketing   | {junior,manager,senior}
```

### Querying the Arrays

```sql
-- Get the first element of the array (1-indexed in PostgreSQL)
SELECT department, (ARRAY_AGG(name ORDER BY salary DESC))[1] AS top_earner
FROM employees
GROUP BY department;
```
```
 department  | top_earner
-------------+------------
 Engineering | Alice
 HR          | Frank
 Marketing   | Ivy
```

```sql
-- Check if a value is in the aggregated array
SELECT department, ARRAY_AGG(name) AS members
FROM employees
GROUP BY department
HAVING 'Alice' = ANY(ARRAY_AGG(name));
```

### ARRAY_AGG vs STRING_AGG

| | ARRAY_AGG | STRING_AGG |
|---|---|---|
| Return type | `type[]` (typed array) | `TEXT` |
| NULLs included | Yes (by default) | No (silently skipped) |
| Indexable result | Yes (`arr[1]`, `ANY`, `@>`) | No |
| Human-readable | Not directly | Yes |
| Use when | You need to query the result further | You need a display string |

```sql
-- ARRAY_AGG includes NULLs by default
SELECT ARRAY_AGG(bonus) FROM employees;
-- {15000,5000,12000,10000,3000,8000,NULL,4500,11000,NULL}

-- Filter NULLs out explicitly
SELECT ARRAY_AGG(bonus) FILTER (WHERE bonus IS NOT NULL) FROM employees;
-- {15000,5000,12000,10000,3000,8000,4500,11000}

-- STRING_AGG silently ignores NULLs (no FILTER needed)
SELECT STRING_AGG(bonus::TEXT, ', ') FROM employees;
-- 15000, 5000, 12000, 10000, 3000, 8000, 4500, 11000
```

### Practical Pattern — Aggregate Then Unnest

```sql
-- Pack rows into an array, then unpack (unnest) them — useful in CTEs/pipelines
SELECT UNNEST(ARRAY_AGG(name ORDER BY salary DESC)) AS name_by_salary
FROM employees
WHERE department = 'Engineering';
```
```
 name_by_salary
----------------
 Alice
 Charlie
 Hank
 Bob
```

```sql
-- Store aggregated arrays in another table
INSERT INTO department_summaries (department, member_names, salaries)
SELECT
  department,
  ARRAY_AGG(name    ORDER BY name),
  ARRAY_AGG(salary  ORDER BY salary DESC)
FROM employees
GROUP BY department;
```

> **Edge Case — Array indexing is 1-based in PostgreSQL (not 0-based):**
> ```sql
> SELECT (ARRAY_AGG(name))[0] FROM employees GROUP BY department;
> -- Returns NULL — index 0 does not exist in Postgres arrays
>
> SELECT (ARRAY_AGG(name))[1] FROM employees GROUP BY department;
> -- Returns the first element — correct
> ```

> **Edge Case — ARRAY_AGG on an empty set returns NULL, not `{}`:**
> ```sql
> SELECT ARRAY_AGG(name) FROM employees WHERE department = 'Finance';
> -- NULL, not an empty array {}
>
> -- Fix: use COALESCE
> SELECT COALESCE(ARRAY_AGG(name), ARRAY[]::TEXT[]) FROM employees
> WHERE department = 'Finance';
> -- {} (empty array)
> ```

> **Edge Case — Very large arrays:** `ARRAY_AGG` with no limit can exhaust memory on large groups. If you only need the top N, use a subquery with `ORDER BY` + `LIMIT` before aggregating, or use `ARRAY_AGG(...)[1:N]` to slice the result:
> ```sql
> -- Keep only first 3 names per department (array slice)
> SELECT department, (ARRAY_AGG(name ORDER BY salary DESC))[1:3] AS top_3
> FROM employees
> GROUP BY department;
> ```

---

## Quick Reference

```sql
-- Aggregate functions
COUNT(*)                          -- all rows including NULLs
COUNT(col)                        -- non-NULL values only
COUNT(DISTINCT col)               -- unique non-NULL values
SUM(col)                          -- total, NULLs ignored
AVG(col)                          -- mean, NULLs ignored
MIN(col)                          -- smallest value, NULLs ignored
MAX(col)                          -- largest value, NULLs ignored

-- FILTER modifier (Postgres 9.4+)
COUNT(*) FILTER (WHERE condition)
SUM(col) FILTER (WHERE condition)
AVG(col) FILTER (WHERE condition)

-- Safe patterns for empty/null results
COALESCE(SUM(col),  0)            -- 0 instead of NULL on empty set
COALESCE(AVG(col),  0)
COALESCE(COUNT(col), 0)           -- COUNT always returns 0, no COALESCE needed

-- GROUP BY
SELECT col, AGG(x)   FROM t GROUP BY col;
SELECT c1, c2, AGG() FROM t GROUP BY c1, c2;
SELECT EXTRACT(YEAR FROM d), AGG() FROM t GROUP BY EXTRACT(YEAR FROM d);

-- HAVING
HAVING COUNT(*) > 5
HAVING AVG(salary) > 75000
HAVING SUM(total) BETWEEN 1000 AND 5000
HAVING COUNT(*) = COUNT(col)      -- all rows have non-NULL col

-- WHERE vs HAVING
WHERE  col = 'x'                  -- filter rows before grouping (fast)
HAVING AGG(col) > n               -- filter groups after aggregation

-- STRING_AGG
STRING_AGG(col, ', ')                          -- comma-separated list
STRING_AGG(col, ', ' ORDER BY col)             -- sorted list
STRING_AGG(DISTINCT col, ', ' ORDER BY col)    -- deduplicated sorted list
STRING_AGG(col, ', ') FILTER (WHERE cond)      -- conditional aggregation

-- ARRAY_AGG
ARRAY_AGG(col)                                 -- array, NULLs included
ARRAY_AGG(col ORDER BY col)                    -- sorted array
ARRAY_AGG(DISTINCT col ORDER BY col)           -- deduplicated sorted array
ARRAY_AGG(col) FILTER (WHERE col IS NOT NULL)  -- exclude NULLs
(ARRAY_AGG(col ORDER BY x DESC))[1]            -- first element (top value)
(ARRAY_AGG(col ORDER BY x DESC))[1:3]          -- first 3 elements (slice)

-- Full query structure
SELECT col, COUNT(*), AVG(salary)
FROM employees
WHERE role != 'intern'            -- 1. filter rows
GROUP BY col                      -- 2. group
HAVING COUNT(*) >= 2              -- 3. filter groups
ORDER BY AVG(salary) DESC         -- 4. sort
LIMIT 5;                          -- 5. cap rows
```
