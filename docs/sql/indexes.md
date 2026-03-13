---
title: Indexes
description: When and why to use indexes — strategy, selectivity, and tradeoffs
tags: [sql, performance, indexes]
---

# Indexes

Indexes are the primary tool for improving query performance. This page covers index strategy — when to add them, how to choose the right columns, and the tradeoffs involved.

> For index DDL syntax (CREATE INDEX, DROP INDEX, index types), see **Definition (DDL) → Indexes**.

---

## How Indexes Speed Up Queries

Without an index, the database reads every row in a table — a full table scan. With an index, it navigates a sorted structure directly to the matching rows.

```
Full table scan:  read 10,000,000 rows → return 5
Index seek:       navigate B-tree → find 5 row pointers → fetch 5 rows
```

The larger the table and the more selective the filter, the greater the benefit.

---

## Selectivity and Cardinality

**Selectivity** measures how much a filter reduces the result set. High selectivity = few rows returned = indexes help a lot.

**Cardinality** is the number of distinct values in a column.

| Column | Cardinality | Selectivity | Index useful? |
|---|---|---|---|
| `customer_id` (PK) | 1,000,000 | Very high | ✅ Excellent |
| `country` | 50 | Medium | ✅ Often |
| `status` | 3 | Low | ⚠️ Rarely |
| `is_active` | 2 | Very low | ❌ Usually not |

!!! tip
    Low-cardinality columns (boolean flags, small enums) rarely benefit from indexes because the database may still need to read most of the table. Use partial indexes instead for specific values.

---

## Queries That Benefit from Indexes

```sql
-- Equality filter on indexed column ✅
WHERE customer_id = 42

-- Range filter on indexed column ✅
WHERE created_at >= '2024-01-01'

-- Sort on indexed column ✅
ORDER BY customer_id

-- JOIN on indexed column ✅
JOIN customers ON orders.customer_id = customers.customer_id

-- Leading column of a composite index ✅
WHERE status = 'completed'  -- index on (status, created_at)
```

---

## Queries That Bypass Indexes

```sql
-- Function applied to indexed column ❌
WHERE UPPER(email) = 'JORGE@EXAMPLE.COM'
WHERE EXTRACT(YEAR FROM created_at) = 2024

-- Leading wildcard ❌
WHERE name LIKE '%jorge'

-- Implicit type mismatch ❌
WHERE customer_id = '42'  -- string vs int

-- NOT on indexed column (often) ❌
WHERE status != 'completed'

-- Low-selectivity filter ❌
WHERE is_active = TRUE  -- if 95% of rows are active
```

---

## Composite Index Column Order

In a composite index, column order matters. The index is most effective when queries filter on the leftmost columns first.

```sql
CREATE INDEX idx_orders_status_date ON orders (status, created_at);

-- Uses the index ✅
WHERE status = 'completed'
WHERE status = 'completed' AND created_at >= '2024-01-01'

-- Does NOT use the index ❌
WHERE created_at >= '2024-01-01'  -- leading column missing
```

**Rule of thumb:** put equality columns first, range columns last.

```sql
-- Good composite index order for this query:
-- WHERE status = 'completed' AND created_at >= '2024-01-01' ORDER BY created_at
CREATE INDEX idx_orders_status_date ON orders (status, created_at);
```

---

## Partial Indexes

Index only the rows that matter — smaller, faster, more targeted.

```sql
-- Only index pending orders (the ones being queried most)
CREATE INDEX idx_orders_pending ON orders (created_at)
WHERE status = 'pending';

-- Only index active customers
CREATE INDEX idx_customers_active_email ON customers (email)
WHERE is_active = TRUE;
```

Partial indexes are ideal when:
- Most queries target a specific subset of rows
- One value dominates the column (e.g. 95% of orders are `completed`)

---

## Index-Only Scans

When a query only needs columns that are all in the index, the database can answer it without touching the table at all — an index-only scan.

```sql
-- If index is on (customer_id, amount):
SELECT customer_id, amount FROM orders WHERE customer_id = 42;
-- → Index-only scan — no table access needed
```

Include frequently selected columns in composite indexes to enable this optimization.

---

## The Cost of Indexes

Indexes are not free. Every index must be maintained on every `INSERT`, `UPDATE`, and `DELETE`.

| Operation | Impact |
|---|---|
| `SELECT` | Faster (if index is used) |
| `INSERT` | Slower — index must be updated |
| `UPDATE` | Slower — index updated if indexed column changes |
| `DELETE` | Slower — index entry must be removed |
| Storage | Each index consumes disk space |

!!! warning
    Over-indexing is a real problem. A table with 15 indexes on it will have very slow writes. Add indexes based on observed query patterns — not speculatively.

---

## When to Add an Index

Add an index when:
- A column is frequently used in `WHERE`, `JOIN ON`, or `ORDER BY`
- The column has high cardinality (many distinct values)
- Query performance is measurably slow and `EXPLAIN` shows a full table scan
- The table is large enough that a scan is actually expensive

Do not add an index when:
- The table is small (full scan is fast enough)
- The column has very low cardinality
- The table has very high write volume and the index would slow writes significantly
- The index would duplicate an existing one

---

## Maintaining Indexes

Indexes can become fragmented over time on tables with heavy write activity — degrading performance.

```sql
-- PostgreSQL — rebuild an index
REINDEX INDEX idx_orders_customer_id;
REINDEX TABLE orders;  -- rebuild all indexes on the table

-- SQL Server — rebuild
ALTER INDEX idx_orders_customer_id ON orders REBUILD;

-- SQL Server — reorganize (lighter, online operation)
ALTER INDEX idx_orders_customer_id ON orders REORGANIZE;
```

---

## Finding Unused Indexes

Unused indexes waste storage and slow down writes without helping reads.

```sql
-- PostgreSQL — find indexes with zero scans
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan AS times_used
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY tablename, indexname;
```

---

## Best Practices

- Index columns used in `WHERE`, `JOIN`, and `ORDER BY` on large tables
- Put equality columns before range columns in composite indexes
- Use partial indexes for queries that consistently target a subset of rows
- Monitor for unused indexes and drop them — they cost writes with no read benefit
- Disable indexes before bulk loads, rebuild after — dramatically speeds up large inserts
- Use `EXPLAIN ANALYZE` to verify that queries actually use the indexes you created
- Never add indexes speculatively — add them in response to measured slow queries
