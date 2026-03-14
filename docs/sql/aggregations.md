---
title: Aggregations
description: Summarizing data with aggregate functions, GROUP BY, and HAVING
tags: [sql, dql, aggregations]
---

# Aggregations

Aggregations summarize multiple rows into a single result. They are the foundation of reporting and analytics — counting records, summing revenue, averaging scores, and finding extremes.

---

## Core Aggregate Functions

| Function | Description |
|---|---|
| `COUNT(*)` | Count all rows |
| `COUNT(column)` | Count non-null values |
| `COUNT(DISTINCT column)` | Count unique non-null values |
| `SUM(column)` | Sum of all values |
| `AVG(column)` | Average of all values |
| `MIN(column)` | Smallest value |
| `MAX(column)` | Largest value |

```sql
SELECT
    COUNT(*)                  AS total_orders,
    COUNT(shipped_at)         AS shipped_orders,
    COUNT(DISTINCT customer_id) AS unique_customers,
    SUM(amount)               AS total_revenue,
    AVG(amount)               AS avg_order_value,
    MIN(amount)               AS smallest_order,
    MAX(amount)               AS largest_order
FROM orders;
```

!!! tip
    `COUNT(*)` counts all rows including nulls. `COUNT(column)` counts only non-null values in that column. Use the right one intentionally.

---

## GROUP BY

`GROUP BY` splits rows into groups before applying aggregate functions. One result row is returned per group.

```sql
-- Total revenue per country
SELECT
    country,
    COUNT(*)    AS total_orders,
    SUM(amount) AS total_revenue
FROM orders
GROUP BY country
ORDER BY total_revenue DESC;
```

### Group by multiple columns

```sql
-- Revenue broken down by country and year
SELECT
    country,
    EXTRACT(YEAR FROM order_date) AS year,
    SUM(amount) AS total_revenue
FROM orders
GROUP BY country, EXTRACT(YEAR FROM order_date)
ORDER BY country, year;
```

!!! tip
    Every column in `SELECT` that is not inside an aggregate function must appear in `GROUP BY`. If it doesn't, the query will fail.

---

## HAVING

`HAVING` filters groups **after** aggregation — the equivalent of `WHERE` but for aggregated results.

```sql
-- Only countries with more than 100 orders
SELECT
    country,
    COUNT(*) AS total_orders
FROM orders
GROUP BY country
HAVING COUNT(*) > 100
ORDER BY total_orders DESC;

-- Only customers who have spent more than 5000
SELECT
    customer_id,
    SUM(amount) AS total_spent
FROM orders
GROUP BY customer_id
HAVING SUM(amount) > 5000;
```

### WHERE vs HAVING

```sql
-- WHERE filters rows before grouping
-- HAVING filters groups after grouping

SELECT
    country,
    SUM(amount) AS total_revenue
FROM orders
WHERE status = 'completed'       -- filter rows first
GROUP BY country
HAVING SUM(amount) > 10000       -- then filter groups
ORDER BY total_revenue DESC;
```

| | WHERE | HAVING |
|---|---|---|
| Runs | Before `GROUP BY` | After `GROUP BY` |
| Filters | Individual rows | Aggregated groups |
| Can use aggregates | ❌ | ✅ |

---

## Aggregations with CASE WHEN

Combine conditional logic with aggregations to build pivot-style summaries.

```sql
-- Count orders by status in a single row
SELECT
    COUNT(*)                                          AS total_orders,
    COUNT(CASE WHEN status = 'completed' THEN 1 END) AS completed,
    COUNT(CASE WHEN status = 'pending'   THEN 1 END) AS pending,
    COUNT(CASE WHEN status = 'cancelled' THEN 1 END) AS cancelled
FROM orders;

-- Revenue by quarter — portable ANSI SQL using MONTH
SELECT
    EXTRACT(YEAR FROM order_date) AS year,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) <= 3  THEN amount ELSE 0 END) AS q1,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) <= 6  THEN amount ELSE 0 END) AS q2,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) <= 9  THEN amount ELSE 0 END) AS q3,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) <= 12 THEN amount ELSE 0 END) AS q4
FROM orders
GROUP BY EXTRACT(YEAR FROM order_date)
ORDER BY year;
```

---

## Aggregations with NULLs

Aggregate functions ignore `NULL` values — except `COUNT(*)`.

```sql
-- If some rows have NULL amount, AVG ignores them
SELECT AVG(amount) FROM orders;

-- To treat NULL as 0 before aggregating
SELECT AVG(COALESCE(amount, 0)) FROM orders;

-- Count how many rows have a NULL value
SELECT COUNT(*) - COUNT(amount) AS null_count
FROM orders;
```

!!! warning
    `AVG` over a column with NULLs gives a different result than `SUM / COUNT(*)`. Be explicit about how you want to handle missing values.

---

## ROLLUP — Subtotals and Grand Totals

`ROLLUP` extends `GROUP BY` to automatically add subtotal and grand total rows.

```sql
SELECT
    country,
    EXTRACT(YEAR FROM order_date) AS year,
    SUM(amount) AS total_revenue
FROM orders
GROUP BY ROLLUP (country, EXTRACT(YEAR FROM order_date))
ORDER BY country, year;
```

The result includes:
- One row per `country + year` combination
- One subtotal row per `country` (year = NULL)
- One grand total row (both NULL)

Use `COALESCE` to replace the NULL markers with readable labels:

```sql
SELECT
    COALESCE(country, 'All Countries')                                         AS country,
    COALESCE(CAST(EXTRACT(YEAR FROM order_date) AS VARCHAR), 'All Years')      AS year,
    SUM(amount)                                                                AS total_revenue
FROM orders
GROUP BY ROLLUP (country, EXTRACT(YEAR FROM order_date))
ORDER BY country, year;
```

!!! tip
    `ROLLUP` is ANSI SQL and supported in PostgreSQL, SQL Server, MySQL, and most modern databases.

---

## Best Practices

- Always be explicit about `COUNT(*)` vs `COUNT(column)` — they return different results when nulls are present
- Filter with `WHERE` before grouping to reduce the rows the database has to process
- Use `HAVING` only for conditions that depend on aggregate results
- Name aggregated columns clearly — `total_revenue`, `avg_order_value`, not just `sum` or `avg`
- When using `ROLLUP`, label null rows with `COALESCE` for readability