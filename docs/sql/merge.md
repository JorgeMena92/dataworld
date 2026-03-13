---
title: MERGE / UPSERT
description: Insert or update in a single statement using ANSI SQL MERGE
tags: [sql, dml, merge, upsert]
---

# MERGE / UPSERT

`MERGE` combines insert and update into a single atomic statement. It compares a source dataset against a target table, then inserts new rows, updates matching ones, and optionally deletes rows that no longer exist in the source.

It is the standard pattern for loading dimension tables, synchronizing datasets, and implementing slowly changing dimensions (SCDs).

---

## Basic Syntax

```sql
MERGE INTO target_table AS target
USING source_table AS source
    ON target.key = source.key

WHEN MATCHED THEN
    UPDATE SET
        target.column1 = source.column1,
        target.column2 = source.column2

WHEN NOT MATCHED THEN
    INSERT (key, column1, column2)
    VALUES (source.key, source.column1, source.column2);
```

---

## Insert or Update — Classic Upsert

```sql
MERGE INTO customers AS target
USING staging_customers AS source
    ON target.customer_id = source.customer_id

WHEN MATCHED THEN
    UPDATE SET
        target.email      = source.email,
        target.country    = source.country,
        target.updated_at = CURRENT_TIMESTAMP

WHEN NOT MATCHED THEN
    INSERT (customer_id, first_name, last_name, email, country, created_at)
    VALUES (source.customer_id, source.first_name, source.last_name,
            source.email, source.country, CURRENT_TIMESTAMP);
```

---

## MERGE with Delete

Delete target rows that no longer exist in the source.

```sql
MERGE INTO products AS target
USING staging_products AS source
    ON target.product_id = source.product_id

WHEN MATCHED THEN
    UPDATE SET
        target.price      = source.price,
        target.updated_at = CURRENT_TIMESTAMP

WHEN NOT MATCHED BY TARGET THEN
    INSERT (product_id, name, price)
    VALUES (source.product_id, source.name, source.price)

WHEN NOT MATCHED BY SOURCE THEN
    DELETE;
```

!!! warning
    `WHEN NOT MATCHED BY SOURCE THEN DELETE` removes every target row not present in the source. Make sure the source is complete before running this — a partial load will delete valid data.

---

## Conditional MERGE

Add conditions to `WHEN MATCHED` to only update rows that have actually changed.

```sql
MERGE INTO customers AS target
USING staging_customers AS source
    ON target.customer_id = source.customer_id

WHEN MATCHED AND (
    target.email   <> source.email OR
    target.country <> source.country
) THEN
    UPDATE SET
        target.email      = source.email,
        target.country    = source.country,
        target.updated_at = CURRENT_TIMESTAMP

WHEN NOT MATCHED THEN
    INSERT (customer_id, first_name, email, country, created_at)
    VALUES (source.customer_id, source.first_name, source.email,
            source.country, CURRENT_TIMESTAMP);
```

This avoids unnecessary writes when the data has not changed — important for performance and for tracking `updated_at` accurately.

---

## MERGE for Slowly Changing Dimensions (SCD Type 1)

SCD Type 1 overwrites the existing record with the latest values — no history is kept.

```sql
MERGE INTO dim_customer AS target
USING staging_customer AS source
    ON target.customer_key = source.customer_id

WHEN MATCHED AND (
    target.email  <> source.email OR
    target.segment <> source.segment
) THEN
    UPDATE SET
        target.email      = source.email,
        target.segment    = source.segment,
        target.updated_at = CURRENT_TIMESTAMP

WHEN NOT MATCHED THEN
    INSERT (customer_key, first_name, email, segment, created_at)
    VALUES (source.customer_id, source.first_name, source.email,
            source.segment, CURRENT_TIMESTAMP);
```

---

## MERGE Using a Subquery as Source

The source does not have to be a table — it can be any query.

```sql
MERGE INTO monthly_summary AS target
USING (
    SELECT
        DATE_TRUNC('month', order_date) AS month,
        SUM(amount)                     AS total_revenue,
        COUNT(*)                        AS total_orders
    FROM orders
    WHERE order_date >= DATE_TRUNC('month', CURRENT_DATE)
    GROUP BY 1
) AS source
    ON target.month = source.month

WHEN MATCHED THEN
    UPDATE SET
        target.total_revenue = source.total_revenue,
        target.total_orders  = source.total_orders

WHEN NOT MATCHED THEN
    INSERT (month, total_revenue, total_orders)
    VALUES (source.month, source.total_revenue, source.total_orders);
```

---

## Vendor Notes

`MERGE` is ANSI SQL (SQL:2003) but implementation details vary.

| Feature | ANSI SQL | SQL Server | PostgreSQL | MySQL |
|---|---|---|---|---|
| `MERGE` statement | ✅ | ✅ | ✅ (15+) | ✅ (8.0.31+) |
| `NOT MATCHED BY SOURCE` | — | ✅ | ❌ | ❌ |
| Alternative upsert | — | — | `INSERT ... ON CONFLICT` | `INSERT ... ON DUPLICATE KEY` |

```sql
-- PostgreSQL alternative (pre-15)
INSERT INTO customers (customer_id, email)
VALUES (42, 'jorge@example.com')
ON CONFLICT (customer_id)
DO UPDATE SET email = EXCLUDED.email;

-- MySQL alternative
INSERT INTO customers (customer_id, email)
VALUES (42, 'jorge@example.com')
ON DUPLICATE KEY UPDATE email = VALUES(email);
```

---

## Best Practices

- Always use a unique key in the `ON` clause — matching on a non-unique key causes unpredictable results
- Add a condition to `WHEN MATCHED` to skip rows that have not changed — avoids unnecessary writes
- Use a subquery or CTE as the source when the data needs transformation before merging
- Test with `SELECT` from the source before running `MERGE` in production
- Be cautious with `WHEN NOT MATCHED BY SOURCE THEN DELETE` — verify the source is complete
- Wrap `MERGE` in a transaction for multi-table pipelines
