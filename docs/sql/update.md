---
title: UPDATE
description: Modifying existing rows with UPDATE in ANSI SQL
tags: [sql, dml, update]
---

# UPDATE

`UPDATE` modifies existing rows in a table. It is one of the most powerful and dangerous DML commands — a missing `WHERE` clause will update every row in the table.

---

## Basic Syntax

```sql
UPDATE table_name
SET column1 = value1,
    column2 = value2
WHERE condition;
```

---

## Update a Single Column

```sql
UPDATE customers
SET email = 'new@example.com'
WHERE customer_id = 42;
```

---

## Update Multiple Columns

```sql
UPDATE employees
SET salary     = salary * 1.10,
    updated_at = CURRENT_TIMESTAMP
WHERE department = 'Engineering'
  AND performance_rating = 'Excellent';
```

---

## Update with Expressions

```sql
-- Apply a percentage increase
UPDATE products
SET price = price * 1.15
WHERE category = 'Electronics';

-- Concatenate to an existing value
UPDATE customers
SET notes = COALESCE(notes, '') || ' — verified'
WHERE verified = TRUE;
```

!!! note
    The `||` operator is ANSI SQL for string concatenation but SQL Server uses `+` instead. For cross-platform queries use `CONCAT(COALESCE(notes, ''), ' — verified')`. See [Scalar Functions](scalar-functions.md).

---

## Update Using a Subquery

Update rows based on values from another table.

```sql
-- Flag orders from inactive customers
UPDATE orders
SET status = 'flagged'
WHERE customer_id IN (
    SELECT customer_id
    FROM customers
    WHERE status = 'inactive'
);
```

---

## Update with JOIN (Vendor Specific)

ANSI SQL does not support `UPDATE ... JOIN` directly. The portable approach uses a subquery or `MERGE`.

```sql
-- ANSI SQL — update using a correlated subquery
UPDATE orders
SET amount = (
    SELECT discounted_price
    FROM pricing
    WHERE pricing.product_id = orders.product_id
)
WHERE EXISTS (
    SELECT 1 FROM pricing WHERE pricing.product_id = orders.product_id
);
```

!!! warning
    If the subquery returns `NULL` for any row — for example when `discounted_price` is NULL in the pricing table — the `amount` column will be silently set to `NULL`. Always validate the source data before updating, or add a `WHERE discounted_price IS NOT NULL` condition inside the subquery.

```sql
-- SQL Server (non-ANSI)
UPDATE o
SET o.amount = p.discounted_price
FROM orders o
JOIN pricing p ON o.product_id = p.product_id;

-- PostgreSQL (non-ANSI)
UPDATE orders o
SET amount = p.discounted_price
FROM pricing p
WHERE o.product_id = p.product_id;
```

!!! tip
    For cross-platform portability, use `MERGE` when you need to update rows based on a join. It is ANSI SQL and expresses the intent more clearly. See [MERGE / UPSERT](merge.md).

---

## Soft Delete via UPDATE

Instead of deleting rows, mark them as inactive. This preserves history and supports auditing.

```sql
UPDATE customers
SET
    is_active  = FALSE,
    deleted_at = CURRENT_TIMESTAMP
WHERE customer_id = 42;
```

---

## Test Before You Execute

Always validate the scope of an `UPDATE` by running the same `WHERE` clause as a `SELECT` first — confirm the row count and affected values before making any changes.

```sql
-- Step 1: verify which rows will be affected
SELECT customer_id, email
FROM customers
WHERE status = 'trial'
  AND created_at < CURRENT_DATE - INTERVAL '30' DAY;

-- Step 2: update those rows
UPDATE customers
SET status = 'expired'
WHERE status = 'trial'
  AND created_at < CURRENT_DATE - INTERVAL '30' DAY;
```

---

## Best Practices

- Always include a `WHERE` clause — an `UPDATE` without one modifies every row
- Test with `SELECT` using the same `WHERE` clause before running the update
- Use transactions when updating multiple related tables together
- Use `COALESCE` when appending to nullable columns to avoid NULL concatenation issues
- Use `CONCAT()` over `||` for string operations in cross-platform queries
- Prefer `MERGE` over `UPDATE ... JOIN` for cross-platform compatibility
- Add an `updated_at` timestamp column to tables that change frequently — update it on every write