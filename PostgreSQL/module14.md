# Module 14: Transactions — ACID and Data Consistency

---

## Table of Contents

- [1. What a Transaction Is (ACID)](#1-what-a-transaction-is-acid)
  - [ACID Properties](#acid-properties)
  - [Real-World Example](#real-world-example)
- [2. BEGIN, COMMIT, ROLLBACK](#2-begin-commit-rollback)
  - [Basic Transaction Flow](#basic-transaction-flow)
  - [Explicit Transactions](#explicit-transactions)
  - [Implicit Transactions](#implicit-transactions)
  - [Auto-commit Mode](#auto-commit-mode)
- [3. Savepoints](#3-savepoints)
  - [SAVEPOINT and ROLLBACK TO](#savepoint-and-rollback-to)
  - [Practical Use Case](#practical-use-case)
  - [Nested Transactions](#nested-transactions)
- [4. Why Transactions Matter (Atomic Operations)](#4-why-transactions-matter-atomic-operations)
  - [Atomicity Guarantee](#atomicity-guarantee)
  - [Data Integrity Without Transactions](#data-integrity-without-transactions)
- [5. Isolation Levels Overview](#5-isolation-levels-overview)
  - [READ UNCOMMITTED](#read-uncommitted)
  - [READ COMMITTED (Default)](#read-committed-default)
  - [REPEATABLE READ](#repeatable-read)
  - [SERIALIZABLE](#serializable)
  - [Isolation Levels Comparison Table](#isolation-levels-comparison-table)
- [6. Common Mistake — Forgetting COMMIT in psql](#6-common-mistake--forgetting-commit-in-psql)
- [Quick Reference](#quick-reference)

---

## 1. What a Transaction Is (ACID)

A **transaction** is a sequence of SQL statements that are executed as a single atomic unit. Either all statements succeed and the data is committed, or an error occurs and everything is rolled back — leaving the database unchanged.

### ACID Properties

ACID is the foundation of reliable databases:

#### **A — Atomicity**

An operation is **all-or-nothing**. The transaction either completes fully or is rolled back entirely — no partial results.

```
Transfer $100 from Alice to Bob:
  1. Deduct $100 from Alice
  2. Add $100 to Bob

Without atomicity: Step 1 succeeds, server crashes, Step 2 never runs → $100 disappears
With atomicity: Both steps run together; if either fails, both are rolled back → $100 stays with Alice
```

#### **C — Consistency**

The database moves from one valid state to another valid state. All constraints (PKs, FKs, CHECKs, NOT NULLs) are enforced.

```
Constraint: account balance >= 0

Transaction tries to transfer $100 but Alice only has $50:
  -> Fails before commit (violates constraint)
  -> Rolls back (Alice still has $50, Bob unchanged)
  -> Database remains consistent
```

#### **I — Isolation**

Concurrent transactions don't interfere with each other. Each transaction sees a consistent view of the data.

```
Without isolation:
  Transaction A: reads Alice balance = $500
  Transaction B: transfers $100 from Alice
  Transaction A: tries to transfer $300, based on stale balance of $500 → data corruption

With isolation:
  Transaction A sees a consistent snapshot (either before or after Transaction B, not in between)
```

#### **D — Durability**

Once a transaction is **committed**, the changes are permanent — even if the server crashes immediately after.

```
COMMIT → data written to disk (WAL log)
Server crashes → PostgreSQL recovers from log → committed data persists
Uncommitted data → rolled back on recovery
```

### Real-World Example

```sql
-- Bank transfer: withdraw from one account, deposit to another
BEGIN TRANSACTION;

UPDATE accounts SET balance = balance - 100 WHERE account_id = 'Alice';
UPDATE accounts SET balance = balance + 100 WHERE account_id = 'Bob';

COMMIT;
```

**What happens:**
- `BEGIN` — PostgreSQL starts tracking this unit of work
- Two UPDATEs — both execute (rows locked, changes buffered)
- `COMMIT` — both changes become permanent (durability: written to WAL + committed)
- If anything fails between BEGIN and COMMIT → `ROLLBACK` reverts both (atomicity)

---

## 2. BEGIN, COMMIT, ROLLBACK

### Basic Transaction Flow

```
BEGIN          ← start transaction
  SQL queries
  COMMIT       ← success: persist all changes
  or ROLLBACK  ← failure: discard all changes
```

### Setup

```sql
CREATE TABLE accounts (
  id       SERIAL PRIMARY KEY,
  name     TEXT NOT NULL,
  balance  NUMERIC(12,2) NOT NULL CHECK (balance >= 0)
);

INSERT INTO accounts (name, balance) VALUES
  ('Alice', 1000.00),
  ('Bob',    500.00),
  ('Charlie', 750.00);
```

### Explicit Transactions

```sql
-- Successful transaction
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
  UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob';
COMMIT;
-- Changes are now permanent

SELECT * FROM accounts;
-- Alice: 900, Bob: 600
```

```sql
-- Failed transaction — rolls back automatically
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
  UPDATE accounts SET balance = balance - 2000 WHERE name = 'Bob';
  -- ERROR: new row violates check constraint "accounts_balance_check"
ROLLBACK;  -- or PostgreSQL auto-rolls back
-- Alice's balance is unchanged (back to 900)

SELECT * FROM accounts;
-- Alice: 900, Bob: 600 (no change, transaction failed)
```

### Implicit Transactions

Without explicit `BEGIN`, PostgreSQL operates in **autocommit mode** — each statement is its own transaction, committed immediately.

```sql
-- Each statement is autocommitted instantly
UPDATE accounts SET balance = balance - 50 WHERE name = 'Alice';
-- Committed immediately, no way to undo

UPDATE accounts SET balance = balance + 50 WHERE name = 'Bob';
-- Committed immediately
```

> **Edge Case — Mixing explicit and implicit transactions:**
> ```sql
> BEGIN;
> UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
> -- No transaction active yet

> UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob';
> COMMIT;  -- Only the Bob update commits; Alice update already autocommitted
> ```
> This is why explicit `BEGIN` is critical for multi-statement operations.

### Auto-commit Mode

By default in psql, auto-commit is **ON** — each statement commits immediately unless you explicitly `BEGIN`.

```sql
-- Check auto-commit status (psql only)
\set
-- If "ON_ERROR_ROLLBACK" is "off", auto-commit is on

-- Disable auto-commit in psql (enter transaction mode)
\set ON_ERROR_ROLLBACK off
BEGIN;
  -- now in explicit transaction mode
  UPDATE ...;
  UPDATE ...;
COMMIT;

-- Re-enable auto-commit (back to autocommit mode)
\set ON_ERROR_ROLLBACK on
```

> **Most production applications use explicit transactions** via application drivers (Python, Ruby, Node, Java, etc.). The application calls `conn.begin()`, runs SQL, then calls `conn.commit()` or `conn.rollback()`. psql is mainly for one-off administrative queries where autocommit is fine.

---

## 3. Savepoints

A **savepoint** is a named marker within a transaction that you can roll back to — without rolling back the entire transaction.

```sql
BEGIN;
  statement_1;
  SAVEPOINT sp1;
    statement_2;
    statement_3;  -- fails
  ROLLBACK TO sp1;  -- undo statements 2-3, keep statement 1
  statement_4;      -- new attempt
COMMIT;  -- commits statement_1 and statement_4
```

### SAVEPOINT and ROLLBACK TO

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
-- Alice: 900

SAVEPOINT before_bob_transfer;

UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob';
-- Bob: 600

-- Oops, meant to transfer to Charlie instead
ROLLBACK TO before_bob_transfer;
-- Bob: 500 (reverted), Alice: 900 (kept)

UPDATE accounts SET balance = balance + 100 WHERE name = 'Charlie';
-- Charlie: 850

COMMIT;
-- Result: Alice -100, Charlie +100, Bob unchanged
```

### Practical Use Case

```sql
-- Data import with error handling: continue after partial failure

BEGIN;

-- Insert batch of 1000 records
INSERT INTO products (name, category, price) 
SELECT name, category, price FROM staging_data WHERE batch_id = 1;
-- 998 inserted successfully, 2 failed (duplicate SKU)

SAVEPOINT after_batch_1;

-- Try batch 2
INSERT INTO products (name, category, price) 
SELECT name, category, price FROM staging_data WHERE batch_id = 2;
-- All 1000 succeeded

SAVEPOINT after_batch_2;

-- Try batch 3
INSERT INTO products (name, category, price) 
SELECT name, category, price FROM staging_data WHERE batch_id = 3;
-- ERROR: one product references non-existent category

-- Rollback only batch 3; batches 1-2 remain
ROLLBACK TO after_batch_2;

COMMIT;  -- Batch 1 (998 records) and Batch 2 (1000 records) are committed
```

### Nested Transactions

PostgreSQL doesn't support true nested transactions, but savepoints simulate them:

```sql
BEGIN;

DO $$
BEGIN
  INSERT INTO log (message) VALUES ('Operation started');
  SAVEPOINT sp_nested;
  
  BEGIN  -- ← This is NOT a nested BEGIN; it's just a no-op comment
    UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
    SAVEPOINT sp_nested_inner;
    
    UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob';
  END;

EXCEPTION WHEN OTHERS THEN
  ROLLBACK TO sp_nested_inner;
  INSERT INTO log (message) VALUES ('Bob update failed');
END;
$$;

COMMIT;
```

> **Edge Case — Savepoint names are transaction-scoped:** If you release or rollback to a savepoint, you cannot use that name again in the same transaction without creating a new one:
> ```sql
> BEGIN;
> SAVEPOINT sp1;
>   UPDATE accounts SET balance = 0;
> ROLLBACK TO sp1;
>
> SAVEPOINT sp1;  -- OK: creating a new savepoint with the same name
> ```

> **Edge Case — Savepoints across transactions:** Savepoints are automatically released when a transaction commits or rolls back entirely. They exist only within the current transaction.

---

## 4. Why Transactions Matter (Atomic Operations)

### Atomicity Guarantee

Without transactions, multi-step operations can fail partway through, leaving corrupted data:

```sql
-- Without explicit transaction (statements autocommit individually):

UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
-- COMMITTED: Alice's balance reduced (server crashes here)

UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob';
-- NEVER RUNS: Bob receives nothing, Alice's money disappears

-- Result: $100 vanished from the system
```

```sql
-- With explicit transaction (all-or-nothing):

BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
  UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob';
COMMIT;  -- (server crashes here, before COMMIT)

-- Recovery after restart:
-- PostgreSQL replays transaction from WAL log
-- Sees that the transaction never committed → rolls back all changes
-- Alice still has $1000, Bob still has $500 (atomicity preserved)
```

### Data Integrity Without Transactions

```sql
-- Scenario: Update a customer's address AND postal code (FK to zip_codes table)

-- Without transaction (bad):
UPDATE customers SET street_address = '123 Main' WHERE id = 1;
-- Committed

-- Postal code doesn't exist, update fails:
UPDATE customers SET postal_code = '99999' WHERE id = 1;
-- ERROR: FK violation
-- Result: Customer now has an inconsistent state (new address, old postal code)

-- With transaction (good):
BEGIN;
  UPDATE customers SET street_address = '123 Main' WHERE id = 1;
  UPDATE customers SET postal_code = '99999' WHERE id = 1;
COMMIT;
-- ERROR: FK violation (before commit)
-- ROLLBACK: both updates undone; customer's data remains consistent
```

### Transfer Consistency

The canonical example — transferring money between accounts:

```sql
-- Without transactions:
SELECT * FROM accounts;
-- Alice: $1000, Bob: $500

UPDATE accounts SET balance = balance - 500 WHERE name = 'Alice';
-- Committed (Alice: $500)
-- Server crashes

-- After restart:
SELECT * FROM accounts;
-- Alice: $500, Bob: $500
-- $500 vanished from the system (inconsistent state)

-- With transactions:
BEGIN;
  UPDATE accounts SET balance = balance - 500 WHERE name = 'Alice';
  UPDATE accounts SET balance = balance + 500 WHERE name = 'Bob';
COMMIT;

-- If server crashes before COMMIT, both are rolled back (consistent)
-- If server crashes after COMMIT, both are durable (consistent)
-- No in-between state possible
```

---

## 5. Isolation Levels Overview

**Isolation levels** define how concurrent transactions interact. PostgreSQL implements the SQL standard isolation levels:

- Weaker isolation = higher concurrency (but more anomalies possible)
- Stronger isolation = lower concurrency (but more consistent)

### READ UNCOMMITTED

The weakest level. Transactions can see **uncommitted changes** from other transactions (**dirty reads**).

PostgreSQL does **not** actually support this level — `READ UNCOMMITTED` is treated as `READ COMMITTED` instead.

```sql
-- Conceptually (not in PostgreSQL):
Transaction A:
  BEGIN;
  UPDATE accounts SET balance = 1000000 WHERE name = 'Alice';
  -- Not committed yet

Transaction B:
  SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
  BEGIN;
  SELECT balance FROM accounts WHERE name = 'Alice';
  -- Sees 1000000 (dirty read — uncommitted value)
  COMMIT;

Transaction A:
  ROLLBACK;  -- Oops, that was a mistake

-- Transaction B has read data that never actually existed (anomaly)
```

### READ COMMITTED (Default)

PostgreSQL's default. Transactions only see **committed** data from other transactions. If a row is modified between your query start and execution, you see the **new committed value**.

```sql
Transaction A:
  BEGIN;
  SELECT balance FROM accounts WHERE name = 'Alice';
  -- Sees $1000 (committed value)
  -- (some time passes)
  SELECT balance FROM accounts WHERE name = 'Alice';
  -- Sees $900 (if Transaction B updated and committed it in between)
  -- Possible anomaly: non-repeatable read (same query returns different results)
  COMMIT;
```

**The problem:** A query can return different data on consecutive runs within the same transaction if another transaction modifies rows in between.

### REPEATABLE READ

Snapshot isolation. Each transaction sees a **consistent snapshot** of the database at the time the transaction started. Updates made by other transactions are not visible, even if they committed.

```sql
Transaction A:
  BEGIN;
  SELECT balance FROM accounts WHERE name = 'Alice';
  -- Sees $1000 (snapshot taken at transaction start)
  -- (some time passes, Transaction B updates and commits Alice to $900)
  SELECT balance FROM accounts WHERE name = 'Alice';
  -- Still sees $1000 (same snapshot)
  COMMIT;
```

**Guarantee:** Repeated queries return the same result within a transaction.

**Anomaly:** Phantom reads still possible (rows inserted by other transactions after your query runs).

### SERIALIZABLE

The strongest isolation level. Transactions behave as if they ran **one after another** sequentially, even if they actually ran concurrently.

```sql
-- Impossible scenarios are prevented:
-- Transaction A updates row 1
-- Transaction B updates row 1 (blocked until A commits)
-- No conflicting concurrent modifications

-- Phantom reads prevented:
-- Transaction A: SELECT * FROM orders WHERE status = 'pending'
-- Transaction B: INSERT INTO orders (status) VALUES ('pending')
-- Transaction A: SELECT * FROM orders WHERE status = 'pending'
--   → Still returns the same rows (phantom insert not visible)
```

**Cost:** Reduced concurrency. Transactions may be rejected with serialization errors if conflicts are detected.

```sql
-- In SERIALIZABLE mode:
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
  UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
  -- ...other transaction modified the same rows...
  UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob';
COMMIT;
-- ERROR: serialization failure — can retry the transaction
```

### Isolation Levels Comparison Table

| Level | Dirty Reads | Non-Repeatable Reads | Phantom Reads | Concurrency |
|---|---|---|---|---|
| READ UNCOMMITTED | Yes | Yes | Yes | Highest |
| READ COMMITTED (default) | No | Yes | Yes | High |
| REPEATABLE READ | No | No | Yes | Medium |
| SERIALIZABLE | No | No | No | Lowest |

**In PostgreSQL specifically:**
- `READ UNCOMMITTED` is treated as `READ COMMITTED`
- `REPEATABLE READ` is actually snapshot isolation (slightly stronger than SQL standard)
- `SERIALIZABLE` uses serialization conflict detection

### Setting Isolation Level

```sql
-- For a single transaction
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
  -- statements
COMMIT;

-- Or on an active transaction
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- For the entire session
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

> **Edge Case — SERIALIZABLE with concurrent writes:** If two transactions try to write to the same rows, PostgreSQL detects the conflict and fails one of them:
> ```sql
> Transaction A:
>   BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
>   SELECT COUNT(*) FROM orders;  -- sees 100
>   INSERT INTO orders VALUES (...);  -- tries to insert
>   -- (Transaction B committed a change to orders in between)
>   COMMIT;
>   -- ERROR: serialization failure — rollback and retry
> ```

---

## 6. Common Mistake — Forgetting COMMIT in psql

The #1 transaction mistake: running `BEGIN`, then queries, then **forgetting COMMIT**. The transaction stays open, locks remain acquired, and other sessions get blocked.

### The Problem

```sql
postgres=# BEGIN;
BEGIN

postgres=# UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
UPDATE 1

postgres=# SELECT * FROM accounts;
 id | name    | balance
----+---------+---------
  1 | Alice   |  900.00
  2 | Bob     |  500.00

postgres=# -- Oops, forgot to commit. Now what?
-- Transaction is still open, rows are locked...

-- Other session tries to update Alice:
UPDATE accounts SET balance = balance + 100 WHERE name = 'Alice';
-- WAITING indefinitely (locked by your transaction)
```

### The Fix

**Always commit or rollback:**

```sql
postgres=# COMMIT;  -- persists changes and releases locks
COMMIT

-- Or if you changed your mind:
postgres=# ROLLBACK;  -- discards changes and releases locks
ROLLBACK
```

### How to Avoid

**1. Use transactions explicitly with code patterns:**

```python
# Python example — application manages the transaction
try:
    cursor.execute("UPDATE accounts ...")
    cursor.execute("UPDATE accounts ...")
    connection.commit()  # ← automatic after success
except Exception as e:
    connection.rollback()  # ← automatic on error
    print(f"Transaction failed: {e}")
```

**2. In psql, use a text editor and template:**

```sql
-- template.sql
BEGIN;

-- your statements here
UPDATE ...;
UPDATE ...;

COMMIT;  -- ← make it impossible to forget
```

Then:
```bash
$ psql -f template.sql -d mydb
```

**3. Enable warnings in psql:**

```sql
-- In your ~/.psqlrc:
\set ON_ERROR_ROLLBACK on  -- auto-rollback on error (helps, but not a complete solution)

-- Or use a transaction timeout:
SET statement_timeout TO '5 minutes';  -- auto-rollback if no action for 5 min
```

**4. Check transaction status:**

```sql
-- In psql, see if you're in a transaction:
\set
-- Shows ON_ERROR_ROLLBACK status

-- Or query PostgreSQL directly:
SELECT current_transaction_isolation;  -- returns current level
```

### Detecting an Open Transaction

```sql
-- If you're unsure whether you're in a transaction:
postgres=# SELECT 1;
-- If response includes "(BEGIN)" status indicator, you're in a transaction

-- More explicit:
postgres=# SHOW transaction_status;  -- returns 'idle' or 'in transaction'
transaction_status
-----------
 in transaction
```

### Recovery from Forgotten COMMIT

If you realize mid-transaction you want to abort:

```sql
postgres=# ROLLBACK;  -- undo all changes, release locks
ROLLBACK
```

If you want to keep changes:

```sql
postgres=# COMMIT;  -- commit changes, release locks
COMMIT
```

> **Edge Case — Idle transactions holding locks:** Even if a transaction does nothing, it keeps locks on accessed rows. This can block other sessions indefinitely:
> ```sql
> Transaction A:
> BEGIN;
> SELECT * FROM accounts WHERE name = 'Alice';  -- acquires locks
> -- ... idle for 2 hours ...
> COMMIT;
>
> Transaction B:
> UPDATE accounts SET balance = 0 WHERE name = 'Alice';
> -- BLOCKED: waiting for Transaction A's locks to be released
> ```
> Solution: Kill idle transactions as a DBA or add idle timeouts:
> ```sql
> -- Terminate idle transactions older than 30 minutes
> SELECT pg_terminate_backend(pid)
> FROM pg_stat_activity
> WHERE state = 'idle in transaction'
>   AND state_change < NOW() - INTERVAL '30 minutes';
> ```

---

## Quick Reference

```sql
-- Basic transaction
BEGIN [TRANSACTION];
  SQL statements here
COMMIT;      -- persist changes, release locks
-- or
ROLLBACK;    -- discard changes, release locks

-- Transaction with isolation level
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
  -- statements
COMMIT;

-- Savepoints
BEGIN;
  statement_1;
  SAVEPOINT sp1;      -- mark a point
    statement_2;
    statement_3;      -- fails
  ROLLBACK TO sp1;    -- undo to sp1, keep statement_1
  statement_4;        -- new attempt
COMMIT;

-- Check transaction status (psql)
\set                                     -- see ON_ERROR_ROLLBACK
SHOW transaction_status;                 -- 'idle' or 'in transaction'

-- Isolation levels (READ UNCOMMITTED treats as READ COMMITTED in PostgreSQL)
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;     -- default
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Auto-commit (psql-specific)
\set ON_ERROR_ROLLBACK off   -- turn off error rollback (default)
\set ON_ERROR_ROLLBACK on    -- auto-rollback on error

-- Transaction in application (Python example):
try:
    conn.execute("BEGIN;")
    conn.execute("UPDATE ...")
    conn.execute("UPDATE ...")
    conn.commit()             # ← don't forget
except Exception as e:
    conn.rollback()
    print(f"Failed: {e}")

-- Detect and kill idle transactions (admin)
SELECT pid, usename, state, state_change
FROM pg_stat_activity
WHERE state = 'idle in transaction';

SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND state_change < NOW() - INTERVAL '30 minutes';

-- ACID properties (conceptual)
-- A (Atomicity):    all-or-nothing (BEGIN...COMMIT or ROLLBACK)
-- C (Consistency):  all constraints enforced
-- I (Isolation):    concurrent transactions don't interfere
-- D (Durability):   committed data survives crashes (WAL)
```
