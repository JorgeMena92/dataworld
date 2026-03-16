---
title: TRUNCATE
description: Removing all rows from a table efficiently with TRUNCATE
tags: [sql, ddl, truncate]
---

# TRUNCATE

`TRUNCATE` removes all rows from a table instantly while keeping the table structure, constraints, and indexes intact. It is the fastest way to empty a table — significantly faster than `DELETE` on large datasets.

---

## Basic Syntax

```sql
TRUNCATE TABLE orders;

-- Multiple tables at once
TRUNCATE TABLE staging_orders, staging_customers;
```

!!! note
    Truncating multiple tables in a single statement is supported in PostgreSQL and SQL Server. MySQL does not support it for InnoDB tables with active foreign key constraints — truncate each table individually in that case.

---

## TRUNCATE vs DELETE

| | `TRUNCATE` | `DELETE` |
|---|---|---|
| Removes all rows | ✅ | ✅ (without WHERE) |
| Can filter with WHERE | ❌ | ✅ |
| Can be rolled back | Depends on DB | ✅ |
| Fires row-level triggers | ❌ (usually) | ✅ |
| Resets identity/sequence | ✅ (usually) | ❌ |
| Speed on large tables | Very fast | Slow |
| Logs individual row deletions | ❌ (minimal logging) | ✅ |

```sql
-- TRUNCATE — empties the table in one operation
TRUNCATE TABLE temp_log;

-- DELETE — removes rows one by one, can be filtered and rolled back
DELETE FROM temp_log WHERE created_at < '2024-01-01';
```

---

## TRUNCATE and Identity / Sequences

One of the key differences from `DELETE` — `TRUNCATE` resets auto-increment counters back to their starting value.

```sql
-- Before TRUNCATE: last inserted ID was 9500
TRUNCATE TABLE orders;
-- After TRUNCATE: next inserted ID will start from 1 again

-- DELETE does NOT reset the sequence
DELETE FROM orders;
-- After DELETE: next inserted ID continues from 9501
```

!!! tip
    If you need to preserve the current sequence value (e.g. during a reload that must maintain ID continuity), use `DELETE` instead of `TRUNCATE`.

---

## TRUNCATE with CASCADE

Some databases support cascading truncation — truncating a parent table and all related child tables in one operation.

```sql
-- PostgreSQL — truncate orders and all tables with foreign keys referencing it
TRUNCATE TABLE orders CASCADE;
```

!!! warning
    `TRUNCATE CASCADE` silently empties all dependent tables. Verify the full dependency chain before using it.

---

## TRUNCATE and Transactions

Transactional behavior varies significantly across databases.

```sql
-- PostgreSQL — TRUNCATE is transactional and can be rolled back
BEGIN;
TRUNCATE TABLE staging_orders;
ROLLBACK;
-- Table still has data

-- MySQL — TRUNCATE causes an implicit commit and cannot be rolled back
BEGIN;
TRUNCATE TABLE staging_orders;
ROLLBACK;
-- Table is empty — TRUNCATE committed immediately
```

| Database | TRUNCATE in transaction |
|---|---|
| PostgreSQL | ✅ Transactional |
| SQL Server | ✅ Transactional |
| MySQL | ❌ Implicit commit |
| Oracle | ❌ Implicit commit |

---

## Common Use Case — Full Reload Pattern

`TRUNCATE` is the standard first step in a full table reload pipeline.

```sql
BEGIN;

-- Clear the table
TRUNCATE TABLE staging_orders;

-- Reload from source
INSERT INTO staging_orders
SELECT * FROM raw_orders WHERE load_date = CURRENT_DATE;

-- Validate
-- Promote to production if valid

COMMIT;
```

---

## Vendor Notes

| Feature | ANSI SQL | SQL Server | PostgreSQL | MySQL |
|---|---|---|---|---|
| Basic TRUNCATE | ✅ | ✅ | ✅ | ✅ |
| Resets identity | Depends | ✅ | ✅ | ✅ |
| CASCADE support | — | ❌ | ✅ | ❌ |
| Transactional | Depends | ✅ | ✅ | ❌ |

!!! note
    Identity/sequence reset on `TRUNCATE` is not defined by the ANSI SQL standard — it is implementation-specific behavior. All major platforms reset the counter but the starting value may vary. For a full comparison of `TRUNCATE` vs `DELETE` behavior including rollback support and trigger firing, see [DELETE](delete.md#delete-vs-truncate).

---

## Best Practices

- Use `TRUNCATE` instead of `DELETE` when clearing entire tables — it is significantly faster
- Be aware of transactional behavior differences across databases before using in pipelines
- Use `TRUNCATE CASCADE` carefully — verify all dependent tables before running
- Wrap `TRUNCATE` in a transaction in databases that support it (PostgreSQL, SQL Server)
- If you need to preserve the sequence value, use `DELETE FROM table` instead