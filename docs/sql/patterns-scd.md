---
title: Slowly Changing Dimensions
description: SCD Type 1, Type 2, and Type 3 patterns for tracking historical changes in SQL
tags: [sql, patterns, scd, data-warehouse]
---

# Slowly Changing Dimensions

A Slowly Changing Dimension (SCD) is a dimension table in a data warehouse where attribute values change over time — but slowly and irregularly. Customer segments change, product categories are renamed, employees move between departments. SCDs define how to handle these changes: overwrite, preserve history, or track the transition.

!!! note
    SCD Type 1 is implemented with `MERGE` — see [MERGE / UPSERT](merge.md) for the full syntax. The latest record pattern (reading the current state of a Type 2 dimension) is covered in [Latest Record per Group](patterns-latest-record.md).

---

## Why SCDs Matter

In a data warehouse, accuracy of historical reporting depends on knowing what was true at the time — not just what is true now.

```
Customer Jorge: segment = 'Standard' in 2023, 'Gold' in 2024

Without SCD: all of Jorge's 2023 orders look like Gold orders
With SCD Type 2: 2023 orders correctly show Standard, 2024 show Gold
```

---

## SCD Type 1 — Overwrite

The simplest approach — update the record with the new value. No history is kept.

```
Before: customer_id=1, segment='Standard'
After:  customer_id=1, segment='Gold'
← 'Standard' is gone
```

```sql
-- MERGE implements SCD Type 1
MERGE INTO dim_customer AS target
USING staging_customer AS source
    ON target.customer_id = source.customer_id

WHEN MATCHED AND (
    target.segment  <> source.segment OR
    target.country  <> source.country
) THEN
    UPDATE SET
        target.segment    = source.segment,
        target.country    = source.country,
        target.updated_at = CURRENT_TIMESTAMP

WHEN NOT MATCHED THEN
    INSERT (customer_id, first_name, segment, country, created_at)
    VALUES (source.customer_id, source.first_name, source.segment,
            source.country, CURRENT_TIMESTAMP);
```

**When to use:** attributes where history is not needed — fixing data errors, updating contact details, correcting typos.

---

## SCD Type 2 — Add a New Row

A new row is inserted for each change. The old row is closed with an expiry date. Full history is preserved.

```
customer_key  customer_id  segment    valid_from   valid_to     is_current
1             42           Standard   2023-01-01   2024-03-15   FALSE
2             42           Gold       2024-03-15   9999-12-31   TRUE
```

```sql
-- Step 1: expire rows that have changed — JOIN approach for performance
UPDATE dim_customer
SET
    valid_to   = CURRENT_DATE,
    is_current = FALSE
FROM staging_customer s
WHERE dim_customer.customer_id = s.customer_id
  AND dim_customer.is_current  = TRUE
  AND (
      dim_customer.segment <> s.segment OR
      dim_customer.country <> s.country
  );

-- Step 2: insert new rows for changed and new records
INSERT INTO dim_customer (
    customer_id, first_name, segment, country,
    valid_from, valid_to, is_current
)
SELECT
    s.customer_id,
    s.first_name,
    s.segment,
    s.country,
    CURRENT_DATE,
    DATE '9999-12-31',
    TRUE
FROM staging_customer s
WHERE NOT EXISTS (
    SELECT 1 FROM dim_customer d
    WHERE d.customer_id = s.customer_id
      AND d.is_current  = TRUE
      AND d.segment     = s.segment
      AND d.country     = s.country
);
```

!!! note
    `DATE '9999-12-31'` is ANSI SQL syntax for a date literal. SQL Server accepts `'9999-12-31'` without the `DATE` keyword. Both forms work in PostgreSQL. Use whichever form your platform requires — the sentinel value itself (`9999-12-31`) is the convention that matters.

!!! tip
    The UPDATE in Step 1 uses a JOIN-based approach (`FROM staging_customer`) rather than correlated subqueries — this is significantly faster on large dimension tables. See [UPDATE](update.md) for the portability note on this syntax.

### Querying SCD Type 2

```sql
-- Current state
SELECT * FROM dim_customer WHERE is_current = TRUE;

-- State at a specific point in time
SELECT *
FROM dim_customer
WHERE customer_id = 42
  AND valid_from <= '2023-06-01'
  AND valid_to   >  '2023-06-01';

-- Join fact table to dimension at the time of the transaction
SELECT
    f.order_date,
    f.amount,
    d.segment
FROM fact_orders f
JOIN dim_customer d
    ON f.customer_key = d.customer_key
    AND f.order_date >= d.valid_from
    AND f.order_date <  d.valid_to;
```

**When to use:** attributes where historical accuracy matters for reporting — customer segment, product category, employee department.

---

## SCD Type 3 — Add a Column

Add a column to track the previous value. Only the current and one prior value are stored — not full history.

```
customer_id  segment_current  segment_previous  segment_changed_at
42           Gold             Standard          2024-03-15
```

```sql
-- Schema
ALTER TABLE dim_customer
ADD COLUMN segment_previous   VARCHAR(50),
ADD COLUMN segment_changed_at TIMESTAMP;

-- Update when segment changes
UPDATE dim_customer
SET
    segment_previous   = segment_current,
    segment_current    = source.segment,
    segment_changed_at = CURRENT_TIMESTAMP
FROM staging_customer source
WHERE dim_customer.customer_id = source.customer_id
  AND dim_customer.segment_current <> source.segment;
```

**When to use:** when you only need to know the previous value (e.g. "what was the customer's previous segment?") and full history is not required.

---

## SCD Comparison

| | Type 1 | Type 2 | Type 3 |
|---|---|---|---|
| History preserved | ❌ | ✅ Full | ⚠️ One prior value |
| Storage | Low | High | Low |
| Query complexity | Simple | Medium | Simple |
| Supports point-in-time queries | ❌ | ✅ | ❌ |
| Best for | Corrections, non-critical attributes | Auditable history, reporting accuracy | Simple before/after tracking |

---

## SCD Type 2 — Table Structure

```sql
CREATE TABLE dim_customer (
    customer_key  INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- surrogate key
    customer_id   INT          NOT NULL,   -- business / natural key
    first_name    VARCHAR(100),
    segment       VARCHAR(50),
    country       CHAR(2),
    valid_from    DATE         NOT NULL DEFAULT CURRENT_DATE,
    valid_to      DATE         NOT NULL DEFAULT DATE '9999-12-31',
    is_current    BOOLEAN      NOT NULL DEFAULT TRUE
);

-- Index for point-in-time lookups
CREATE INDEX idx_dim_customer_id_current
ON dim_customer (customer_id, is_current);
```

---

## Best Practices

- Use SCD Type 1 by default — only add Type 2 complexity when historical accuracy is a reporting requirement
- Always use a surrogate key as the dimension primary key — the natural key repeats across Type 2 rows
- Use `is_current = TRUE` for the current record and a `valid_to = '9999-12-31'` sentinel date
- Index `(customer_id, is_current)` for fast current-record lookups
- Index `(customer_id, valid_from, valid_to)` for point-in-time joins
- Use a JOIN-based UPDATE for expiring rows — avoid correlated subqueries on large dimension tables
- Document which attributes are tracked as Type 2 — not every column needs history