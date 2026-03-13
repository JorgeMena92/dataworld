---
title: Execution Plans
description: Reading and interpreting query execution plans with EXPLAIN and EXPLAIN ANALYZE
tags: [sql, performance, execution-plans]
---

# Execution Plans

An execution plan shows exactly how the database engine will execute a query — which indexes it uses, how it joins tables, how many rows it processes at each step, and where the time is spent. Reading execution plans is the most reliable way to diagnose slow queries.

---

## EXPLAIN

`EXPLAIN` shows the execution plan without running the query. It is safe to use on any query — including slow ones on production data.

```sql
EXPLAIN
SELECT *
FROM orders
WHERE customer_id = 42;
```

---

## EXPLAIN ANALYZE

`EXPLAIN ANALYZE` runs the query and shows both the estimated and actual execution statistics. Use this to verify that estimates match reality.

```sql
EXPLAIN ANALYZE
SELECT *
FROM orders
WHERE customer_id = 42;
```

!!! warning
    `EXPLAIN ANALYZE` actually executes the query. Avoid using it with `INSERT`, `UPDATE`, or `DELETE` on production data — or wrap it in a transaction and roll back.

```sql
-- Safe pattern for DML
BEGIN;
EXPLAIN ANALYZE
DELETE FROM orders WHERE status = 'cancelled';
ROLLBACK;
```

---

## Reading a PostgreSQL Execution Plan

```
Seq Scan on orders  (cost=0.00..18450.00 rows=1 width=72)
                          (actual time=0.042..183.210 rows=1 loops=1)
  Filter: (customer_id = 42)
  Rows Removed by Filter: 1000000
Planning Time: 0.5 ms
Execution Time: 183.3 ms
```

| Field | Meaning |
|---|---|
| `Seq Scan` | Node type — full sequential scan |
| `cost=0.00..18450.00` | Estimated startup cost .. total cost |
| `rows=1` | Estimated rows returned |
| `actual time=0.042..183.210` | Actual startup ms .. total ms |
| `rows=1` (actual) | Rows actually returned |
| `Rows Removed by Filter` | Rows scanned but discarded |
| `loops=1` | How many times this node executed |

---

## Common Node Types

| Node | Meaning |
|---|---|
| `Seq Scan` | Full table scan — reads every row |
| `Index Scan` | Navigates index, then fetches rows from table |
| `Index Only Scan` | Reads index only — no table access |
| `Bitmap Index Scan` | Uses index to build a bitmap, then scans heap |
| `Nested Loop` | Join strategy — for each row in outer, scan inner |
| `Hash Join` | Builds hash table from smaller input, probes with larger |
| `Merge Join` | Joins two pre-sorted inputs |
| `Sort` | Sorts rows — triggered by ORDER BY or Merge Join |
| `Hash` | Builds a hash table |
| `Aggregate` | Computes GROUP BY or aggregate functions |
| `Limit` | Stops after N rows |

---

## Seq Scan — When It Is a Problem

A `Seq Scan` is not always bad. On small tables, it is often faster than an index lookup. On large tables with selective filters, it indicates a missing or unused index.

```sql
-- Large table, selective filter → Seq Scan is a problem
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 42;

-- Result: Seq Scan, Rows Removed by Filter: 9,999,995
-- Fix: CREATE INDEX idx_orders_customer_id ON orders (customer_id);
```

After adding the index:

```
Index Scan using idx_orders_customer_id on orders
  (cost=0.43..8.45 rows=5 width=72)
  (actual time=0.021..0.035 rows=5 loops=1)
  Index Cond: (customer_id = 42)
```

---

## Identifying Bottlenecks

Look for:

**High actual rows vs estimated rows** — statistics are stale. Run `ANALYZE`.
```
rows=1 (estimated) vs rows=500000 (actual) → bad estimate
```

**Large `Rows Removed by Filter`** — many rows scanned but discarded. Add an index.

**`Sort` node on large datasets** — expensive. Check if an index on the `ORDER BY` column would eliminate it.

**`Nested Loop` on large outer input** — can be slow if the inner side is not indexed.

**`Hash Join` with large hash table** — check `work_mem` settings if the hash spills to disk.

---

## EXPLAIN Output Formats

```sql
-- Default text format
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;

-- JSON format — useful for tooling and detailed analysis
EXPLAIN (FORMAT JSON, ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 42;

-- BUFFERS — shows disk and memory I/O
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 42;
```

---

## SQL Server Execution Plans

SQL Server uses `SET STATISTICS` and graphical execution plans in SSMS.

```sql
-- Show estimated plan (no execution)
SET SHOWPLAN_TEXT ON;
GO
SELECT * FROM orders WHERE customer_id = 42;
GO

-- Show actual plan after execution
SET STATISTICS PROFILE ON;
SELECT * FROM orders WHERE customer_id = 42;
SET STATISTICS PROFILE OFF;

-- Show I/O and time statistics
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
SELECT * FROM orders WHERE customer_id = 42;
SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
```

In SSMS: use **Query → Include Actual Execution Plan** (Ctrl+M) for the graphical view.

---

## Vendor Notes

| Feature | PostgreSQL | SQL Server | MySQL |
|---|---|---|---|
| Estimated plan | `EXPLAIN` | `SET SHOWPLAN_TEXT ON` | `EXPLAIN` |
| Actual plan | `EXPLAIN ANALYZE` | `SET STATISTICS PROFILE ON` | `EXPLAIN ANALYZE` |
| JSON output | `EXPLAIN (FORMAT JSON)` | XML format | `EXPLAIN FORMAT=JSON` |
| I/O statistics | `EXPLAIN (BUFFERS)` | `SET STATISTICS IO ON` | — |

---

## Best Practices

- Always use `EXPLAIN ANALYZE` on slow queries before adding indexes — confirm the problem first
- Compare estimated vs actual rows — large differences indicate stale statistics
- Look for `Seq Scan` on large tables with selective filters — usually a missing index
- Check `Rows Removed by Filter` — high numbers mean you are reading far more than you need
- Use `EXPLAIN (ANALYZE, BUFFERS)` in PostgreSQL to see disk I/O alongside timing
- Wrap `EXPLAIN ANALYZE` on DML in a transaction and roll back to avoid side effects in production
- Re-run `EXPLAIN ANALYZE` after adding indexes to confirm the plan changed as expected
