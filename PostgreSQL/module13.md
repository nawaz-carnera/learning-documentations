# Module 13: Window Functions — Calculations Across Related Rows

---

## Table of Contents

- [1. OVER Clause Basics](#1-over-clause-basics)
  - [What a Window Function Is](#what-a-window-function-is)
  - [Anatomy of OVER](#anatomy-of-over)
  - [Named Windows (WINDOW Clause)](#named-windows-window-clause)
- [2. ROW_NUMBER](#2-row_number)
- [3. RANK and DENSE_RANK](#3-rank-and-dense_rank)
  - [RANK](#rank)
  - [DENSE_RANK](#dense_rank)
  - [ROW_NUMBER vs RANK vs DENSE_RANK](#row_number-vs-rank-vs-dense_rank)
- [4. PARTITION BY](#4-partition-by)
- [5. LAG and LEAD](#5-lag-and-lead)
  - [LAG — Look Back](#lag--look-back)
  - [LEAD — Look Forward](#lead--look-forward)
  - [Default Values and Offset](#default-values-and-offset)
- [6. FIRST_VALUE and LAST_VALUE](#6-first_value-and-last_value)
  - [FIRST_VALUE](#first_value)
  - [LAST_VALUE and the Frame Trap](#last_value-and-the-frame-trap)
  - [NTH_VALUE](#nth_value)
- [7. Running Totals (SUM with OVER)](#7-running-totals-sum-with-over)
  - [Cumulative SUM](#cumulative-sum)
  - [Running AVG, COUNT, MIN, MAX](#running-avg-count-min-max)
  - [Window Frames Explained](#window-frames-explained)
  - [Moving Averages](#moving-averages)
- [8. Difference from GROUP BY](#8-difference-from-group-by)
- [Quick Reference](#quick-reference)

---

## 1. OVER Clause Basics

### What a Window Function Is

A **window function** performs a calculation across a set of rows that are related to the current row — without collapsing those rows into a single output row the way `GROUP BY` does.

```
Regular aggregate (GROUP BY):     Window function (OVER):
────────────────────────────────   ────────────────────────────────────────
 dept    | avg_salary               name    | dept        | salary | dept_avg
─────────+───────────              ─────────+─────────────+────────+---------
 Eng     | 92000                    Alice   | Engineering | 120000 | 92000
 HR      | 60667                    Bob     | Engineering |  65000 | 92000
 Mktg    | 80000                    Charlie | Engineering |  91000 | 92000
                                    Dana    | HR          |  80000 | 60667
 3 rows (departments collapsed)     Eve     | HR          |  52000 | 60667
                                    Frank   | Marketing   |  80000 | 80000
                                    10 rows (all rows kept)
```

The key insight: **window functions return one output row for every input row**. They add information without removing rows.

### Anatomy of OVER

```sql
window_function() OVER (
  [PARTITION BY partition_cols]   -- divide rows into groups (like GROUP BY, but doesn't collapse)
  [ORDER BY sort_cols]            -- define row order within each partition
  [frame_clause]                  -- define which rows relative to current are in the "window"
)
```

- **`PARTITION BY`** — splits the dataset into independent groups. The function resets for each group.
- **`ORDER BY`** — defines the row sequence within each partition. Required for ranking and running totals.
- **`frame_clause`** — defines the sliding window of rows relative to the current row (covered in section 7).
- All three are optional. An empty `OVER ()` applies the function across the entire result set.

### Setup (used throughout this module)

```sql
CREATE TABLE employees (
  id         SERIAL PRIMARY KEY,
  name       TEXT NOT NULL,
  department TEXT NOT NULL,
  role       TEXT NOT NULL,
  salary     NUMERIC(10,2) NOT NULL,
  hired_at   DATE NOT NULL
);

INSERT INTO employees (name, department, role, salary, hired_at) VALUES
  ('Alice',   'Engineering', 'senior',  120000, '2019-03-01'),
  ('Bob',     'Engineering', 'junior',   65000, '2022-07-15'),
  ('Charlie', 'Engineering', 'senior',   91000, '2020-11-20'),
  ('Dana',    'HR',          'manager',  80000, '2018-06-10'),
  ('Eve',     'HR',          'junior',   52000, '2023-01-05'),
  ('Frank',   'HR',          'junior',   50000, '2023-04-18'),
  ('Grace',   'Marketing',   'manager',  88000, '2019-09-01'),
  ('Hank',    'Marketing',   'senior',   75000, '2021-02-14'),
  ('Ivy',     'Marketing',   'junior',   57000, '2022-10-30'),
  ('Jay',     'Engineering', 'manager',  95000, '2017-05-22');

CREATE TABLE monthly_sales (
  id         SERIAL PRIMARY KEY,
  rep_name   TEXT NOT NULL,
  region     TEXT NOT NULL,
  sale_month DATE NOT NULL,
  revenue    NUMERIC(12,2) NOT NULL
);

INSERT INTO monthly_sales (rep_name, region, sale_month, revenue) VALUES
  ('Alice', 'East', '2024-01-01', 12000),
  ('Alice', 'East', '2024-02-01', 15500),
  ('Alice', 'East', '2024-03-01',  9800),
  ('Alice', 'East', '2024-04-01', 17200),
  ('Bob',   'West', '2024-01-01',  8500),
  ('Bob',   'West', '2024-02-01', 11000),
  ('Bob',   'West', '2024-03-01', 13400),
  ('Bob',   'West', '2024-04-01',  9600),
  ('Carol', 'East', '2024-01-01', 19000),
  ('Carol', 'East', '2024-02-01', 21500),
  ('Carol', 'East', '2024-03-01', 18300),
  ('Carol', 'East', '2024-04-01', 22100);
```

### Named Windows (WINDOW Clause)

When the same `OVER (...)` definition is used multiple times, extract it with a `WINDOW` clause:

```sql
-- Repetitive: same OVER definition written three times
SELECT
  name, department, salary,
  RANK()    OVER (PARTITION BY department ORDER BY salary DESC),
  ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC),
  AVG(salary) OVER (PARTITION BY department ORDER BY salary DESC)
FROM employees;

-- Clean: define the window once and reference by name
SELECT
  name, department, salary,
  RANK()       OVER dept_window,
  ROW_NUMBER() OVER dept_window,
  AVG(salary)  OVER dept_window
FROM employees
WINDOW dept_window AS (PARTITION BY department ORDER BY salary DESC);
```

> **Edge Case — Window functions cannot be used in WHERE or HAVING:** They are computed after filtering, so you must wrap them in a subquery or CTE to filter on their results:
> ```sql
> -- ERROR: window functions not allowed in WHERE
> SELECT name, RANK() OVER (ORDER BY salary DESC) AS rnk
> FROM employees
> WHERE RANK() OVER (ORDER BY salary DESC) <= 3;
>
> -- Fix: wrap in CTE or subquery
> WITH ranked AS (
>   SELECT name, RANK() OVER (ORDER BY salary DESC) AS rnk
>   FROM employees
> )
> SELECT * FROM ranked WHERE rnk <= 3;
> ```

---

## 2. ROW_NUMBER

`ROW_NUMBER()` assigns a unique sequential integer to each row within a partition, starting at 1. No ties — every row gets a different number.

```sql
ROW_NUMBER() OVER ([PARTITION BY cols] ORDER BY cols)
```

### Basic Usage

```sql
-- Number every employee by salary (highest = 1)
SELECT
  name, department, salary,
  ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
FROM employees;
```
```
 name    | department  | salary    | rn
---------+-------------+-----------+----
 Alice   | Engineering | 120000.00 |  1
 Jay     | Engineering |  95000.00 |  2
 Charlie | Engineering |  91000.00 |  3
 Grace   | Marketing   |  88000.00 |  4
 Dana    | HR          |  80000.00 |  5
 Hank    | Marketing   |  75000.00 |  6
 Bob     | Engineering |  65000.00 |  7
 Ivy     | Marketing   |  57000.00 |  8
 Eve     | HR          |  52000.00 |  9
 Frank   | HR          |  50000.00 | 10
```

### With PARTITION BY — Reset per Group

```sql
-- Number employees within each department by salary
SELECT
  name, department, salary,
  ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees
ORDER BY department, dept_rank;
```
```
 name    | department  | salary    | dept_rank
---------+-------------+-----------+-----------
 Alice   | Engineering | 120000.00 |         1
 Jay     | Engineering |  95000.00 |         2
 Charlie | Engineering |  91000.00 |         3
 Bob     | Engineering |  65000.00 |         4
 Dana    | HR          |  80000.00 |         1   ← resets for HR
 Eve     | HR          |  52000.00 |         2
 Frank   | HR          |  50000.00 |         3
 Grace   | Marketing   |  88000.00 |         1   ← resets for Marketing
 Hank    | Marketing   |  75000.00 |         2
 Ivy     | Marketing   |  57000.00 |         3
```

### Practical Patterns

```sql
-- Get the top earner per department (most common ROW_NUMBER use case)
WITH ranked AS (
  SELECT
    name, department, salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
  FROM employees
)
SELECT name, department, salary
FROM ranked
WHERE rn = 1;
```
```
 name  | department  | salary
-------+-------------+-----------
 Alice | Engineering | 120000.00
 Dana  | HR          |  80000.00
 Grace | Marketing   |  88000.00
```

```sql
-- Get the 3 most recent orders per customer
WITH ranked_orders AS (
  SELECT
    customer_id, id AS order_id, total, created_at,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
  FROM orders
)
SELECT customer_id, order_id, total, created_at
FROM ranked_orders
WHERE rn <= 3;
```

```sql
-- Deduplicate: keep only the latest row per user (remove duplicates)
WITH dupes AS (
  SELECT *,
    ROW_NUMBER() OVER (PARTITION BY email ORDER BY created_at DESC) AS rn
  FROM users
)
DELETE FROM users
WHERE id IN (SELECT id FROM dupes WHERE rn > 1);
```

> **Edge Case — ROW_NUMBER is non-deterministic when ORDER BY has ties:** If two rows have identical salary and no tiebreaker, PostgreSQL may assign row numbers in any order between those rows. Always add a unique column (like `id`) as a tiebreaker for deterministic results:
> ```sql
> ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC, id ASC)
> -- id as tiebreaker ensures stable, reproducible ordering
> ```

---

## 3. RANK and DENSE_RANK

Both assign rank numbers based on ordering, but handle **ties** differently from `ROW_NUMBER`.

### RANK

`RANK()` gives tied rows the **same rank**, then skips the next rank(s). Like Olympic medals: two golds means no silver.

```sql
SELECT
  name, department, salary,
  RANK() OVER (ORDER BY salary DESC) AS rnk
FROM employees;
```

If Alice (120000) and a new employee also had 120000:
```
 name     | salary    | rnk
----------+-----------+-----
 Alice    | 120000.00 |   1
 NewHire  | 120000.00 |   1   ← same rank (tied)
 Jay      |  95000.00 |   3   ← skips 2 (gap after tie)
 Charlie  |  91000.00 |   4
```

### DENSE_RANK

`DENSE_RANK()` gives tied rows the **same rank** but does **not** skip the next rank — numbers are consecutive.

```sql
SELECT
  name, salary,
  RANK()       OVER (ORDER BY salary DESC) AS rnk,
  DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rnk
FROM employees;
```

```
 name    | salary    | rnk | dense_rnk
---------+-----------+-----+-----------
 Alice   | 120000.00 |   1 |         1
 Jay     |  95000.00 |   2 |         2
 Charlie |  91000.00 |   3 |         3
 Grace   |  88000.00 |   4 |         4
 Dana    |  80000.00 |   5 |         5
 Hank    |  75000.00 |   6 |         6
 Bob     |  65000.00 |   7 |         7
 Ivy     |  57000.00 |   8 |         8
 Eve     |  52000.00 |   9 |         9
 Frank   |  50000.00 |  10 |        10
```
No ties here, so they're identical. Let's force a tie to see the difference:

```sql
-- With tied salaries (Eve and Frank both earn 51000):
-- salary | RANK | DENSE_RANK | ROW_NUMBER
-- 52000  |   1  |     1      |     1
-- 51000  |   2  |     2      |     2      ← tie
-- 51000  |   2  |     2      |     3      ← tie
-- 50000  |   4  |     3      |     4      ← RANK skips to 4, DENSE_RANK goes to 3
```

### ROW_NUMBER vs RANK vs DENSE_RANK

| Function | Ties | Gap After Tie | Always Unique |
|---|---|---|---|
| `ROW_NUMBER` | Different numbers for ties | No gap | Yes |
| `RANK` | Same number for ties | Yes — skips | No |
| `DENSE_RANK` | Same number for ties | No gap | No |

```
Salary:      120000  95000  91000  91000  80000
ROW_NUMBER:       1      2      3      4      5   ← always unique, arbitrary tie order
RANK:             1      2      3      3      5   ← tied = same, then skips 4
DENSE_RANK:       1      2      3      3      4   ← tied = same, no gap
```

**When to use which:**
- `ROW_NUMBER` — deduplicate, paginate, get exactly the Nth row
- `RANK` — "2nd place" competitions where ties matter and gaps are expected
- `DENSE_RANK` — "top 3 salary bands" where you want the 3rd distinct tier, not the 3rd row

```sql
-- Use DENSE_RANK to get the top 3 salary tiers per department
-- (not top 3 employees — top 3 distinct salary levels)
WITH ranked AS (
  SELECT name, department, salary,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS tier
  FROM employees
)
SELECT name, department, salary, tier
FROM ranked
WHERE tier <= 3;
```

---

## 4. PARTITION BY

`PARTITION BY` divides the dataset into independent subsets. The window function is applied separately within each partition — resetting for each group.

```sql
-- Without PARTITION BY: window spans the entire table
SELECT name, salary,
  AVG(salary) OVER () AS company_avg
FROM employees;

-- With PARTITION BY: window spans only rows in the same department
SELECT name, department, salary,
  AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees;
```

### Multi-Column Partitioning

```sql
-- Rank within department AND role
SELECT
  name, department, role, salary,
  RANK() OVER (PARTITION BY department, role ORDER BY salary DESC) AS role_rank
FROM employees
ORDER BY department, role, role_rank;
```
```
 name    | department  | role    | salary    | role_rank
---------+-------------+---------+-----------+-----------
 Alice   | Engineering | senior  | 120000.00 |         1
 Charlie | Engineering | senior  |  91000.00 |         2
 Bob     | Engineering | junior  |  65000.00 |         1   ← junior partition resets
 Jay     | Engineering | manager |  95000.00 |         1   ← manager partition resets
 Dana    | HR          | manager |  80000.00 |         1
 Eve     | HR          | junior  |  52000.00 |         1
 Frank   | HR          | junior  |  50000.00 |         2
 Grace   | Marketing   | manager |  88000.00 |         1
 Hank    | Marketing   | senior  |  75000.00 |         1
 Ivy     | Marketing   | junior  |  57000.00 |         1
```

### Comparing Each Row to Its Group

```sql
-- Salary vs department average: who earns above/below average?
SELECT
  name,
  department,
  salary,
  ROUND(AVG(salary) OVER (PARTITION BY department), 2) AS dept_avg,
  salary - ROUND(AVG(salary) OVER (PARTITION BY department), 2) AS diff_from_avg,
  CASE
    WHEN salary > AVG(salary) OVER (PARTITION BY department) THEN 'above avg'
    WHEN salary < AVG(salary) OVER (PARTITION BY department) THEN 'below avg'
    ELSE 'at avg'
  END AS vs_avg
FROM employees
ORDER BY department, salary DESC;
```

```sql
-- Each employee's salary as % of department total
SELECT
  name, department, salary,
  ROUND(
    100.0 * salary / SUM(salary) OVER (PARTITION BY department), 1
  ) AS pct_of_dept_payroll
FROM employees
ORDER BY department, pct_of_dept_payroll DESC;
```

### PARTITION BY vs GROUP BY — Side by Side

```sql
-- GROUP BY collapses to one row per department
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department;
-- 3 rows returned

-- PARTITION BY keeps all rows, adds the average as a column
SELECT name, department, salary,
  AVG(salary) OVER (PARTITION BY department) AS avg_salary
FROM employees;
-- 10 rows returned, each knows its department average
```

---

## 5. LAG and LEAD

`LAG` and `LEAD` let you **look at another row's value from within the current row** — the previous or next row in the defined order.

```sql
LAG  (expr [, offset [, default]]) OVER ([PARTITION BY ...] ORDER BY ...)
LEAD (expr [, offset [, default]]) OVER ([PARTITION BY ...] ORDER BY ...)
```

- `offset` — how many rows back (`LAG`) or forward (`LEAD`) to look (default: 1)
- `default` — value to return when no such row exists (default: `NULL`)

### LAG — Look Back

```sql
-- Compare each month's revenue to the previous month
SELECT
  rep_name,
  sale_month,
  revenue,
  LAG(revenue) OVER (PARTITION BY rep_name ORDER BY sale_month) AS prev_revenue,
  revenue - LAG(revenue) OVER (PARTITION BY rep_name ORDER BY sale_month) AS change,
  ROUND(
    100.0 * (revenue - LAG(revenue) OVER (PARTITION BY rep_name ORDER BY sale_month))
    / NULLIF(LAG(revenue) OVER (PARTITION BY rep_name ORDER BY sale_month), 0),
    1
  ) AS pct_change
FROM monthly_sales
ORDER BY rep_name, sale_month;
```
```
 rep_name | sale_month | revenue  | prev_revenue | change  | pct_change
----------+------------+----------+--------------+---------+-----------
 Alice    | 2024-01-01 | 12000.00 |         NULL |    NULL |       NULL  ← no prior month
 Alice    | 2024-02-01 | 15500.00 |     12000.00 | 3500.00 |       29.2
 Alice    | 2024-03-01 |  9800.00 |     15500.00 |-5700.00 |      -36.8
 Alice    | 2024-04-01 | 17200.00 |      9800.00 | 7400.00 |       75.5
 Bob      | 2024-01-01 |  8500.00 |         NULL |    NULL |       NULL  ← partition resets
 Bob      | 2024-02-01 | 11000.00 |      8500.00 | 2500.00 |       29.4
 ...
```

### LEAD — Look Forward

```sql
-- Show each month's revenue alongside the next month's target
SELECT
  rep_name,
  sale_month,
  revenue,
  LEAD(revenue) OVER (PARTITION BY rep_name ORDER BY sale_month) AS next_month_revenue,
  LEAD(revenue) OVER (PARTITION BY rep_name ORDER BY sale_month) - revenue AS projected_change
FROM monthly_sales
ORDER BY rep_name, sale_month;
```
```
 rep_name | sale_month | revenue  | next_month_revenue | projected_change
----------+------------+----------+--------------------+------------------
 Alice    | 2024-01-01 | 12000.00 |          15500.00  |          3500.00
 Alice    | 2024-02-01 | 15500.00 |           9800.00  |         -5700.00
 Alice    | 2024-03-01 |  9800.00 |          17200.00  |          7400.00
 Alice    | 2024-04-01 | 17200.00 |               NULL |              NULL ← no next month
```

### Default Values and Offset

```sql
-- LAG with offset=2 (look back 2 rows) and default=0 (instead of NULL)
SELECT
  rep_name, sale_month, revenue,
  LAG(revenue, 1, 0) OVER (PARTITION BY rep_name ORDER BY sale_month) AS prev_1,
  LAG(revenue, 2, 0) OVER (PARTITION BY rep_name ORDER BY sale_month) AS prev_2
FROM monthly_sales
WHERE rep_name = 'Alice'
ORDER BY sale_month;
```
```
 rep_name | sale_month | revenue  | prev_1   | prev_2
----------+------------+----------+----------+---------
 Alice    | 2024-01-01 | 12000.00 |     0.00 |    0.00  ← defaults (no prior rows)
 Alice    | 2024-02-01 | 15500.00 | 12000.00 |    0.00  ← prev_2 still has no row
 Alice    | 2024-03-01 |  9800.00 | 15500.00 | 12000.00
 Alice    | 2024-04-01 | 17200.00 |  9800.00 | 15500.00
```

### Practical Patterns

```sql
-- Detect first and last record per group (no prior/next = boundary row)
SELECT
  rep_name, sale_month, revenue,
  CASE WHEN LAG(sale_month)  OVER (PARTITION BY rep_name ORDER BY sale_month) IS NULL
       THEN 'first month' END AS is_first,
  CASE WHEN LEAD(sale_month) OVER (PARTITION BY rep_name ORDER BY sale_month) IS NULL
       THEN 'last month'  END AS is_last
FROM monthly_sales;

-- Identify consecutive row gaps (missing months)
SELECT
  rep_name, sale_month,
  LAG(sale_month) OVER (PARTITION BY rep_name ORDER BY sale_month) AS prev_month,
  sale_month - LAG(sale_month) OVER (PARTITION BY rep_name ORDER BY sale_month) AS days_gap
FROM monthly_sales
ORDER BY rep_name, sale_month;
```

> **Edge Case — LAG/LEAD ORDER BY is mandatory:** Without `ORDER BY`, "previous" and "next" row are undefined — results are non-deterministic. Always include `ORDER BY` inside `OVER`.

> **Edge Case — LAG/LEAD across partitions:** LAG and LEAD respect partition boundaries. The first row in each partition has no previous row (`LAG` returns `NULL` or the default), and the last row has no next row (`LEAD` returns `NULL` or the default). They never "look across" into another partition.

---

## 6. FIRST_VALUE and LAST_VALUE

Return the value of an expression from the **first or last row** in the window frame.

```sql
FIRST_VALUE(expr) OVER ([PARTITION BY ...] ORDER BY ... [frame])
LAST_VALUE(expr)  OVER ([PARTITION BY ...] ORDER BY ... [frame])
```

### FIRST_VALUE

```sql
-- Show each employee's salary and the highest earner in their department
SELECT
  name, department, salary,
  FIRST_VALUE(name)   OVER (PARTITION BY department ORDER BY salary DESC) AS top_earner,
  FIRST_VALUE(salary) OVER (PARTITION BY department ORDER BY salary DESC) AS top_salary
FROM employees
ORDER BY department, salary DESC;
```
```
 name    | department  | salary    | top_earner | top_salary
---------+-------------+-----------+------------+-----------
 Alice   | Engineering | 120000.00 | Alice      | 120000.00
 Jay     | Engineering |  95000.00 | Alice      | 120000.00
 Charlie | Engineering |  91000.00 | Alice      | 120000.00
 Bob     | Engineering |  65000.00 | Alice      | 120000.00
 Dana    | HR          |  80000.00 | Dana       |  80000.00
 Eve     | HR          |  52000.00 | Dana       |  80000.00
 Frank   | HR          |  50000.00 | Dana       |  80000.00
```

### LAST_VALUE and the Frame Trap

`LAST_VALUE` has a critical gotcha. The default window frame is `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — meaning the "window" only extends from the start up to the current row. So `LAST_VALUE` returns the **current row's value**, not the actual last row in the partition.

```sql
-- WRONG: LAST_VALUE returns current row's value, not partition's last row
SELECT
  name, department, salary,
  LAST_VALUE(salary) OVER (PARTITION BY department ORDER BY salary DESC) AS lowest_salary
FROM employees;
-- "lowest_salary" just equals "salary" for each row — useless!
```

```sql
-- CORRECT: extend the frame to include all rows in the partition
SELECT
  name, department, salary,
  LAST_VALUE(salary) OVER (
    PARTITION BY department
    ORDER BY salary DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING  -- ← fix
  ) AS lowest_salary
FROM employees
ORDER BY department, salary DESC;
```
```
 name    | department  | salary    | lowest_salary
---------+-------------+-----------+--------------
 Alice   | Engineering | 120000.00 |     65000.00  ← now correctly the lowest
 Jay     | Engineering |  95000.00 |     65000.00
 Charlie | Engineering |  91000.00 |     65000.00
 Bob     | Engineering |  65000.00 |     65000.00
```

> **`FIRST_VALUE` does not have this problem** because the frame always starts at the beginning — the first row is always within the default frame. Only `LAST_VALUE` (and `NTH_VALUE` for values near the end) requires the explicit frame extension.

### NTH_VALUE

Returns the value of the Nth row in the window frame:

```sql
-- The 2nd highest earner per department
SELECT DISTINCT
  department,
  FIRST_VALUE(name) OVER w AS top_1,
  NTH_VALUE(name, 2) OVER w AS top_2,
  NTH_VALUE(name, 3) OVER w AS top_3
FROM employees
WINDOW w AS (
  PARTITION BY department
  ORDER BY salary DESC
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
);
```
```
 department  | top_1 | top_2   | top_3
-------------+-------+---------+---------
 Engineering | Alice | Jay     | Charlie
 HR          | Dana  | Eve     | Frank
 Marketing   | Grace | Hank    | Ivy
```

---

## 7. Running Totals (SUM with OVER)

Aggregate functions (`SUM`, `AVG`, `COUNT`, `MIN`, `MAX`) become **window aggregates** when paired with `OVER`. Instead of collapsing rows, they compute a value relative to each row's window.

### Cumulative SUM

```sql
-- Running total of revenue per sales rep, month by month
SELECT
  rep_name,
  sale_month,
  revenue,
  SUM(revenue) OVER (
    PARTITION BY rep_name
    ORDER BY sale_month
  ) AS cumulative_revenue
FROM monthly_sales
ORDER BY rep_name, sale_month;
```
```
 rep_name | sale_month | revenue  | cumulative_revenue
----------+------------+----------+--------------------
 Alice    | 2024-01-01 | 12000.00 |           12000.00
 Alice    | 2024-02-01 | 15500.00 |           27500.00
 Alice    | 2024-03-01 |  9800.00 |           37300.00
 Alice    | 2024-04-01 | 17200.00 |           54500.00
 Bob      | 2024-01-01 |  8500.00 |            8500.00  ← resets for Bob
 Bob      | 2024-02-01 | 11000.00 |           19500.00
 ...
```

```sql
-- Cumulative count of employees hired over time
SELECT
  name, hired_at,
  COUNT(*) OVER (ORDER BY hired_at) AS employees_so_far
FROM employees
ORDER BY hired_at;
```

### Running AVG, COUNT, MIN, MAX

```sql
SELECT
  rep_name, sale_month, revenue,
  ROUND(AVG(revenue)   OVER (PARTITION BY rep_name ORDER BY sale_month), 2) AS running_avg,
  COUNT(*)             OVER (PARTITION BY rep_name ORDER BY sale_month)     AS months_counted,
  MIN(revenue)         OVER (PARTITION BY rep_name ORDER BY sale_month)     AS running_min,
  MAX(revenue)         OVER (PARTITION BY rep_name ORDER BY sale_month)     AS running_max
FROM monthly_sales
ORDER BY rep_name, sale_month;
```
```
 rep_name | sale_month | revenue  | running_avg | months_counted | running_min | running_max
----------+------------+----------+-------------+----------------+-------------+------------
 Alice    | 2024-01-01 | 12000.00 |    12000.00 |              1 |    12000.00 |   12000.00
 Alice    | 2024-02-01 | 15500.00 |    13750.00 |              2 |    12000.00 |   15500.00
 Alice    | 2024-03-01 |  9800.00 |    12433.33 |              3 |     9800.00 |   15500.00
 Alice    | 2024-04-01 | 17200.00 |    13625.00 |              4 |     9800.00 |   17200.00
```

### Window Frames Explained

The **frame clause** defines exactly which rows relative to the current row are included in the window calculation.

```sql
SUM(revenue) OVER (
  ORDER BY sale_month
  ROWS BETWEEN <start> AND <end>
)
```

**Frame boundaries:**

| Boundary | Meaning |
|---|---|
| `UNBOUNDED PRECEDING` | First row of the partition |
| `N PRECEDING` | N rows before the current row |
| `CURRENT ROW` | The current row |
| `N FOLLOWING` | N rows after the current row |
| `UNBOUNDED FOLLOWING` | Last row of the partition |

**Common frame patterns:**

```sql
-- Default when ORDER BY present (cumulative from start to current row):
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

-- Entire partition (same result for every row in partition):
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING

-- Previous row + current row + next row (3-row window):
ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING

-- Last 3 rows including current (3-row rolling window):
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
```

**ROWS vs RANGE:**

```sql
-- ROWS: counts physical rows (precise)
SUM(revenue) OVER (ORDER BY sale_month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)

-- RANGE: counts rows with the same ORDER BY value as a group (default for ORDER BY only)
SUM(revenue) OVER (ORDER BY sale_month RANGE BETWEEN 2 PRECEDING AND CURRENT ROW)
-- WARNING: RANGE with numeric/date offset behaves differently than ROWS
-- For most cases, use ROWS — it is more predictable
```

### Moving Averages

```sql
-- 3-month moving average (current + 2 prior months)
SELECT
  rep_name,
  sale_month,
  revenue,
  ROUND(
    AVG(revenue) OVER (
      PARTITION BY rep_name
      ORDER BY sale_month
      ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ), 2
  ) AS moving_avg_3m
FROM monthly_sales
ORDER BY rep_name, sale_month;
```
```
 rep_name | sale_month | revenue  | moving_avg_3m
----------+------------+----------+---------------
 Alice    | 2024-01-01 | 12000.00 |      12000.00  ← only 1 row available
 Alice    | 2024-02-01 | 15500.00 |      13750.00  ← 2 rows
 Alice    | 2024-03-01 |  9800.00 |      12433.33  ← 3 rows (full window)
 Alice    | 2024-04-01 | 17200.00 |      14166.67  ← slides: drops Jan, adds Apr
```

```sql
-- 7-day rolling sum (last 7 days of activity)
SELECT
  event_date,
  daily_count,
  SUM(daily_count) OVER (
    ORDER BY event_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) AS rolling_7d_sum
FROM daily_event_counts
ORDER BY event_date;
```

> **Edge Case — Running total with ties in ORDER BY:** When multiple rows share the same ORDER BY value, `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` (the default) includes all tied rows in the current frame — which can cause "jumps" in the running total. Use `ROWS` for strict row-by-row accumulation:
> ```sql
> -- Two sales on same day cause a jump with RANGE (default):
> SUM(revenue) OVER (ORDER BY sale_date)
> -- Row 1: sale_date=Jan 1, revenue=100, running_total=300 (both Jan 1 rows included!)
> -- Row 2: sale_date=Jan 1, revenue=200, running_total=300 (same value — confusing)
>
> -- Use ROWS to accumulate strictly one row at a time:
> SUM(revenue) OVER (ORDER BY sale_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
> -- Row 1: running_total=100
> -- Row 2: running_total=300
> ```

---

## 8. Difference from GROUP BY

This is the most important conceptual distinction with window functions.

### Core Difference

| | GROUP BY | Window Function (OVER) |
|---|---|---|
| Output rows | One per group | One per input row |
| Collapses rows | Yes | No |
| Can mix row + group data | No | Yes |
| Filter result | `HAVING` | Subquery / CTE |
| Access other rows | No | Yes (LAG, LEAD, FIRST_VALUE) |
| Can use alongside | Aggregates only | Any SELECT column |

### Same Question, Different Approach

```sql
-- QUESTION: What is each employee's salary and their department's average?

-- With GROUP BY: must choose one — you can't have both detail and aggregate
SELECT department, AVG(salary) AS dept_avg
FROM employees
GROUP BY department;
-- Only 3 rows (departments), individual employee rows lost

-- With window function: both in the same row
SELECT
  name,
  department,
  salary,
  AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees;
-- 10 rows — full detail preserved
```

### When You Cannot Use GROUP BY

```sql
-- Show running total — impossible with GROUP BY alone
SELECT
  sale_month, revenue,
  SUM(revenue) OVER (ORDER BY sale_month) AS running_total
FROM monthly_sales
WHERE rep_name = 'Alice';
-- Each row shows its contribution to the running total — GROUP BY can't do this

-- Compare to previous row — impossible with GROUP BY
SELECT name, salary,
  LAG(salary) OVER (ORDER BY salary DESC) AS next_lower_salary,
  salary - LAG(salary) OVER (ORDER BY salary DESC) AS gap
FROM employees;
-- GROUP BY has no concept of "previous row"

-- Rank within group while keeping all rows
WITH ranked AS (
  SELECT name, department, salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
  FROM employees
)
SELECT * FROM ranked WHERE dept_rank = 1;
-- Can't filter on RANK in the same query — needs a CTE/subquery
```

### Using Both Together

Window functions and `GROUP BY` can coexist — the `GROUP BY` runs first, then window functions apply to the aggregated rows:

```sql
-- Step 1: GROUP BY collapses to monthly totals per rep
-- Step 2: Window function computes running total across those totals
SELECT
  rep_name,
  sale_month,
  SUM(revenue)   AS monthly_total,                             -- GROUP BY aggregate
  SUM(SUM(revenue)) OVER (                                     -- window over grouped rows
    PARTITION BY rep_name
    ORDER BY sale_month
  ) AS cumulative_total
FROM monthly_sales
GROUP BY rep_name, sale_month
ORDER BY rep_name, sale_month;
```
```
 rep_name | sale_month | monthly_total | cumulative_total
----------+------------+---------------+-----------------
 Alice    | 2024-01-01 |      12000.00 |        12000.00
 Alice    | 2024-02-01 |      15500.00 |        27500.00
 Alice    | 2024-03-01 |       9800.00 |        37300.00
 Alice    | 2024-04-01 |      17200.00 |        54500.00
```

> Notice `SUM(SUM(revenue))` — the inner `SUM` is the GROUP BY aggregate, the outer `SUM(...) OVER (...)` is the window function. This nested syntax is valid and common in reporting queries.

### SQL Execution Order with Window Functions

```
FROM
  → JOIN
  → WHERE           ← cannot reference window functions here
  → GROUP BY        ← runs before window functions
  → HAVING          ← cannot reference window functions here
  → SELECT          ← window functions computed here
  → DISTINCT
  → ORDER BY        ← can reference window function aliases
  → LIMIT / OFFSET
```

Window functions are computed in the `SELECT` phase — after `WHERE`, `GROUP BY`, and `HAVING` have already filtered and grouped the data. This is why you need a CTE or subquery to filter on a window function's result.

---

## Quick Reference

```sql
-- OVER clause anatomy
fn() OVER (
  [PARTITION BY col1, col2]
  [ORDER BY col ASC/DESC]
  [ROWS BETWEEN <start> AND <end>]
)

-- Ranking functions
ROW_NUMBER() OVER (ORDER BY col)                    -- unique, no ties
RANK()       OVER (ORDER BY col)                    -- ties get same rank, gaps after
DENSE_RANK() OVER (ORDER BY col)                    -- ties get same rank, no gaps
NTILE(4)     OVER (ORDER BY col)                    -- divide into N buckets (quartiles)
PERCENT_RANK() OVER (ORDER BY col)                  -- 0.0 to 1.0 relative rank
CUME_DIST()    OVER (ORDER BY col)                  -- cumulative distribution

-- With PARTITION BY
RANK() OVER (PARTITION BY dept ORDER BY salary DESC)

-- LAG / LEAD
LAG(col)           OVER (ORDER BY col)              -- previous row value
LAG(col, 2)        OVER (ORDER BY col)              -- 2 rows back
LAG(col, 1, 0)     OVER (ORDER BY col)              -- 1 row back, default 0 if missing
LEAD(col)          OVER (ORDER BY col)              -- next row value
LEAD(col, 1, 0)    OVER (ORDER BY col)              -- default 0 if no next row

-- FIRST_VALUE / LAST_VALUE
FIRST_VALUE(col) OVER (PARTITION BY x ORDER BY y)
LAST_VALUE(col)  OVER (                             -- ← always needs explicit frame!
  PARTITION BY x ORDER BY y
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
NTH_VALUE(col, 2) OVER (
  PARTITION BY x ORDER BY y
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)

-- Running totals / aggregates
SUM(col)   OVER (ORDER BY date)                     -- cumulative sum
AVG(col)   OVER (ORDER BY date)                     -- running average
COUNT(*)   OVER (ORDER BY date)                     -- running count

-- Moving window (last 3 rows including current)
AVG(col) OVER (ORDER BY date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)

-- Entire partition (same value for every row in group)
SUM(col) OVER (PARTITION BY dept ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)

-- Named window (reuse definition)
SELECT fn1() OVER w, fn2() OVER w
FROM t
WINDOW w AS (PARTITION BY dept ORDER BY salary DESC);

-- Filter on window function result (must use CTE or subquery)
WITH ranked AS (
  SELECT *, RANK() OVER (PARTITION BY dept ORDER BY salary DESC) AS rnk
  FROM employees
)
SELECT * FROM ranked WHERE rnk = 1;

-- Key frame boundaries
UNBOUNDED PRECEDING   -- first row of partition
N PRECEDING           -- N rows before current
CURRENT ROW           -- current row
N FOLLOWING           -- N rows after current
UNBOUNDED FOLLOWING   -- last row of partition
```