---
title: Temporary Tables
description: Creating and using temporary tables for intermediate results and multi-step transformations
tags: [sql, dql, temporary-tables]
---

# Temporary Tables

A temporary table is a table that exists only for the duration of a session or transaction. It behaves like a regular table — you can insert, update, delete, index, and query it — but it is automatically dropped when the session ends.

Temporary tables are useful when intermediate results are large, reused multiple times, or too complex to express cleanly in a single CTE chain.

---

## Creating a Temporary Table

```sql
-- Create and define structure explicitly
CREATE TEMPORARY TABLE temp_high_value_customers (
    customer_id   INT,
    total_spent   DECIMAL(10,2)
);

-- Create and populate from a query (most common pattern)
CREATE TEMPORARY TABLE temp_high_value_customers AS
SELECT
    customer_id,
    SUM(amount) AS total_spent
FROM orders
GROUP BY customer_id
HAVING SUM(amount) > 5000;
```

!!! tip
    The `TEMPORARY` keyword can be shortened to `TEMP` in most databases — `TEMPORARY` is the strict ANSI keyword but `TEMP` is widely accepted as a shorthand.

---

## Using a Temporary Table

Once created, a temp table is queried exactly like a regular table.

```sql
-- Populate
CREATE TEMP TABLE temp_completed_orders AS
SELECT *
FROM orders
WHERE status = 'completed'
  AND order_date >= CURRENT_DATE - INTERVAL '90' DAY;

-- Reuse multiple times in the same session
SELECT COUNT(*) FROM temp_completed_orders;

SELECT
    country,
    SUM(amount) AS total_revenue
FROM temp_completed_orders o
JOIN customers c ON o.customer_id = c.customer_id
GROUP BY country;
```

---

## INSERT INTO a Temporary Table

```sql
CREATE TEMP TABLE temp_summary (
    region        VARCHAR(100),
    total_orders  INT,
    total_revenue DECIMAL(12,2)
);

INSERT INTO temp_summary
SELECT
    c.region,
    COUNT(*)      AS total_orders,
    SUM(o.amount) AS total_revenue
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
GROUP BY c.region;

SELECT * FROM temp_summary ORDER BY total_revenue DESC;
```

---

## Dropping a Temporary Table

Temp tables are dropped automatically at the end of the session, but you can drop them explicitly when no longer needed.

```sql
DROP TABLE IF EXISTS temp_high_value_customers;
```

---

## Temporary Tables vs CTEs

Both are tools for managing intermediate results. The right choice depends on the situation.

| | CTE | Temporary Table |
|---|---|---|
| Scope | Single query | Entire session |
| Reusable across queries | ❌ | ✅ |
| Can be indexed | ❌ | ✅ |
| Materializes results | Depends on engine | ✅ Always |
| Syntax overhead | Low | Medium |
| Best for | Readable multi-step logic | Large intermediate results reused multiple times |

!!! tip
    Start with a CTE. Move to a temporary table when the intermediate result is large, queried multiple times, or needs an index to support further joins.

---

## Temporary Tables vs Subqueries

```sql
-- Subquery — computed every time it is referenced
SELECT *
FROM orders
WHERE customer_id IN (
    SELECT customer_id
    FROM customers
    WHERE country = 'Peru'
);

-- Temporary table — computed once, reused
CREATE TEMP TABLE temp_peru_customers AS
SELECT customer_id FROM customers WHERE country = 'Peru';

SELECT * FROM orders   WHERE customer_id IN (SELECT customer_id FROM temp_peru_customers);
SELECT * FROM invoices WHERE customer_id IN (SELECT customer_id FROM temp_peru_customers);
```

---

## Adding Indexes to Temporary Tables

Indexes on temporary tables can significantly speed up subsequent joins and filters on large intermediate datasets.

```sql
CREATE TEMP TABLE temp_orders AS
SELECT *
FROM orders
WHERE status = 'completed';

-- Add an index to support the upcoming join
CREATE INDEX idx_temp_orders_customer
ON temp_orders(customer_id);

-- Now this join uses the index
SELECT c.first_name, t.amount
FROM temp_orders t
JOIN customers c ON t.customer_id = c.customer_id;
```

---

## Vendor Differences

Temporary table syntax varies more than most SQL features across platforms.

| Database | Syntax | Scope |
|---|---|---|
| PostgreSQL | `CREATE TEMP TABLE` | Session |
| SQL Server | `CREATE TABLE #temp_name` | Session (`#`) or global (`##`) |
| MySQL | `CREATE TEMPORARY TABLE` | Session |
| Oracle | `CREATE GLOBAL TEMPORARY TABLE` | Session or transaction |
| Snowflake | `CREATE TEMPORARY TABLE` | Session |
| BigQuery | `CREATE TEMP TABLE` | Script / session |

```sql
-- ANSI SQL / PostgreSQL / MySQL / Snowflake
CREATE TEMP TABLE temp_orders AS
SELECT * FROM orders WHERE status = 'completed';

-- SQL Server — uses SELECT INTO with # prefix, no AS SELECT syntax
SELECT *
INTO #temp_orders
FROM orders
WHERE status = 'completed';

SELECT * FROM #temp_orders;
```

!!! warning
    SQL Server does not support `CREATE TABLE ... AS SELECT`. Use `SELECT ... INTO #table_name` instead. The `#` prefix is required — it replaces the `TEMP`/`TEMPORARY` keyword.

---

## Common Use Cases

| Use Case | Why Temp Table |
|---|---|
| Multi-step ETL transformations | Results are large and reused across multiple steps |
| Staging data before loading | Isolate raw data before applying transformations |
| Debugging complex queries | Inspect intermediate results independently |
| Performance optimization | Materialize expensive subqueries used multiple times |
| Session-scoped working data | Store state across multiple queries in the same session |

---

## Best Practices

- Always use `DROP TABLE IF EXISTS` before creating a temp table in scripts — avoids errors if the table already exists from a previous run
- Explicitly drop temp tables when you are done — do not rely on session end cleanup in long-running sessions
- Use `CREATE TEMP TABLE ... AS SELECT` over `CREATE` + `INSERT` when you just need to persist a query result
- Add indexes when the temp table will be joined or filtered on large datasets
- Prefer CTEs for single-query logic — use temp tables when results are reused across multiple queries
- Name temp tables with a `temp_` prefix to make their nature obvious in scripts

---

## Putting It All Together

A complete script pattern combining all best practices — drop before create, ANSI-portable date filter, multiple reuses, and explicit cleanup at the end.

```sql
-- Safe script pattern
DROP TABLE IF EXISTS temp_orders;

CREATE TEMP TABLE temp_orders AS
SELECT *
FROM orders
WHERE status = 'completed'
  AND EXTRACT(YEAR FROM order_date) = EXTRACT(YEAR FROM CURRENT_DATE);

-- Use it multiple times
SELECT COUNT(*) FROM temp_orders;
SELECT country, SUM(amount) FROM temp_orders JOIN customers USING (customer_id) GROUP BY country;

-- Clean up
DROP TABLE IF EXISTS temp_orders;
```