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
    CONCAT(TRIM(first_name), ' ', TRIM(last_name)),
    LOWER(email),
    UPPER(country_code),
    CURRENT_DATE
FROM staging_customers
WHERE email IS NOT NULL
  AND country_code IS NOT NULL;
```

!!! note
    `CONCAT()` is used here instead of `||` for cross-platform portability — SQL Server does not support the `||` operator. See [Scalar Functions](scalar-functions.md).

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
    EXTRACT(YEAR  FROM order_date) AS order_year,
    EXTRACT(MONTH FROM order_date) AS order_month,
    country,
    SUM(amount) AS total_revenue,
    COUNT(*)    AS total_orders
FROM orders
GROUP BY
    EXTRACT(YEAR  FROM order_date),
    EXTRACT(MONTH FROM order_date),
    country;
```

!!! note
    `DATE_TRUNC` is a common alternative for truncating dates to month boundaries but is not ANSI SQL. The `EXTRACT(YEAR ...)` / `EXTRACT(MONTH ...)` approach above is fully portable. See [Scalar Functions](scalar-functions.md).

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

When updating or deleting millions of rows, doing it in one statement can lock the table and fill transaction logs. Split into batches instead using a standard primary key column.

```sql
-- Delete in batches of 10,000 rows using the primary key
DELETE FROM logs
WHERE log_id IN (
    SELECT log_id
    FROM logs
    WHERE created_at < '2023-01-01'
    FETCH FIRST 10000 ROWS ONLY
);
-- Repeat until 0 rows deleted
```

!!! note
    The example above uses `FETCH FIRST` (ANSI SQL) and a standard primary key column (`log_id`) for portability. Avoid platform-specific row identifiers like `ctid` (PostgreSQL) or `ROWID` (Oracle) — they are internal and not portable.

!!! note
    Automating the batch loop (repeat until 0 rows deleted) requires procedural SQL extensions like PL/pgSQL (PostgreSQL) or T-SQL (SQL Server) — these are not ANSI SQL. In portable pipelines, handle the loop logic in the orchestration layer or calling application.

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

-- Validate row count before promoting
SELECT COUNT(*) FROM staging_orders;

-- Promote to production
INSERT INTO orders SELECT * FROM staging_orders;

COMMIT;
```

---

## Best Practices

- Use `INSERT ... SELECT` over row-by-row inserts whenever loading from another table
- Use `CTAS` to build snapshots and derived tables in one step
- Use platform bulk load tools (`COPY`, `BULK INSERT`) for file-based loads
- Batch large `DELETE` and `UPDATE` operations using primary key ranges — avoid platform-specific row identifiers
- Always wrap bulk loads in transactions — partial loads are worse than no load
- Validate row counts and checksums after bulk loads before committing
- Use `CONCAT()` over `||` for string operations in cross-platform bulk transforms