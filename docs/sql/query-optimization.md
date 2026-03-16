---
title: Query Optimization
description: Writing efficient SQL, understanding the optimizer, and avoiding common anti-patterns
tags: [sql, performance, optimization]
---

# Query Optimization

Query optimization is the process of writing SQL that the database engine can execute efficiently — minimizing the amount of data read, the number of operations performed, and the time spent waiting for results.

---

## How the Query Optimizer Works

When you submit a SQL query, the database does not execute it literally. The query optimizer analyzes the query and generates an execution plan — a step-by-step strategy for retrieving the data as efficiently as possible.

The optimizer considers:
- Available indexes
- Table statistics (row counts, data distribution, cardinality)
- Join order and join strategy
- Filter selectivity — how many rows each condition eliminates

```sql
-- You write this
SELECT * FROM orders WHERE customer_id = 42;

-- The optimizer decides:
-- → Use index on customer_id? (yes, if it exists and is selective)
-- → Full table scan? (if the table is tiny or no index exists)
-- → Estimated rows returned: 5 (from statistics)
```

---

## Table Statistics

Statistics are metadata the optimizer uses to estimate how many rows a query will return at each step. Accurate statistics lead to good plans. Stale statistics lead to bad ones.

```sql
-- PostgreSQL — update statistics manually
ANALYZE orders;
ANALYZE;  -- analyze all tables

-- SQL Server
UPDATE STATISTICS orders;
UPDATE STATISTICS orders WITH FULLSCAN;

-- View statistics info (PostgreSQL)
SELECT
    tablename,
    attname,
    n_distinct,
    correlation
FROM pg_stats
WHERE tablename = 'orders';
```

!!! note
    There is no ANSI SQL standard for managing statistics. `ANALYZE` is PostgreSQL syntax, also used in SQLite. SQL Server uses `UPDATE STATISTICS`. MySQL updates statistics automatically on `ANALYZE TABLE`. Always refer to your platform's documentation for statistics maintenance commands.

!!! tip
    Most databases update statistics automatically on a schedule. After large bulk loads or deletes, run `ANALYZE` manually to ensure the optimizer has accurate information before running expensive queries.

---

## Filter Early

The less data a query processes, the faster it runs. Push filters as close to the source as possible — most modern optimizers will push `WHERE` conditions down automatically, but writing the filter explicitly makes the intent clear and helps in cases where the optimizer cannot.

```sql
-- Both forms are typically equivalent — the optimizer pushes the filter down
SELECT c.first_name, o.amount
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE c.country = 'Peru';

-- CTE form — same performance in most optimizers, clearer intent
WITH peruvian_customers AS (
    SELECT customer_id, first_name
    FROM customers
    WHERE country = 'Peru'
)
SELECT c.first_name, o.amount
FROM orders o
JOIN peruvian_customers c ON o.customer_id = c.customer_id;
```

!!! note
    Some older databases or complex queries may not push filters through CTEs automatically. Use `EXPLAIN ANALYZE` to verify the filter is applied early in the plan — do not assume the CTE form is always faster.

---

## Avoid SELECT *

`SELECT *` fetches every column — even ones you do not need. It prevents index-only scans and increases memory and I/O usage.

```sql
-- Bad
SELECT * FROM orders WHERE customer_id = 42;

-- Good — fetch only what you need
SELECT order_id, amount, status FROM orders WHERE customer_id = 42;
```

---

## Avoid Functions on Indexed Columns

Applying a function to an indexed column in a `WHERE` clause prevents the index from being used.

```sql
-- Bad — function on column, index cannot be used
WHERE UPPER(email) = 'JORGE@EXAMPLE.COM'
WHERE EXTRACT(YEAR FROM created_at) = 2024
WHERE amount + 10 > 100

-- Good — transform the value, not the column
WHERE email = LOWER('JORGE@EXAMPLE.COM')
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
WHERE amount > 90
```

---

## EXISTS vs IN vs JOIN

All three can test for the presence of related rows — but they perform differently.

```sql
-- IN — evaluates the full subquery, returns a list
SELECT * FROM customers
WHERE customer_id IN (SELECT customer_id FROM orders);

-- EXISTS — stops at the first match per row (faster on large subqueries)
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);

-- JOIN — most flexible, optimizer can choose the best strategy
SELECT DISTINCT c.*
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id;
```

!!! tip
    Use `EXISTS` when checking for presence. Use `JOIN` when you need columns from both tables. Avoid `NOT IN` with nullable subqueries — use `NOT EXISTS` instead.

---

## Use UNION ALL Instead of UNION

`UNION` deduplicates results by sorting and comparing all rows. `UNION ALL` skips this step.

```sql
-- Bad — unnecessary sort and deduplication
SELECT city FROM customers
UNION
SELECT city FROM suppliers;

-- Good — when duplicates are acceptable or already impossible
SELECT city FROM customers
UNION ALL
SELECT city FROM suppliers;
```

---

## Avoid Correlated Subqueries on Large Tables

A correlated subquery runs once per row of the outer query. On large tables this is equivalent to a nested loop over millions of rows.

```sql
-- Bad — correlated subquery runs once per employee row
SELECT employee_id, salary,
    (SELECT AVG(salary) FROM employees WHERE department = e.department) AS dept_avg
FROM employees e;

-- Good — compute once with a window function
SELECT
    employee_id,
    salary,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees;
```

---

## Limit Results Early

When you only need a few rows, tell the database — it can stop scanning once it has enough.

```sql
-- Get the 10 most recent orders
SELECT order_id, amount, created_at
FROM orders
ORDER BY created_at DESC
FETCH FIRST 10 ROWS ONLY;
```

---

## Common Anti-Patterns

| Anti-pattern | Problem | Fix |
|---|---|---|
| `SELECT *` | Fetches unnecessary data | Select only needed columns |
| Function on indexed column | Bypasses index | Transform the value, not the column |
| `NOT IN` with nullable subquery | Returns no rows | Use `NOT EXISTS` |
| Correlated subquery | Runs N times (once per row) | Rewrite as JOIN or window function |
| `UNION` instead of `UNION ALL` | Unnecessary sort | Use `UNION ALL` unless dedup is needed |
| Missing `WHERE` on large table | Full table scan | Always filter on indexed columns |
| `ORDER BY` without `FETCH FIRST` | Sorts entire result | Pair with row limit |
| Implicit type conversion | Bypasses index | Cast explicitly, match column types |

---

## Best Practices

- Use `EXPLAIN` / `EXPLAIN ANALYZE` to understand what the database actually does — see [Execution Plans](execution-plans.md)
- Write queries that filter as early and as selectively as possible
- Match the data types of compared values to the column type — mismatches cause implicit casts that bypass indexes
- Update statistics after large data loads
- Benchmark with realistic data volumes — a query that is fast on 1,000 rows may be slow on 10,000,000
- Optimize the most frequently executed queries first — not the most complex ones