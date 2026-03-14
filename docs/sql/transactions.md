---
title: Transactions
description: COMMIT, ROLLBACK, SAVEPOINT, and transaction control in ANSI SQL
tags: [sql, dml, transactions, tcl]
---

# Transactions

A transaction is a group of SQL statements that execute as a single unit. Either all statements succeed and the changes are saved, or the transaction fails and every change is undone. This is the foundation of data integrity in relational databases.

---

## ACID Properties

Transactions guarantee four properties known as ACID:

| Property | Description |
|---|---|
| **Atomicity** | All statements succeed or none of them do |
| **Consistency** | The database moves from one valid state to another |
| **Isolation** | Concurrent transactions do not interfere with each other |
| **Durability** | Committed changes survive system failures |

---

## Basic Transaction Control

```sql
BEGIN;          -- or START TRANSACTION in some databases

    INSERT INTO accounts (account_id, balance) VALUES (1, 1000);
    INSERT INTO accounts (account_id, balance) VALUES (2, 500);

COMMIT;         -- save all changes permanently
```

```sql
BEGIN;

    UPDATE accounts SET balance = balance - 200 WHERE account_id = 1;
    UPDATE accounts SET balance = balance + 200 WHERE account_id = 2;

ROLLBACK;       -- undo all changes, as if they never happened
```

---

## The Classic Bank Transfer

The canonical transaction example — money must leave one account and arrive in another atomically.

```sql
BEGIN;

UPDATE accounts SET balance = balance - 500 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 500 WHERE account_id = 2;

-- Both updates succeeded — commit
COMMIT;

-- If anything failed — rollback
-- ROLLBACK;
```

Without a transaction, a system failure between the two `UPDATE` statements would leave the database in an inconsistent state — money deducted from one account but never added to the other.

---

## SAVEPOINT

A savepoint marks a point within a transaction that you can roll back to without undoing the entire transaction.

```sql
BEGIN;

INSERT INTO orders (customer_id, amount) VALUES (42, 199.99);
SAVEPOINT after_order;

INSERT INTO order_items (order_id, product_id, quantity) VALUES (101, 7, 2);
SAVEPOINT after_items;

UPDATE inventory SET stock = stock - 2 WHERE product_id = 7;

-- Something went wrong with inventory — roll back to after_items
ROLLBACK TO after_items;

-- The order and order_items inserts are still intact
COMMIT;
```

```sql
-- Release a savepoint when no longer needed
RELEASE SAVEPOINT after_order;
```

---

## Autocommit

Most databases run in autocommit mode by default — each statement is its own transaction and is committed immediately.

```sql
-- In autocommit mode, this is committed instantly with no way to roll back
DELETE FROM orders WHERE status = 'cancelled';

-- To control the transaction, explicitly begin one
BEGIN;
DELETE FROM orders WHERE status = 'cancelled';
-- Now you can COMMIT or ROLLBACK before it is permanent
ROLLBACK;
```

!!! tip
    Always use explicit `BEGIN` / `COMMIT` blocks when running multi-step DML in production. Never rely on autocommit for destructive operations.

---

## Transactions in Data Pipelines

Transactions are essential in ETL to ensure a pipeline either fully completes or leaves the target in its previous clean state.

```sql
BEGIN;

-- Step 1: clear the staging area
TRUNCATE TABLE staging_orders;

-- Step 2: load new data
INSERT INTO staging_orders
SELECT * FROM raw_orders WHERE load_date = CURRENT_DATE;

-- Step 3: validate before promoting
-- Run this SELECT and verify the count before proceeding
-- If the result is 0, ROLLBACK instead of committing
SELECT COUNT(*) FROM staging_orders;

-- Step 4: promote to production
INSERT INTO orders SELECT * FROM staging_orders;

COMMIT;
-- or ROLLBACK if validation in Step 3 failed
```

!!! note
    Inline validation logic (raising exceptions on empty results) requires procedural SQL extensions like PL/pgSQL (PostgreSQL) or T-SQL (SQL Server) — these are not ANSI SQL. In portable pipelines, run the validation `SELECT` as a separate step and handle the decision to commit or rollback in the calling application or orchestration layer.

!!! warning
    `TRUNCATE` inside a transaction is only reliably rollback-safe in PostgreSQL. In SQL Server and MySQL, `TRUNCATE` commits immediately and cannot be rolled back. See [DELETE vs TRUNCATE](delete.md#delete-vs-truncate) for platform details.

---

## Isolation Levels

Isolation levels control how much a transaction is affected by other concurrent transactions.

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| `READ UNCOMMITTED` | ✅ Possible | ✅ Possible | ✅ Possible |
| `READ COMMITTED` | ❌ Prevented | ✅ Possible | ✅ Possible |
| `REPEATABLE READ` | ❌ Prevented | ❌ Prevented | ✅ Possible |
| `SERIALIZABLE` | ❌ Prevented | ❌ Prevented | ❌ Prevented |

```sql
-- Set isolation level for the current transaction
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN;
-- ...
COMMIT;
```

!!! tip
    `READ COMMITTED` is the default in most databases and the right choice for most analytics and pipeline workloads. Use `SERIALIZABLE` only when strict consistency is required — it has significant performance costs.

---

## Vendor Notes

| Feature | ANSI SQL | SQL Server | PostgreSQL | MySQL |
|---|---|---|---|---|
| Start transaction | `BEGIN` | `BEGIN TRANSACTION` | `BEGIN` | `START TRANSACTION` |
| Savepoints | `SAVEPOINT` | `SAVE TRANSACTION` | `SAVEPOINT` | `SAVEPOINT` |
| Default autocommit | Depends | ON | ON | ON |

---

## Best Practices

- Use explicit `BEGIN` / `COMMIT` for any multi-step DML operation
- Always have a `ROLLBACK` path — either explicit or via error handling
- Use `SAVEPOINT` for long transactions with multiple logical steps
- Keep transactions as short as possible — long transactions lock resources and block other operations
- Never run destructive operations (`DELETE`, `UPDATE`, `TRUNCATE`) outside a transaction in production
- Validate data after loading and before committing — rollback if validation fails