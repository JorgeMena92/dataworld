---
title: Incremental Data Loading
description: Patterns for loading only new or changed data in SQL pipelines
tags: [sql, dml, incremental, etl]
---

# Incremental Data Loading

Incremental loading means loading only new or changed records since the last pipeline run — rather than reloading the entire dataset every time. It reduces processing time, resource usage, and load on source systems as data volumes grow.

---

## Full Load vs Incremental Load

| | Full Load | Incremental Load |
|---|---|---|
| What loads | Everything | Only new or changed rows |
| Complexity | Low | Medium |
| Performance | Slow on large tables | Fast |
| Data loss risk | Low (full reload) | Higher (depends on detection logic) |
| Best for | Small tables, reference data | Large fact tables, event logs |

---

## Incremental Detection Strategies

### 1. Timestamp-Based

The most common approach — filter rows where a date column is newer than the last load.

```sql
-- Load orders created since the last run
INSERT INTO orders_warehouse
SELECT *
FROM orders_source
WHERE created_at > (
    SELECT COALESCE(MAX(created_at), '1900-01-01')
    FROM orders_warehouse
);
```

```sql
-- Using a watermark table to track the last loaded timestamp
INSERT INTO orders_warehouse
SELECT *
FROM orders_source
WHERE created_at > (SELECT last_loaded_at FROM pipeline_watermarks WHERE table_name = 'orders');

-- Update the watermark after a successful load
UPDATE pipeline_watermarks
SET last_loaded_at = CURRENT_TIMESTAMP
WHERE table_name = 'orders';
```

!!! warning
    Timestamp-based detection misses rows with backdated timestamps or late-arriving data. Always include a small overlap window (e.g. last 24 hours) to catch late arrivals.

### 2. Sequence / ID-Based

Use an auto-incrementing ID to detect new rows. Simpler and more reliable than timestamps for append-only tables.

```sql
-- Load rows with an ID greater than the last loaded ID
INSERT INTO events_warehouse
SELECT *
FROM events_source
WHERE event_id > (
    SELECT COALESCE(MAX(event_id), 0)
    FROM events_warehouse
);
```

### 3. Change Data Capture (CDC)

CDC captures row-level changes (insert, update, delete) directly from the database transaction log. The SQL layer consumes the CDC output rather than querying the source table directly.

```sql
-- Consume CDC output table (SQL Server example)
SELECT *
FROM cdc.dbo_orders_CT
WHERE __$operation IN (1, 2, 4)  -- 1=delete, 2=insert, 4=update
  AND __$start_lsn > @last_lsn
ORDER BY __$start_lsn;
```

---

## Incremental Upsert Pattern

For tables where rows can be both inserted and updated, combine incremental detection with `MERGE`.

```sql
-- Step 1: load changed rows into staging
INSERT INTO staging_customers
SELECT *
FROM source_customers
WHERE updated_at > (
    SELECT COALESCE(MAX(updated_at), '1900-01-01')
    FROM customers_warehouse
);

-- Step 2: merge staging into the warehouse
MERGE INTO customers_warehouse AS target
USING staging_customers AS source
    ON target.customer_id = source.customer_id

WHEN MATCHED AND (
    target.email   <> source.email OR
    target.country <> source.country
) THEN
    UPDATE SET
        target.email      = source.email,
        target.country    = source.country,
        target.updated_at = source.updated_at

WHEN NOT MATCHED THEN
    INSERT (customer_id, first_name, email, country, created_at, updated_at)
    VALUES (source.customer_id, source.first_name, source.email,
            source.country, source.created_at, source.updated_at);
```

---

## Watermark Table

A watermark table tracks the state of each pipeline run — which tables were loaded and up to what point.

```sql
CREATE TABLE pipeline_watermarks (
    table_name      VARCHAR(100) PRIMARY KEY,
    last_loaded_at  TIMESTAMP,
    last_loaded_id  BIGINT,
    rows_loaded     INT,
    status          VARCHAR(20),
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Read the watermark before loading
SELECT last_loaded_at
FROM pipeline_watermarks
WHERE table_name = 'orders';

-- Update the watermark after a successful load
UPDATE pipeline_watermarks
SET
    last_loaded_at = CURRENT_TIMESTAMP,
    rows_loaded    = (SELECT COUNT(*) FROM staging_orders),
    status         = 'success',
    updated_at     = CURRENT_TIMESTAMP
WHERE table_name = 'orders';
```

---

## Handling Late-Arriving Data

Late-arriving data — records that arrive after the pipeline has already processed their time window — is one of the most common problems in incremental pipelines.

```sql
-- Use an overlap window to catch late arrivals
-- Instead of loading from exactly the last watermark...
WHERE created_at > (SELECT last_loaded_at FROM watermarks WHERE table_name = 'orders')

-- ...load from slightly before it (e.g. 1 hour overlap)
WHERE created_at > (SELECT last_loaded_at - INTERVAL '1 hour' FROM watermarks WHERE table_name = 'orders')
```

The overlap means some rows will be processed twice — handle duplicates with `MERGE` or deduplication logic downstream.

---

## Incremental Load with Partitioning

When the target table is partitioned by date, replace entire partitions rather than merging row by row.

```sql
-- Delete the partition for today
DELETE FROM orders_warehouse
WHERE DATE_TRUNC('day', created_at) = CURRENT_DATE;

-- Reload the full partition
INSERT INTO orders_warehouse
SELECT *
FROM orders_source
WHERE DATE_TRUNC('day', created_at) = CURRENT_DATE;
```

This pattern is atomic (delete + insert in one transaction), simple to reason about, and works well with date-partitioned tables in modern data warehouses.

---

## Best Practices

- Always store a watermark — never recalculate the load boundary from the target table alone
- Use an overlap window to handle late-arriving data
- Use `MERGE` for tables with updates, plain `INSERT` for append-only tables
- Validate row counts after each load — alert if the count is zero or anomalously low
- Test your incremental logic against a full reload periodically to verify consistency
- For high-volume tables, prefer partition replacement over row-level `MERGE` — it is simpler and faster
