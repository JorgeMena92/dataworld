---
title: Idempotent Operations
description: Writing SQL operations that can be safely re-run without side effects
tags: [sql, patterns, idempotent, pipelines]
---

# Idempotent Operations

An idempotent operation produces the same result regardless of how many times it is executed. In data pipelines, idempotency means a pipeline can be safely re-run after a failure — without creating duplicate data, corrupting existing records, or requiring manual cleanup.

---

## Why Idempotency Matters

Pipelines fail. Networks drop. Servers restart. When a pipeline fails midway, the safest recovery is to re-run it from the beginning. If the pipeline is not idempotent, a re-run creates duplicates or leaves the data in an inconsistent state.

```
Non-idempotent pipeline re-run:
Run 1 (partial): inserted 5,000 rows → failed
Run 2 (full):    inserted 10,000 rows → 5,000 are duplicates

Idempotent pipeline re-run:
Run 1 (partial): inserted 5,000 rows → failed
Run 2 (full):    result = 10,000 correct rows, no duplicates
```

---

## Pattern 1 — TRUNCATE and Reload

The simplest idempotent pattern — clear the target completely before each load.

```sql
BEGIN;

TRUNCATE TABLE staging_orders;

INSERT INTO staging_orders
SELECT * FROM raw_orders
WHERE load_date = CURRENT_DATE;

COMMIT;
```

Re-running always produces the same result — the previous data is cleared first.

**Best for:** staging tables, full reloads, small-to-medium tables.

---

## Pattern 2 — DELETE and Reload by Partition

For large tables, delete only the affected partition before reloading it.

```sql
BEGIN;

-- Clear only today's data
DELETE FROM orders_warehouse
WHERE order_date = CURRENT_DATE;

-- Reload today's data
INSERT INTO orders_warehouse
SELECT * FROM staging_orders
WHERE order_date = CURRENT_DATE;

COMMIT;
```

Re-running always replaces today's partition with the current source data.

**Best for:** date-partitioned tables in incremental pipelines.

---

## Pattern 3 — MERGE (Upsert)

`MERGE` is inherently idempotent — it updates existing rows and inserts new ones, so re-running produces the same final state.

```sql
MERGE INTO customers AS target
USING staging_customers AS source
    ON target.customer_id = source.customer_id

WHEN MATCHED THEN
    UPDATE SET
        target.email      = source.email,
        target.segment    = source.segment,
        target.updated_at = CURRENT_TIMESTAMP

WHEN NOT MATCHED THEN
    INSERT (customer_id, first_name, email, segment, created_at)
    VALUES (source.customer_id, source.first_name, source.email,
            source.segment, CURRENT_TIMESTAMP);
```

Re-running applies the same updates and skips already-inserted rows.

**Best for:** dimension tables, entities that can be both inserted and updated.

---

## Pattern 4 — INSERT WHERE NOT EXISTS

Only insert rows that do not already exist in the target.

```sql
INSERT INTO orders_warehouse (order_id, customer_id, amount, order_date)
SELECT s.order_id, s.customer_id, s.amount, s.order_date
FROM staging_orders s
WHERE NOT EXISTS (
    SELECT 1 FROM orders_warehouse t
    WHERE t.order_id = s.order_id
);
```

Re-running inserts only the rows that are genuinely new.

**Best for:** append-only tables, event logs, audit tables.

---

## Pattern 5 — INSERT ON CONFLICT (Upsert)

A cleaner, atomic alternative to `INSERT WHERE NOT EXISTS` in PostgreSQL.

```sql
-- PostgreSQL
INSERT INTO customers (customer_id, email, segment)
VALUES (42, 'jorge@example.com', 'Gold')
ON CONFLICT (customer_id)
DO UPDATE SET
    email     = EXCLUDED.email,
    segment   = EXCLUDED.segment,
    updated_at = CURRENT_TIMESTAMP;
```

```sql
-- Do nothing if the row already exists
INSERT INTO events (event_id, user_id, event_time)
VALUES (1001, 42, CURRENT_TIMESTAMP)
ON CONFLICT (event_id)
DO NOTHING;
```

---

## Pattern 6 — CREATE TABLE IF NOT EXISTS

DDL scripts should be idempotent too — safe to run multiple times without errors.

```sql
CREATE TABLE IF NOT EXISTS customers (
    customer_id INT PRIMARY KEY,
    email       VARCHAR(255) NOT NULL
);

DROP TABLE IF EXISTS temp_staging;
```

---

## Making Scripts Fully Idempotent

A complete idempotent pipeline script:

```sql
BEGIN;

-- 1. Clear the load window
DELETE FROM orders_warehouse
WHERE load_date = CURRENT_DATE;

-- 2. Load from staging
INSERT INTO orders_warehouse (order_id, customer_id, amount, order_date, load_date)
SELECT order_id, customer_id, amount, order_date, CURRENT_DATE
FROM staging_orders
WHERE load_date = CURRENT_DATE;

-- 3. Update watermark
UPDATE pipeline_watermarks
SET
    last_loaded_at = CURRENT_TIMESTAMP,
    status         = 'success'
WHERE table_name = 'orders_warehouse';

COMMIT;
```

---

## Common Non-Idempotent Patterns to Avoid

```sql
-- ❌ INSERT without duplicate check — creates duplicates on re-run
INSERT INTO orders_warehouse SELECT * FROM staging_orders;

-- ❌ UPDATE without condition — updates rows that were already correct
UPDATE customers SET updated_at = CURRENT_TIMESTAMP;

-- ❌ CREATE TABLE without IF NOT EXISTS — fails on re-run
CREATE TABLE temp_results (...);
```

---

## Idempotency Checklist

Before deploying a pipeline, verify:

- [ ] Re-running produces the same final row count and values
- [ ] No duplicate rows are created on re-run
- [ ] DDL scripts use `IF EXISTS` / `IF NOT EXISTS`
- [ ] DML scripts use `MERGE`, `ON CONFLICT`, or `DELETE + INSERT` patterns
- [ ] Watermarks are updated atomically with the data load
- [ ] The pipeline is tested by running it twice in sequence

---

## Best Practices

- Design every pipeline to be idempotent from the start — retrofitting is harder
- Use `TRUNCATE + INSERT` for full loads, `DELETE + INSERT` for partition-based incremental loads
- Use `MERGE` or `INSERT ON CONFLICT` for upsert patterns
- Use `IF EXISTS` / `IF NOT EXISTS` in all DDL scripts
- Wrap the load and watermark update in the same transaction — they succeed or fail together
- Test idempotency explicitly — run the pipeline twice and compare the results
