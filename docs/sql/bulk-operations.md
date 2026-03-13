---
title: Bulk Operations
description: Loading and manipulating large volumes of data efficiently in SQL
tags: [sql, dml, bulk, performance]
---

# Bulk Operations

Bulk operations load or manipulate large volumes of data in a single operation rather than row by row. They are the backbone of ETL pipelines, data migrations, and large-scale transformations — where performance matters as much as correctness.

---

## Why Bulk Operations Matter

Row-by-row processing is the slowest way to move data. Each individual statement has overhead — parsing, planning, network round trips, and logging. Bulk operations reduce this overhead dramatically by processing many rows in a single pass.

| Approach | 1M rows | Notes |
|---|---|---|
| Single-row INSERT loop | Very slow | One transaction per row |
| Multi-row INSERT | Fast | One transaction, many rows |
| `INSERT ... SELECT` | Very fast | Set-based, no application overhead |
| Bulk load (COPY/BCP) | Fastest | Bypasses normal SQL overhead |

---

## Multi-Row INSERT

The simplest bulk improvement — insert many rows in a single statement.

```sql
INSERT INTO products (product_id, name, price, category)
VALUES
    (1, 'Laptop',   999.99, 'Electronics'),
    (2, 'Monitor',  349.99, 'Electronics'),
    (3, 'Keyboard',  79.99, 'Accessories'),
    (4, 'Mouse',     39.99, 'Accessories');
```

---

## INSERT ... SELECT

Load data from another table or query in a single set-based operation. No application layer involved.

```sql
-- Load from staging to production
INSERT INTO orders (order_id, customer_id, amount, status, created_at)
SELECT order_id, customer_id, amount, status, created_at
FROM staging_orders
WHERE is_valid = TRUE
  AND load_date = CURRENT_DATE;
```

```sql
-- Transform and load in one step
INSERT INTO dim_customer (customer_key, full_name, email, country, load_date)
SELECT
    customer_id,
    TRIM(first_name) || ' ' || TRIM(last_name),
    LOWER(email),
    UPPER(country_code),
    CURRENT_DATE
FROM staging_customers
WHERE email IS NOT NULL
  AND country_code IS NOT NULL;
```

---

## CREATE TABLE AS SELECT (CTAS)

Create a new table and populate it from a query in one statement. Common for building derived tables and snapshots.

```sql
CREATE TABLE orders_2024 AS
SELECT *
FROM orders
WHERE EXTRACT(YEAR FROM created_at) = 2024;
```

```sql
-- Build a summary table
CREATE TABLE monthly_revenue AS
SELECT
    DATE_TRUNC('month', order_date) AS month,
    country,
    SUM(amount)  AS total_revenue,
    COUNT(*)     AS total_orders
FROM orders
GROUP BY 1, 2;
```

---

## Bulk Load with COPY (PostgreSQL)

`COPY` is the fastest way to load data from flat files into PostgreSQL — it bypasses most of the SQL overhead.

```sql
-- Load from a CSV file
COPY customers (customer_id, first_name, last_name, email)
FROM '/data/customers.csv'
WITH (FORMAT CSV, HEADER TRUE, DELIMITER ',');

-- Export to a CSV file
COPY orders TO '/data/orders_export.csv'
WITH (FORMAT CSV, HEADER TRUE);
```

---

## Bulk Insert Patterns by Platform

| Platform | Bulk Load Tool | Notes |
|---|---|---|
| PostgreSQL | `COPY` | Fastest option, file-based |
| SQL Server | `BULK INSERT` / BCP | File-based bulk load |
| MySQL | `LOAD DATA INFILE` | File-based bulk load |
| Snowflake | `COPY INTO` | Loads from cloud storage (S3, Azure) |
| BigQuery | `LOAD DATA` | Loads from GCS |

```sql
-- SQL Server
BULK INSERT orders
FROM 'C:\data\orders.csv'
WITH (FORMAT = 'CSV', FIRSTROW = 2, FIELDTERMINATOR = ',');

-- Snowflake
COPY INTO orders
FROM @my_stage/orders.csv
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1);
```

---

## Batching Large Operations

When updating or deleting millions of rows, doing it in one statement can lock the table and fill transaction logs. Split into batches instead.

```sql
-- Delete in batches of 10,000 rows
DELETE FROM logs
WHERE created_at < '2023-01-01'
  AND ctid IN (
      SELECT ctid FROM logs
      WHERE created_at < '2023-01-01'
      LIMIT 10000
  );
-- Repeat until 0 rows deleted
```

```sql
-- PostgreSQL batch delete loop (using DO block)
DO $$
DECLARE
    deleted INT;
BEGIN
    LOOP
        DELETE FROM logs
        WHERE id IN (
            SELECT id FROM logs
            WHERE created_at < '2023-01-01'
            LIMIT 10000
        );

        GET DIAGNOSTICS deleted = ROW_COUNT;
        EXIT WHEN deleted = 0;

        PERFORM pg_sleep(0.1); -- brief pause between batches
    END LOOP;
END $$;
```

---

## Performance Tips for Bulk Operations

- **Disable indexes before loading, rebuild after** — indexes slow down inserts significantly on large loads
- **Use transactions** — wrap bulk loads in `BEGIN / COMMIT` to avoid partial loads
- **Drop and recreate constraints** on staging tables when doing full reloads
- **Use `TRUNCATE` instead of `DELETE`** to clear tables before a full reload
- **Avoid triggers on bulk load tables** — they fire row by row and eliminate bulk performance gains

```sql
-- Efficient full reload pattern
BEGIN;

TRUNCATE TABLE staging_orders;

INSERT INTO staging_orders
SELECT * FROM raw_orders WHERE load_date = CURRENT_DATE;

-- Validate
-- Promote
INSERT INTO orders SELECT * FROM staging_orders;

COMMIT;
```

---

## Best Practices

- Use `INSERT ... SELECT` over row-by-row inserts whenever loading from another table
- Use `CTAS` to build snapshots and derived tables in one step
- Use platform bulk load tools (`COPY`, `BULK INSERT`) for file-based loads
- Batch large `DELETE` and `UPDATE` operations to avoid locking and log overflow
- Always wrap bulk loads in transactions — partial loads are worse than no load
- Validate row counts and checksums after bulk loads before committing
