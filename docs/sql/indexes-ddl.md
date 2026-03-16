---
title: Indexes
description: Creating, managing, and understanding indexes as database objects
tags: [sql, ddl, indexes]
---

# Indexes

An index is a database object that stores a sorted reference to data in a table, allowing the database engine to find rows without scanning the entire table. Indexes are created and managed with DDL commands.

!!! note
    This page covers indexes as **DDL objects** — what they are, how to create and manage them. For query optimization strategy and execution plans, see [Performance → Indexes](indexes.md).

---

## What Is an Index?

Without an index, the database scans every row in a table to find matches — a full table scan. An index provides a shortcut — a pre-sorted structure that points directly to the matching rows.

```
Table scan (no index):  read all 10,000,000 rows → find 5 matches
Index lookup:           read index → find 5 row pointers → fetch 5 rows
```

---

## CREATE INDEX

```sql
-- Basic index on a single column
CREATE INDEX idx_orders_customer_id
ON orders (customer_id);

-- Index with explicit sort order
CREATE INDEX idx_orders_created_at
ON orders (created_at DESC);
```

---

## UNIQUE Index

Enforces uniqueness on a column while also providing index performance.

```sql
CREATE UNIQUE INDEX idx_customers_email
ON customers (email);
```

A `UNIQUE` constraint automatically creates a unique index behind the scenes. Creating a unique index explicitly gives you more control over its name and options.

---

## Composite Index

An index on multiple columns — useful for queries that filter or sort on more than one column.

```sql
CREATE INDEX idx_orders_status_date
ON orders (status, created_at);
```

Column order matters in a composite index. The index supports queries that filter on:
- `status` alone
- `status` AND `created_at`

But **not** on `created_at` alone — the leading column must be present.

```sql
-- Uses the index (leading column present)
WHERE status = 'completed'
WHERE status = 'completed' AND created_at >= '2024-01-01'

-- Does NOT use the index (leading column missing)
WHERE created_at >= '2024-01-01'
```

---

## Partial Index

An index that covers only a subset of rows — based on a condition. Smaller and faster than a full index when queries only target a specific subset.

```sql
-- PostgreSQL — index only on active customers
CREATE INDEX idx_customers_active_email
ON customers (email)
WHERE is_active = TRUE;

-- Useful when most queries filter on a specific value
CREATE INDEX idx_orders_pending
ON orders (created_at)
WHERE status = 'pending';
```

!!! note
    Partial indexes are supported in PostgreSQL, SQLite, and Snowflake. SQL Server supports the same concept under the name **filtered indexes** — the syntax is identical:
    ```sql
    -- SQL Server filtered index
    CREATE INDEX idx_orders_pending
    ON orders (created_at)
    WHERE status = 'pending';
    ```
    MySQL does not support partial or filtered indexes.

---

## DROP INDEX

```sql
-- ANSI SQL
DROP INDEX idx_orders_customer_id;

-- Vendor extension — supported in PostgreSQL and MySQL, not in SQL Server
DROP INDEX IF EXISTS idx_orders_customer_id;

-- SQL Server — requires table name, no IF EXISTS support
DROP INDEX idx_orders_customer_id ON orders;

-- MySQL
DROP INDEX idx_orders_customer_id ON orders;
```

---

## Index Types by Database

Most databases support multiple index types beyond the default B-tree.

| Type | Use Case | Support |
|---|---|---|
| B-tree | General purpose — equality and range queries | All databases |
| Hash | Equality lookups only — faster for exact matches | PostgreSQL, MySQL |
| GIN | Full-text search, arrays, JSONB | PostgreSQL |
| GiST | Geometric data, full-text | PostgreSQL |
| Columnstore | Analytical queries on large datasets | SQL Server, Snowflake |
| Bitmap | Low-cardinality columns in analytical workloads | Oracle |

```sql
-- PostgreSQL — create a hash index
CREATE INDEX idx_customers_id_hash
ON customers USING HASH (customer_id);
```

---

## Listing Indexes

There is no ANSI SQL standard for listing indexes — `INFORMATION_SCHEMA` does not cover index metadata. Use platform-specific system catalog queries instead.

```sql
-- PostgreSQL
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'orders';

-- SQL Server
SELECT name, type_desc
FROM sys.indexes
WHERE object_id = OBJECT_ID('orders');

-- MySQL
SHOW INDEX FROM orders;
```

---

## Indexes and Constraints

Primary key and unique constraints automatically create indexes.

```sql
-- This PRIMARY KEY implicitly creates an index on customer_id
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,  -- index created automatically
    email VARCHAR(255) UNIQUE     -- index created automatically
);
```

---

## Best Practices

- Index columns used frequently in `WHERE`, `JOIN ON`, and `ORDER BY`
- Name indexes consistently — `idx_tablename_columnname`
- Use composite indexes for multi-column filters — put the most selective column first
- Use partial indexes when queries consistently target a subset of rows
- Do not over-index — every index slows down `INSERT`, `UPDATE`, and `DELETE`
- Drop unused indexes — they consume space and add write overhead with no read benefit
- Rebuild or reorganize indexes periodically on tables with heavy write activity

!!! note
    For query execution analysis and index tuning strategies, see [Performance → Indexes](indexes.md).