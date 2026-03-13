---
title: Partitioning
description: Table partitioning strategies for performance and manageability at scale
tags: [sql, performance, partitioning]
---

# Partitioning

Partitioning divides a large table into smaller, more manageable physical segments called partitions — while keeping it accessible as a single logical table. Queries that filter on the partition key only scan the relevant partitions, skipping the rest — a process called **partition pruning**.

---

## Why Partition?

| Benefit | Description |
|---|---|
| Query performance | Queries scan only relevant partitions, not the whole table |
| Maintenance | Archive, truncate, or drop old partitions without touching the rest |
| Parallel processing | Each partition can be processed independently |
| Storage management | Move older partitions to cheaper storage tiers |

Partitioning becomes valuable when:
- Tables have hundreds of millions or billions of rows
- Most queries filter on a consistent column (date, region, status)
- You need to archive or purge old data regularly

---

## Partitioning Strategies

### Range Partitioning

Divide rows based on ranges of a column's values. Most common for date-based data.

```sql
-- PostgreSQL — range partitioning by year
CREATE TABLE orders (
    order_id   INT,
    order_date DATE NOT NULL,
    amount     DECIMAL(10, 2)
) PARTITION BY RANGE (order_date);

-- Create partitions
CREATE TABLE orders_2022 PARTITION OF orders
    FOR VALUES FROM ('2022-01-01') TO ('2023-01-01');

CREATE TABLE orders_2023 PARTITION OF orders
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

Queries filtering on `order_date` will only scan the relevant partition:

```sql
-- Only scans orders_2024
SELECT * FROM orders WHERE order_date >= '2024-01-01';
```

### List Partitioning

Divide rows based on discrete values — useful for region, country, or category.

```sql
-- PostgreSQL — list partitioning by region
CREATE TABLE sales (
    sale_id INT,
    region  VARCHAR(20) NOT NULL,
    amount  DECIMAL(10, 2)
) PARTITION BY LIST (region);

CREATE TABLE sales_latam PARTITION OF sales
    FOR VALUES IN ('Peru', 'Chile', 'Argentina', 'Colombia');

CREATE TABLE sales_emea PARTITION OF sales
    FOR VALUES IN ('Spain', 'France', 'Germany', 'UK');
```

### Hash Partitioning

Distribute rows evenly across a fixed number of partitions using a hash function. Used when there is no natural range or list key.

```sql
-- PostgreSQL — hash partitioning into 4 buckets
CREATE TABLE events (
    event_id   INT,
    user_id    INT NOT NULL,
    event_type VARCHAR(50)
) PARTITION BY HASH (user_id);

CREATE TABLE events_0 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE events_1 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE events_2 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE events_3 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

---

## Partition Pruning

Partition pruning is when the optimizer eliminates partitions that cannot contain matching rows — based on the `WHERE` clause.

```sql
-- With partition pruning — only scans orders_2024
EXPLAIN SELECT * FROM orders WHERE order_date = '2024-06-15';

-- Without pruning — scans all partitions (partition key not in filter)
EXPLAIN SELECT * FROM orders WHERE amount > 1000;
```

!!! tip
    For partition pruning to work, the `WHERE` clause must reference the partition key directly — without functions or expressions that obscure the value.

```sql
-- Pruning works ✅
WHERE order_date = '2024-06-15'
WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01'

-- Pruning does NOT work ❌
WHERE EXTRACT(YEAR FROM order_date) = 2024  -- function prevents pruning
```

---

## Adding and Dropping Partitions

One of the biggest operational advantages of partitioning — you can add new partitions or archive old ones without touching existing data.

```sql
-- Add a new partition for 2025
CREATE TABLE orders_2025 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

-- Drop an old partition (fast — no row-by-row delete)
DROP TABLE orders_2022;

-- Detach a partition (keeps the table, removes it from the parent)
ALTER TABLE orders DETACH PARTITION orders_2022;
-- orders_2022 now exists as a standalone table for archiving
```

Dropping a partition is nearly instant — far faster than `DELETE FROM orders WHERE order_date < '2023-01-01'` on a billion-row table.

---

## Indexes on Partitioned Tables

In PostgreSQL, indexes created on a partitioned table are automatically created on all partitions.

```sql
-- Creates the index on all partitions
CREATE INDEX idx_orders_customer_id ON orders (customer_id);
```

You can also create indexes on individual partitions.

---

## Partitioning in Data Warehouses

Modern cloud data warehouses use partitioning as a core performance strategy.

```sql
-- Snowflake — cluster key (similar to partitioning)
CREATE TABLE orders (
    order_id   INT,
    order_date DATE,
    amount     DECIMAL(10, 2)
) CLUSTER BY (order_date);

-- BigQuery — partition by date
CREATE TABLE orders
PARTITION BY DATE(order_date)
AS SELECT * FROM source_orders;

-- BigQuery — query only hits the relevant partition
SELECT * FROM orders WHERE DATE(order_date) = '2024-06-15';
```

---

## Vendor Support

| Feature | PostgreSQL | SQL Server | MySQL | Snowflake | BigQuery |
|---|---|---|---|---|---|
| Range | ✅ | ✅ | ✅ | Cluster key | ✅ |
| List | ✅ | ✅ | ✅ | — | ✅ |
| Hash | ✅ | ✅ | ✅ | — | — |
| Partition pruning | ✅ | ✅ | ✅ | ✅ | ✅ |
| Attach / Detach | ✅ | ✅ (switch) | ❌ | — | — |

---

## Best Practices

- Partition on the column most commonly used in `WHERE` filters — usually a date column
- Always use the partition key directly in filters — avoid functions that prevent pruning
- Use range partitioning for time-series data — it maps naturally to data pipelines
- Drop or detach old partitions instead of deleting rows — it is dramatically faster
- Add new partitions before they are needed — do not wait until data arrives
- Index the partition key and frequently joined columns on each partition
- Monitor partition sizes — uneven distribution (skew) can defeat the purpose of partitioning
