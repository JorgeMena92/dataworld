---
title: DELETE
description: Removing rows with DELETE in ANSI SQL
tags: [sql, dml, delete]
---

# DELETE

`DELETE` removes rows from a table. Like `UPDATE`, it is irreversible once committed — and a missing `WHERE` clause deletes every row in the table.

---

## Basic Syntax

```sql
DELETE FROM table_name
WHERE condition;
```

---

## Delete Specific Rows

```sql
-- Delete a single record
DELETE FROM customers
WHERE customer_id = 42;

-- Delete based on multiple conditions
DELETE FROM orders
WHERE status = 'cancelled'
  AND created_at < '2023-01-01';
```

---

## Delete Using a Subquery

```sql
-- Delete orders from inactive customers
DELETE FROM orders
WHERE customer_id IN (
    SELECT customer_id
    FROM customers
    WHERE status = 'inactive'
);

-- Delete with NOT EXISTS
DELETE FROM products
WHERE NOT EXISTS (
    SELECT 1
    FROM order_items
    WHERE order_items.product_id = products.product_id
);
```

---

## DELETE vs TRUNCATE

Both remove rows, but they behave very differently.

| | `DELETE` | `TRUNCATE` |
|---|---|---|
| Filters rows with `WHERE` | ✅ | ❌ |
| Can be rolled back | ✅ | Depends on DB |
| Fires triggers | ✅ | ❌ (usually) |
| Resets identity/sequence | ❌ | ✅ (usually) |
| Speed on large tables | Slow | Fast |

```sql
-- DELETE — removes rows one by one, can be filtered and rolled back
DELETE FROM temp_log WHERE created_at < '2024-01-01';

-- TRUNCATE — removes all rows instantly, cannot be filtered
TRUNCATE TABLE temp_log;
```

!!! note
    `TRUNCATE` rollback support varies by platform. PostgreSQL supports `TRUNCATE` inside a transaction and it can be rolled back. SQL Server and MySQL commit `TRUNCATE` immediately — it cannot be rolled back once executed. See [TRUNCATE](truncate.md) for the full treatment.

!!! tip
    Use `TRUNCATE` when you want to clear an entire table quickly. Use `DELETE` when you need to filter which rows to remove or need rollback support.

---

## Soft Delete — The Safer Alternative

Instead of physically removing rows, mark them as deleted. This preserves history and is reversible.

```sql
-- Soft delete
UPDATE customers
SET
    is_active  = FALSE,
    deleted_at = CURRENT_TIMESTAMP
WHERE customer_id = 42;

-- Query only active records
SELECT * FROM customers WHERE is_active = TRUE;
-- or
SELECT * FROM customers WHERE deleted_at IS NULL;
```

Soft deletes are the preferred pattern in production systems where:
- Audit trails are required
- Data recovery needs to be possible
- Downstream reports reference the same table

---

## Cascading Deletes

When a parent row is deleted, related child rows can be deleted automatically via foreign key constraints.

```sql
-- Define cascade at table creation
CREATE TABLE order_items (
    item_id    INT PRIMARY KEY,
    order_id   INT,
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE
);

-- Deleting the order automatically deletes its items
DELETE FROM orders WHERE order_id = 101;
```

!!! warning
    Cascading deletes can remove far more data than intended if the dependency chain is deep. Always understand the full cascade path before using `ON DELETE CASCADE` in production.

---

## Test Before You Execute

Always validate the scope of a `DELETE` by running the same `WHERE` clause as a `SELECT` first. Wrapping the operation in a transaction gives you a final safety net before committing.

```sql
BEGIN;

-- Step 1: verify what will be deleted
SELECT COUNT(*)
FROM orders
WHERE status = 'cancelled'
  AND created_at < '2023-01-01';

-- Step 2: delete
DELETE FROM orders
WHERE status = 'cancelled'
  AND created_at < '2023-01-01';

-- Step 3: review row count, then commit or rollback
COMMIT;
-- or ROLLBACK;
```

---

## Best Practices

- Always include a `WHERE` clause — a `DELETE` without one removes every row
- Test with `SELECT COUNT(*)` using the same `WHERE` clause before deleting
- Wrap deletes in a transaction so you can roll back if something looks wrong
- Prefer soft deletes in production systems where audit history matters
- Use `TRUNCATE` instead of `DELETE` when clearing entire tables — it is significantly faster
- Be careful with cascading deletes — verify the full dependency chain first