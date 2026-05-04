# Module 15: Built-in Functions — Strings, Dates, JSON, and Arrays

---

## Table of Contents

- [1. String Functions](#1-string-functions)
  - [CONCAT and String Concatenation](#concat-and-string-concatenation)
  - [SUBSTRING](#substring)
  - [UPPER and LOWER](#upper-and-lower)
  - [LENGTH](#length)
  - [REPLACE](#replace)
  - [SPLIT_PART](#split_part)
  - [REGEXP_REPLACE](#regexp_replace)
  - [Other Useful String Functions](#other-useful-string-functions)
- [2. Date/Time Functions](#2-datetime-functions)
  - [NOW and CURRENT_DATE](#now-and-current_date)
  - [EXTRACT](#extract)
  - [DATE_TRUNC](#date_trunc)
  - [AGE](#age)
  - [INTERVAL Arithmetic](#interval-arithmetic)
  - [Other Date Functions](#other-date-functions)
- [3. Type Casting](#3-type-casting)
  - [:: Operator (PostgreSQL Shorthand)](#-operator-postgresql-shorthand)
  - [CAST Function](#cast-function)
  - [Common Casts](#common-casts)
  - [Casting Edge Cases](#casting-edge-cases)
- [4. JSON and JSONB Basics](#4-json-and-jsonb-basics)
  - [Difference Between JSON and JSONB](#difference-between-json-and-jsonb)
  - [Accessing JSON Data](#accessing-json-data)
  - [Modifying JSONB](#modifying-jsonb)
  - [Querying JSONB](#querying-jsonb)
  - [Converting to/from JSON](#converting-tofrom-json)
- [5. Array Functions](#5-array-functions)
  - [ANY — Checking If Value Exists](#any--checking-if-value-exists)
  - [ALL — Comparing Against All Values](#all--comparing-against-all-values)
  - [UNNEST — Expand Array Into Rows](#unnest--expand-array-into-rows)
  - [Other Array Functions](#other-array-functions)
- [Quick Reference](#quick-reference)

---

## 1. String Functions

### CONCAT and String Concatenation

Concatenate multiple strings into one:

```sql
-- CONCAT function
SELECT CONCAT('Hello', ' ', 'World');  -- 'Hello World'

-- || operator (preferred in PostgreSQL)
SELECT 'Hello' || ' ' || 'World';      -- 'Hello World'

-- With NULL handling (CONCAT treats NULL as empty, || returns NULL if any part is NULL)
SELECT CONCAT('Hello', NULL, 'World');  -- 'HelloWorld'
SELECT 'Hello' || NULL || 'World';      -- NULL
SELECT COALESCE('Hello', '') || COALESCE(NULL, '') || COALESCE('World', '');  -- workaround
```

```sql
-- Build dynamic email from parts
SELECT
  CONCAT(LOWER(first_name), '.', LOWER(last_name), '@company.com') AS email
FROM employees;

-- Concatenate with type casting
SELECT 'User ID: ' || id || ', Age: ' || age FROM users;  -- works, auto-casts to text
```

### SUBSTRING

Extract part of a string:

```sql
SUBSTRING(string, start [, length])
SUBSTRING(string FROM start [FOR length])  -- SQL standard syntax
```

```sql
-- Extract characters from position (1-indexed)
SELECT SUBSTRING('PostgreSQL', 1, 8);      -- 'PostgreS'
SELECT SUBSTRING('PostgreSQL', 10);        -- 'QL' (from pos 10 to end)

-- SQL standard syntax
SELECT SUBSTRING('PostgreSQL' FROM 1 FOR 8);  -- 'PostgreS'
SELECT SUBSTRING('PostgreSQL' FROM 10);       -- 'QL'

-- Extract by pattern
SELECT SUBSTRING('2024-12-25', '\d{4}');  -- '2024' (regex)
SELECT SUBSTRING('Hello World', '\w+ (\w+)');  -- 'World' (second word in parens)
```

### UPPER and LOWER

Convert case:

```sql
SELECT UPPER('PostgreSQL');     -- 'POSTGRESQL'
SELECT LOWER('PostgreSQL');     -- 'postgresql'

-- Case-insensitive comparison (good for usernames/emails)
SELECT * FROM users WHERE LOWER(email) = LOWER('Alice@Example.COM');

-- Combine for title case (crude; see INITCAP)
SELECT UPPER(SUBSTRING(name, 1, 1)) || LOWER(SUBSTRING(name, 2)) AS title_case
FROM employees;
```

### LENGTH

String length (number of characters, not bytes):

```sql
SELECT LENGTH('PostgreSQL');    -- 10
SELECT LENGTH('🐘');            -- 1 (emoji counts as 1 character)
SELECT LENGTH('hello world');   -- 11 (includes space)
SELECT LENGTH('');              -- 0
SELECT LENGTH(NULL);            -- NULL

-- Find strings longer than 50 characters
SELECT * FROM articles WHERE LENGTH(body) > 50;
```

### REPLACE

Replace all occurrences of a substring:

```sql
SELECT REPLACE('hello world', 'world', 'PostgreSQL');  -- 'hello PostgreSQL'

SELECT REPLACE('abc abc abc', 'abc', 'xyz');           -- 'xyz xyz xyz'

-- Case matters
SELECT REPLACE('Hello World', 'hello', 'Hi');          -- 'Hello World' (no match, case differs)

-- Replace in a column
UPDATE users SET email = REPLACE(email, '@oldomain.com', '@newdomain.com');

-- Remove a substring (replace with empty string)
SELECT REPLACE('hello-world', '-', '');                -- 'helloworld'
```

### SPLIT_PART

Split a string and get the Nth part:

```sql
SPLIT_PART(string, delimiter, field_number)  -- 1-indexed
```

```sql
-- Split by comma
SELECT SPLIT_PART('one,two,three', ',', 1);  -- 'one'
SELECT SPLIT_PART('one,two,three', ',', 2);  -- 'two'
SELECT SPLIT_PART('one,two,three', ',', 3);  -- 'three'

-- Split by space
SELECT SPLIT_PART('John Doe Smith', ' ', 1);  -- 'John'
SELECT SPLIT_PART('John Doe Smith', ' ', 3);  -- 'Smith'

-- Extract domain from email
SELECT SPLIT_PART('alice@example.com', '@', 2);  -- 'example.com'

-- Extract IP octets
SELECT SPLIT_PART('192.168.1.1', '.', 1);  -- '192'
```

> **Edge Case — Out of bounds returns empty string:**
> ```sql
> SELECT SPLIT_PART('one,two,three', ',', 5);  -- '' (no 5th part)
> ```

### REGEXP_REPLACE

Replace using regex patterns:

```sql
REGEXP_REPLACE(string, pattern, replacement [, flags])
```

```sql
-- Replace all digits with 'X'
SELECT REGEXP_REPLACE('Call 555-1234', '\d+', 'X', 'g');  -- 'Call XXX-XXXX'

-- Replace first match only (no 'g' flag)
SELECT REGEXP_REPLACE('Call 555-1234', '\d+', 'X');       -- 'Call X-1234'

-- Capture groups: swap first and last name
SELECT REGEXP_REPLACE('Smith, John', '(\w+), (\w+)', '\2 \1');  -- 'John Smith'

-- Remove non-alphanumeric characters
SELECT REGEXP_REPLACE('hello-world_123!', '[^a-zA-Z0-9]', '', 'g');  -- 'helloworld123'

-- Mask sensitive data (keep first 2 and last 2 digits)
SELECT REGEXP_REPLACE('4532-1234-5678-9012', '(\d{2})\d+(\d{2})', '\1**-****-****-\2', 'g');
-- '45**-****-****-12'
```

### Other Useful String Functions

```sql
-- TRIM: remove leading/trailing whitespace
SELECT TRIM('  hello  ');                -- 'hello'
SELECT LTRIM('  hello  ');               -- 'hello  '
SELECT RTRIM('  hello  ');               -- '  hello'

-- LPAD / RPAD: pad to length
SELECT LPAD('5', 3, '0');                -- '005' (left-pad with zeros)
SELECT RPAD('hello', 10, '.');           -- 'hello.....' (right-pad with dots)

-- REPEAT: repeat string N times
SELECT REPEAT('ab', 3);                  -- 'ababab'

-- REVERSE: reverse a string
SELECT REVERSE('hello');                 -- 'olleh'

-- STARTS_WITH / ENDS_WITH
SELECT * FROM products WHERE name STARTS_WITH 'Post';    -- 'PostgreSQL', 'Postgres'...
SELECT * FROM files WHERE filename ENDS_WITH '.pdf';

-- POSITION: find first occurrence (1-indexed)
SELECT POSITION('world' IN 'hello world');  -- 7 (0 if not found)

-- INITCAP: capitalize first letter of each word
SELECT INITCAP('hello world');           -- 'Hello World'
```

---

## 2. Date/Time Functions

### NOW and CURRENT_DATE

Get the current date/time:

```sql
-- NOW / CURRENT_TIMESTAMP: full timestamp with timezone
SELECT NOW();                            -- 2024-12-25 14:30:45.123456+00:00
SELECT CURRENT_TIMESTAMP;                -- same as NOW()

-- CURRENT_DATE: date only
SELECT CURRENT_DATE;                     -- 2024-12-25

-- CURRENT_TIME: time only
SELECT CURRENT_TIME;                     -- 14:30:45.123456+00:00

-- All above are frozen within a transaction (consistent value)
-- Use CLOCK_TIMESTAMP() if you need live updates mid-transaction
SELECT CLOCK_TIMESTAMP();                -- updates every call
```

```sql
-- Typical usage: record when a row was modified
CREATE TABLE posts (
  id        SERIAL PRIMARY KEY,
  title     TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Update timestamps
UPDATE posts SET updated_at = NOW() WHERE id = 1;
```

### EXTRACT

Extract a component from a date/time:

```sql
EXTRACT(field FROM date_expr)
```

```sql
-- From TIMESTAMP
SELECT EXTRACT(YEAR   FROM NOW());       -- 2024
SELECT EXTRACT(MONTH  FROM NOW());       -- 12
SELECT EXTRACT(DAY    FROM NOW());       -- 25
SELECT EXTRACT(HOUR   FROM NOW());       -- 14
SELECT EXTRACT(MINUTE FROM NOW());       -- 30
SELECT EXTRACT(SECOND FROM NOW());       -- 45.123456

-- From DATE
SELECT EXTRACT(YEAR   FROM '2024-12-25');  -- 2024
SELECT EXTRACT(MONTH  FROM '2024-12-25');  -- 12
SELECT EXTRACT(QUARTER FROM '2024-12-25'); -- 4

-- Day of week (0=Sunday, 6=Saturday in ISO format)
SELECT EXTRACT(DOW FROM '2024-12-25');     -- 3 (Wednesday)

-- Day of year
SELECT EXTRACT(DOY FROM '2024-12-25');     -- 360

-- All as integers
SELECT
  EXTRACT(YEAR FROM NOW())::INT,
  EXTRACT(MONTH FROM NOW())::INT,
  EXTRACT(DAY FROM NOW())::INT;
```

### DATE_TRUNC

Truncate a date/time to a specific unit (set smaller units to 0):

```sql
DATE_TRUNC(unit, timestamp)
```

```sql
-- Truncate timestamp to start of various units
SELECT DATE_TRUNC('year',   '2024-06-15 14:30:45');    -- 2024-01-01 00:00:00
SELECT DATE_TRUNC('month',  '2024-06-15 14:30:45');    -- 2024-06-01 00:00:00
SELECT DATE_TRUNC('day',    '2024-06-15 14:30:45');    -- 2024-06-15 00:00:00
SELECT DATE_TRUNC('hour',   '2024-06-15 14:30:45');    -- 2024-06-15 14:00:00
SELECT DATE_TRUNC('minute', '2024-06-15 14:30:45');    -- 2024-06-15 14:30:00

-- Group orders by month
SELECT
  DATE_TRUNC('month', created_at) AS month,
  COUNT(*) AS order_count,
  SUM(total) AS revenue
FROM orders
GROUP BY DATE_TRUNC('month', created_at)
ORDER BY month;
```

### AGE

Calculate age between two timestamps:

```sql
AGE(timestamp1, timestamp2)  -- returns INTERVAL
AGE(timestamp)               -- equivalent to AGE(NOW(), timestamp)
```

```sql
-- Years, months, days between two dates
SELECT AGE('2000-01-01'::DATE);                        -- 24 years 11 mons 25 days
SELECT AGE('2024-12-25'::DATE, '2000-01-01'::DATE);   -- 24 years 11 mons 24 days

-- Employee age (years only)
SELECT name, EXTRACT(YEAR FROM AGE(birthdate)) AS age_years
FROM employees;

-- How long an employee has worked (in years)
SELECT name, EXTRACT(YEAR FROM AGE(hired_at)) AS years_employed
FROM employees;

-- Results as integer
SELECT EXTRACT(YEAR FROM AGE(birthdate))::INT AS age FROM employees;
```

### INTERVAL Arithmetic

Add/subtract time durations:

```sql
-- Dates + INTERVAL
SELECT CURRENT_DATE + INTERVAL '1 day';             -- tomorrow
SELECT CURRENT_DATE - INTERVAL '30 days';          -- 30 days ago
SELECT NOW() + INTERVAL '2 hours 30 minutes';      -- 2.5 hours from now
SELECT NOW() + INTERVAL '1 year 2 months 3 days';  -- complex interval

-- Shorthand
SELECT CURRENT_DATE + 1;                            -- tomorrow (integer = days)
SELECT CURRENT_DATE + '1 day'::INTERVAL;            -- same

-- Multiply intervals
SELECT INTERVAL '1 day' * 5;                        -- 5 days
SELECT INTERVAL '1 hour' * 24;                      -- 1 day

-- Extract duration components
SELECT EXTRACT(DAYS FROM INTERVAL '1 year 2 months 3 days');  -- 3 (days component only)

-- Common durations
INTERVAL '1 second'
INTERVAL '1 minute'
INTERVAL '1 hour'
INTERVAL '1 day'
INTERVAL '1 week'
INTERVAL '1 month'
INTERVAL '1 year'
```

```sql
-- Practical examples
-- Find events in the next 7 days
SELECT * FROM events WHERE event_date BETWEEN NOW() AND NOW() + '7 days';

-- Find old records (older than 90 days)
SELECT * FROM logs WHERE created_at < NOW() - '90 days'::INTERVAL;

-- Calculate session timeout (inactive for 30 minutes)
SELECT * FROM sessions WHERE last_activity < NOW() - INTERVAL '30 minutes';
```

### Other Date Functions

```sql
-- TO_DATE: parse string to date
SELECT TO_DATE('25-12-2024', 'DD-MM-YYYY');         -- 2024-12-25

-- TO_TIMESTAMP: parse string to timestamp
SELECT TO_TIMESTAMP('25-12-2024 14:30:45', 'DD-MM-YYYY HH24:MI:SS');

-- TO_CHAR: format date/timestamp as string
SELECT TO_CHAR(NOW(), 'YYYY-MM-DD HH24:MI:SS');    -- '2024-12-25 14:30:45'
SELECT TO_CHAR(NOW(), 'Mon DD, YYYY');             -- 'Dec 25, 2024'

-- GENERATE_SERIES: generate date range
SELECT * FROM GENERATE_SERIES('2024-01-01'::DATE, '2024-01-31'::DATE, '1 day');
-- Returns 31 rows, one per day in January

-- MAKE_DATE / MAKE_TIMESTAMP
SELECT MAKE_DATE(2024, 12, 25);                    -- 2024-12-25
SELECT MAKE_TIMESTAMP(2024, 12, 25, 14, 30, 45);  -- 2024-12-25 14:30:45
```

---

## 3. Type Casting

Converting values between types:

### :: Operator (PostgreSQL Shorthand)

PostgreSQL-specific syntax for casting:

```sql
value::target_type
```

```sql
-- Cast to different types
SELECT '123'::INTEGER;                   -- 123 (text to int)
SELECT 123::TEXT;                        -- '123' (int to text)
SELECT 123.45::INTEGER;                  -- 123 (truncates, not rounded)
SELECT '2024-12-25'::DATE;               -- 2024-12-25
SELECT 123::NUMERIC(5,2);                -- 123.00 (to decimal)
SELECT '3.5'::FLOAT;                     -- 3.5

-- Cast in WHERE clauses
SELECT * FROM products WHERE price::INTEGER > 100;  -- cast to int, then compare

-- Type casting with NULL (NULL remains NULL)
SELECT NULL::INTEGER;                    -- NULL
SELECT NULL::TEXT;                       -- NULL
```

### CAST Function

SQL standard syntax:

```sql
CAST(value AS target_type)
```

```sql
-- Same results as :: but more verbose
SELECT CAST('123' AS INTEGER);           -- 123
SELECT CAST(123.45 AS INTEGER);          -- 123
SELECT CAST(NOW() AS DATE);              -- today's date

-- Some people prefer CAST for readability
SELECT
  CAST(order_id AS TEXT) AS order_id_str,
  CAST(price * quantity AS NUMERIC(10,2)) AS line_total
FROM order_items;
```

### Common Casts

| Source | Target | Example | Result |
|---|---|---|---|
| TEXT | INTEGER | `'123'::INT` | 123 |
| TEXT | FLOAT | `'3.14'::FLOAT` | 3.14 |
| TEXT | DATE | `'2024-12-25'::DATE` | 2024-12-25 |
| TEXT | BOOLEAN | `'true'::BOOL` | true |
| INTEGER | TEXT | `123::TEXT` | '123' |
| INTEGER | FLOAT | `5::FLOAT` | 5.0 |
| FLOAT | INTEGER | `3.99::INT` | 3 (truncates) |
| DATE | TEXT | `'2024-12-25'::TEXT` | '2024-12-25' |
| TIMESTAMP | DATE | `NOW()::DATE` | today |
| TIMESTAMP | TEXT | `NOW()::TEXT` | '2024-12-25 14:30:45.123456+00:00' |
| ARRAY | TEXT | `ARRAY[1,2,3]::TEXT` | '{1,2,3}' |

### Casting Edge Cases

```sql
-- Invalid cast raises an error
SELECT 'hello'::INTEGER;
-- ERROR: invalid input syntax for integer: "hello"

-- Implicit casts (sometimes automatic)
SELECT 1 + 2.5;                          -- 3.5 (integer promoted to float automatically)
SELECT '5' || 10;                        -- '510' (10 cast to text automatically)

-- Integer division truncates (doesn't round)
SELECT 5 / 2;                            -- 2 (not 2.5)
SELECT 5.0 / 2;                          -- 2.5 (float division)
SELECT 5::FLOAT / 2;                     -- 2.5 (cast to float first)

-- NULL casts remain NULL
SELECT NULL::INTEGER;                    -- NULL
SELECT NULL::TEXT;                       -- NULL
```

---

## 4. JSON and JSONB Basics

### Difference Between JSON and JSONB

| | JSON | JSONB |
|---|---|---|
| Storage | Stored as text | Stored as binary |
| Processing | Re-parsed on each read | Pre-parsed (faster) |
| Indexing | No B-tree index support | Supports GIN/GIST indexes |
| Performance | Slower for large documents | Faster |
| Whitespace | Preserved | Normalized (stripped) |
| Key order | Preserved | Sorted, deduplicates keys |
| Recommendation | Rarely use | **Always prefer JSONB** |

```sql
-- In practice, always use JSONB
CREATE TABLE products (
  id       SERIAL PRIMARY KEY,
  name     TEXT NOT NULL,
  metadata JSONB     -- not JSON
);
```

### Accessing JSON Data

Use `->` (returns JSON) or `->>` (returns TEXT):

```sql
INSERT INTO products (name, metadata) VALUES
  ('Laptop', '{"brand": "Dell", "specs": {"ram": 16, "cpu": "Intel i7"}, "price": 999.99}');

-- -> returns JSON
SELECT metadata -> 'brand' AS brand_json FROM products;
-- Result: "Dell" (with quotes, as JSON string)

-- ->> returns TEXT
SELECT metadata ->> 'brand' AS brand_text FROM products;
-- Result: Dell (without quotes, as text)

-- Nested access
SELECT metadata -> 'specs' -> 'ram' AS ram_json FROM products;     -- 16 (as JSON number)
SELECT metadata -> 'specs' ->> 'ram' AS ram_text FROM products;    -- "16" (as TEXT)
SELECT (metadata -> 'specs' ->> 'ram')::INTEGER AS ram_int FROM products;  -- 16 (cast to int)

-- Path notation
SELECT metadata #> '{specs, ram}' AS ram_via_path FROM products;   -- 16 (JSON)
SELECT metadata #>> '{specs, ram}' AS ram_via_path FROM products;  -- "16" (TEXT)

-- Array indexing (0-based)
INSERT INTO products (name, metadata) VALUES
  ('Mouse', '{"brands": ["Logitech", "Razer", "SteelSeries"]}');

SELECT metadata -> 'brands' -> 0 AS first_brand FROM products WHERE name = 'Mouse';  -- "Logitech"
SELECT metadata -> 'brands' ->> 1 AS second_brand FROM products WHERE name = 'Mouse'; -- "Razer"
```

### Modifying JSONB

```sql
-- jsonb_set: update a value
SELECT jsonb_set(
  '{"brand": "Dell", "price": 999.99}'::JSONB,
  '{price}',                    -- path
  '899.99'::JSONB              -- new value
);
-- Result: {"brand": "Dell", "price": 899.99}

-- Add a new key
SELECT jsonb_set(
  '{"brand": "Dell"}'::JSONB,
  '{color}',
  '"silver"'::JSONB
);
-- Result: {"brand": "Dell", "color": "silver"}

-- || operator: merge JSON objects
SELECT '{"a": 1}'::JSONB || '{"b": 2}'::JSONB;
-- Result: {"a": 1, "b": 2}

-- Merge with override
SELECT '{"a": 1, "b": 2}'::JSONB || '{"b": 3}'::JSONB;
-- Result: {"a": 1, "b": 3}  (b is overridden)

-- Delete a key
SELECT '{"name": "Alice", "age": 30}'::JSONB - 'age';
-- Result: {"name": "Alice"}

-- Delete a path
SELECT '{"user": {"name": "Alice", "age": 30}}'::JSONB #- '{user, age}';
-- Result: {"user": {"name": "Alice"}}
```

### Querying JSONB

```sql
-- ? : key exists
SELECT * FROM products WHERE metadata ? 'brand';

-- ?| : any of these keys exist
SELECT * FROM products WHERE metadata ?| ARRAY['brand', 'model'];

-- ?& : all these keys exist
SELECT * FROM products WHERE metadata ?& ARRAY['brand', 'price'];

-- @> : left contains right (subset check)
SELECT * FROM products WHERE metadata @> '{"brand": "Dell"}';

-- <@ : left is contained in right
SELECT * FROM products WHERE '{"brand": "Dell"}' <@ metadata;

-- @> with nested checks
SELECT * FROM products WHERE metadata @> '{"specs": {"ram": 16}}';

-- jsonb_exists: function form of ?
SELECT jsonb_exists(metadata, 'brand') FROM products;

-- jsonb_exists_any: function form of ?|
SELECT jsonb_exists_any(metadata, ARRAY['brand', 'model']) FROM products;

-- jsonb_keys: get all keys as array
SELECT jsonb_keys(metadata) FROM products;
-- Result: {brand, specs, price}

-- jsonb_each: expand into key-value pairs
SELECT key, value FROM jsonb_each(metadata) WHERE key = 'brand';
```

### Converting to/from JSON

```sql
-- ROW to JSON
SELECT ROW_TO_JSON(ROW('Alice', 30, 'alice@example.com')) AS user_json;
-- Result: {"f1":"Alice","f2":30,"f3":"alice@example.com"}

-- With column naming
SELECT ROW_TO_JSON(
  ROW(name, age, email) AS user
) AS user_json
FROM users;

-- Aggregate to JSON array
SELECT JSON_AGE(ROW_TO_JSON(t))
FROM (SELECT * FROM users LIMIT 3) t;
-- Result: [{"id":1,"name":"Alice",...}, {"id":2,...}, ...]

-- Convert array to JSON
SELECT ARRAY_TO_JSON(ARRAY[1,2,3]);     -- [1,2,3]

-- JSON_BUILD_OBJECT: construct JSON from key-value pairs
SELECT JSON_BUILD_OBJECT('name', 'Alice', 'age', 30, 'email', 'alice@example.com');
-- Result: {"name":"Alice","age":30,"email":"alice@example.com"}

-- JSON_BUILD_ARRAY: construct JSON array
SELECT JSON_BUILD_ARRAY('Alice', 30, true);
-- Result: ["Alice", 30, true]

-- JSONB_STRIP_NULLS: remove null values
SELECT JSONB_STRIP_NULLS('{"name": "Alice", "age": null, "email": "alice@example.com"}'::JSONB);
-- Result: {"name":"Alice","email":"alice@example.com"}
```

---

## 5. Array Functions

PostgreSQL arrays are denoted with `{}` and are 1-indexed.

### ANY — Checking If Value Exists

`ANY` checks if a value matches any element in an array:

```sql
value = ANY(array)
value <> ANY(array)
value > ANY(array)
```

```sql
-- Check if value is in array
SELECT * FROM products WHERE category = ANY(ARRAY['Electronics', 'Software']);

-- Equivalent to IN
SELECT * FROM products WHERE category IN ('Electronics', 'Software');

-- Use with column that contains an array
CREATE TABLE users (
  id       SERIAL PRIMARY KEY,
  name     TEXT NOT NULL,
  interests TEXT[]    -- array of interests
);

INSERT INTO users VALUES (1, 'Alice', ARRAY['hiking', 'cooking', 'photography']);

-- Check if user likes hiking
SELECT * FROM users WHERE 'hiking' = ANY(interests);

-- Check if user likes any of these topics
SELECT * FROM users WHERE 'hiking' = ANY(interests) OR 'cooking' = ANY(interests);

-- More readable with ANY
SELECT * FROM users WHERE ('hiking', 'cooking') OVERLAPS interests;  -- or use ANY loop
```

### ALL — Comparing Against All Values

`ALL` applies a comparison against every element:

```sql
value op ALL(array)    -- true if operator true for ALL elements
```

```sql
-- Value greater than all elements
SELECT 10 > ALL(ARRAY[1, 2, 3, 5]);     -- true

-- Value less than all elements
SELECT 1 < ALL(ARRAY[5, 10, 15]);       -- true

-- Practical: find products more expensive than all in the 'budget' category
SELECT * FROM products
WHERE price > ALL(
  SELECT price FROM products WHERE category = 'budget'
);
```

### UNNEST — Expand Array Into Rows

`UNNEST` converts an array into individual rows — the opposite of aggregation:

```sql
UNNEST(array) [AS alias(column_name)]
```

```sql
-- Expand array into rows
SELECT UNNEST(ARRAY['Alice', 'Bob', 'Charlie']);
-- Result:
--  unnest
-- --------
--  Alice
--  Bob
--  Charlie

-- With alias
SELECT name FROM UNNEST(ARRAY['Alice', 'Bob', 'Charlie']) AS names(name);

-- Unnest column array (each user's interests as separate rows)
SELECT id, name, interest
FROM users, UNNEST(interests) AS interest
WHERE name = 'Alice';
-- Result:
--  id | name  | interest
-- ----+-------+-------------
--   1 | Alice | hiking
--   1 | Alice | cooking
--   1 | Alice | photography

-- With ORDER BY and filtering
SELECT interest
FROM users, UNNEST(interests) AS interest
WHERE name = 'Alice'
ORDER BY interest;
```

### Other Array Functions

```sql
-- ARRAY_LENGTH: get array dimension length
SELECT ARRAY_LENGTH(ARRAY[1, 2, 3, 4, 5], 1);  -- 5

-- ARRAY_APPEND / ARRAY_PREPEND
SELECT ARRAY_APPEND(ARRAY[1, 2, 3], 4);        -- {1,2,3,4}
SELECT ARRAY_PREPEND(0, ARRAY[1, 2, 3]);       -- {0,1,2,3}

-- ARRAY_CAT: concatenate arrays
SELECT ARRAY_CAT(ARRAY[1, 2], ARRAY[3, 4]);    -- {1,2,3,4}

-- ARRAY_REMOVE: remove all matching values
SELECT ARRAY_REMOVE(ARRAY[1, 2, 3, 2, 1], 2);  -- {1,3,1}

-- ARRAY_DISTINCT: unique elements (overlaps)
SELECT ARRAY_AGG(DISTINCT category) FROM products;

-- @ and <@ : containment
SELECT ARRAY[1, 2, 3] @> ARRAY[2, 3];          -- true (contains)
SELECT ARRAY[2, 3] <@ ARRAY[1, 2, 3];          -- true (is contained)

-- && : overlaps
SELECT ARRAY[1, 2, 3] && ARRAY[3, 4, 5];       -- true (shares element 3)

-- ARRAY indexing (1-indexed, not 0)
SELECT ARRAY[10, 20, 30][1];                   -- 10
SELECT ARRAY[10, 20, 30][2];                   -- 20
SELECT ARRAY[10, 20, 30][-1];                  -- 30 (last element)

-- ARRAY slicing
SELECT ARRAY[10, 20, 30, 40, 50][2:4];         -- {20,30,40}
```

---

## Quick Reference

```sql
-- STRING FUNCTIONS
CONCAT(str1, str2, ...)                          -- concatenate
str1 || str2 || str3                             -- concatenate (preferred)
SUBSTRING(str, start [, length])                 -- extract part
UPPER(str) / LOWER(str)                          -- case conversion
LENGTH(str)                                      -- character count
REPLACE(str, old, new)                           -- replace all occurrences
SPLIT_PART(str, delimiter, field_number)        -- split and get Nth part
REGEXP_REPLACE(str, pattern, replacement, flags) -- regex replace
TRIM / LTRIM / RTRIM(str)                        -- remove whitespace
LPAD / RPAD(str, length, pad_char)               -- pad to length
REVERSE(str)                                     -- reverse string
INITCAP(str)                                     -- capitalize first letter of words

-- DATE/TIME FUNCTIONS
NOW() / CURRENT_TIMESTAMP                        -- current timestamp
CURRENT_DATE                                     -- today's date
EXTRACT(field FROM date_expr)                    -- extract year/month/day/hour/etc.
DATE_TRUNC(unit, timestamp)                      -- truncate to unit
AGE(timestamp1, timestamp2)                      -- age between two dates
INTERVAL 'value unit'                            -- duration (e.g., '7 days', '2 hours')
date + INTERVAL / date - INTERVAL                -- date arithmetic
TO_DATE(str, format)                             -- parse string to date
TO_TIMESTAMP(str, format)                        -- parse string to timestamp
TO_CHAR(date, format)                            -- format as string
MAKE_DATE(year, month, day)                      -- construct date
GENERATE_SERIES(start, end, step)                -- generate date range

-- TYPE CASTING
value::type                                      -- PostgreSQL shorthand (preferred)
CAST(value AS type)                              -- SQL standard
'123'::INTEGER                                   -- text to integer
123::TEXT                                        -- integer to text
'2024-12-25'::DATE                               -- text to date
NOW()::DATE                                      -- timestamp to date

-- JSON / JSONB
json_col -> 'key'                                -- extract as JSON
json_col ->> 'key'                               -- extract as TEXT
json_col #> '{key1, key2}'                       -- extract nested as JSON
json_col #>> '{key1, key2}'                      -- extract nested as TEXT
jsonb_set(obj, '{path}', value)                  -- update value
jsonb_obj || jsonb_obj2                          -- merge
jsonb_obj - 'key'                                -- delete key
jsonb_col ? 'key'                                -- key exists
jsonb_col @> '{"key": "value"}'                  -- contains
jsonb_keys(obj)                                  -- all keys
jsonb_each(obj)                                  -- key-value pairs
ROW_TO_JSON(row)                                 -- convert row to JSON
JSON_BUILD_OBJECT('key1', val1, 'key2', val2)   -- construct JSON

-- ARRAYS
ARRAY[val1, val2, val3]                          -- create array
value = ANY(array)                               -- value in array
value > ALL(array)                               -- value greater than all
UNNEST(array)                                    -- expand to rows
ARRAY_LENGTH(arr, dimension)                     -- array size
ARRAY_APPEND(arr, val)                           -- add element
ARRAY_CAT(arr1, arr2)                            -- concatenate
arr @> arr2                                      -- contains
arr && arr2                                      -- overlaps
arr[1]                                           -- index (1-indexed)
arr[1:3]                                         -- slice
```
