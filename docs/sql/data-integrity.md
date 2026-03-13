---
title: Data Integrity
description: Ensuring accuracy and consistency across DML operations and data pipelines
tags: [sql, data-integrity]
---

# Data Integrity

Data integrity means the data in your database is accurate, consistent, and trustworthy. It can break silently — through duplicate inserts, orphaned records, missing foreign keys, or unchecked nulls entering a pipeline. This page covers practical patterns to enforce and verify integrity across the full data lifecycle.

---

## Types of Integrity

| Type | Description | Enforced by |
|---|---|---|
| **Entity** | Each row is uniquely identifiable | Primary keys |
| **Referential** | Foreign keys point to existing rows | Foreign key constraints |
| **Domain** | Values fall within accepted ranges | Data types, CHECK, NOT NULL |
| **Pipeline** | Loads are complete, consistent, and non-duplicating | Validation queries, defensive DML |

---

## Constraints — Database-Level Enforcement

Constraints are the first and most reliable line of defense. They are enforced automatically on every DML operation.

```sql
CREATE TABLE orders (
    order_id    INT           PRIMARY KEY,
    customer_id INT           NOT NULL REFERENCES customers(customer_id),
    amount      DECIMAL(10,2) CHECK (amount > 0),
    status      VARCHAR(20)   DEFAULT 'pending'
                              CHECK (status IN ('pending', 'completed', 'cancelled')),
    created_at  TIMESTAMP     DEFAULT CURRENT_TIMESTAMP
);
```

Any `INSERT` or `UPDATE` that violates a constraint fails immediately — before the data is written.

---

## Pre-load Validation

Validate source data before inserting it into a target table. Catch problems early, not after they are committed.

```sql
-- Check for NULLs in required columns
SELECT COUNT(*) AS null_customer_ids
FROM staging_orders
WHERE customer_id IS NULL;

-- Check for invalid values
SELECT COUNT(*) AS invalid_amounts
FROM staging_orders
WHERE amount <= 0;

-- Check referential integrity — orphan detection before insert
SELECT COUNT(*) AS missing_customers
FROM staging_orders s
WHERE NOT EXISTS (
    SELECT 1 FROM customers c
    WHERE c.customer_id = s.customer_id
);

-- Check for duplicates in the source
SELECT customer_id, COUNT(*) AS occurrences
FROM staging_customers
GROUP BY customer_id
HAVING COUNT(*) > 1;
```

!!! tip
    Run pre-load validation as the first step of every pipeline. If any check fails, abort the load and alert — do not insert partial or dirty data.

---

## Defensive DML

Filter invalid rows during the insert itself — only load what passes validation.

```sql
-- Only insert rows with valid foreign keys and positive amounts
INSERT INTO orders (customer_id, amount, status)
SELECT
    s.customer_id,
    s.amount,
    COALESCE(s.status, 'pending')
FROM staging_orders s
WHERE EXISTS (
    SELECT 1 FROM customers c
    WHERE c.customer_id = s.customer_id
)
AND s.amount > 0
AND s.customer_id IS NOT NULL;
```

```sql
-- Only insert customers that do not already exist
INSERT INTO customers (customer_id, email, created_at)
SELECT s.customer_id, s.email, CURRENT_TIMESTAMP
FROM staging_customers s
WHERE NOT EXISTS (
    SELECT 1 FROM customers c
    WHERE c.customer_id = s.customer_id
);
```

---

## Post-load Checks

Verify the result after a load completes — confirm row counts, detect orphans, and validate aggregates.

```sql
-- Row count matches expectation
SELECT COUNT(*) FROM orders
WHERE load_date = CURRENT_DATE;

-- No orphaned order items after a load or delete
SELECT COUNT(*) AS orphaned_items
FROM order_items oi
WHERE NOT EXISTS (
    SELECT 1 FROM orders o
    WHERE o.order_id = oi.order_id
);

-- Aggregate check — total revenue in target matches source
SELECT
    (SELECT SUM(amount) FROM staging_orders)  AS source_total,
    (SELECT SUM(amount) FROM orders
     WHERE load_date = CURRENT_DATE)          AS target_total;
```

---

## Handling Duplicates

Duplicates are one of the most common integrity problems in data pipelines — especially with incremental loads.

```sql
-- Detect duplicates
SELECT customer_id, COUNT(*) AS occurrences
FROM customers
GROUP BY customer_id
HAVING COUNT(*) > 1;

-- Remove duplicates — keep the latest record per key
DELETE FROM customers
WHERE customer_id NOT IN (
    SELECT MAX(customer_id)
    FROM customers
    GROUP BY email
);

-- Deduplicate with ROW_NUMBER before inserting
INSERT INTO customers_clean
SELECT * FROM (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY email
            ORDER BY created_at DESC
        ) AS rn
    FROM customers_raw
) t
WHERE rn = 1;
```

---

## Referential Integrity in Pipelines

When loading related tables, always load parent tables before child tables.

```sql
-- Correct order — parents first
INSERT INTO customers ...;   -- parent
INSERT INTO orders ...;      -- child — references customers
INSERT INTO order_items ...; -- grandchild — references orders

-- Wrong order will cause foreign key violations
INSERT INTO order_items ...; -- fails — orders don't exist yet
```

When deleting, reverse the order — delete children before parents, or use `CASCADE`.

```sql
-- Delete children first
DELETE FROM order_items WHERE order_id IN (...);
DELETE FROM orders WHERE customer_id = 42;
DELETE FROM customers WHERE customer_id = 42;
```

---

## Best Practices

- Define constraints at the DDL level — let the database enforce what it can
- Run pre-load validation before every pipeline load and abort on failure
- Use defensive `INSERT ... WHERE EXISTS / NOT EXISTS` to filter invalid rows at load time
- Always check row counts and key aggregates after loading
- Load parent tables before child tables — delete in reverse order
- Never silently swallow constraint violations in application code — surface them as pipeline failures
- Log validation results so failures are traceable and auditable
