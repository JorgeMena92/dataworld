---
title: Pivot and Unpivot
description: Transforming rows to columns and columns to rows — ANSI SQL vs vendor syntax
tags: [sql, patterns, pivot, unpivot]
---

# Pivot and Unpivot

Pivoting transforms row values into columns — turning a normalized, tall dataset into a wide, crosstab format. Unpivoting does the reverse — converting columns into rows. These transformations are common in reporting and data preparation.

---

## The Problem

**Before pivot (tall format):**
```
product    month    revenue
Laptop     Jan      5000
Laptop     Feb      6200
Monitor    Jan      2100
Monitor    Feb      2800
```

**After pivot (wide format):**
```
product    Jan     Feb
Laptop     5000    6200
Monitor    2100    2800
```

---

## PIVOT — ANSI SQL Approach (CASE WHEN)

ANSI SQL does not have a native `PIVOT` keyword. The standard approach uses conditional aggregation with `CASE WHEN`.

```sql
-- Monthly revenue by product — pivot months into columns
SELECT
    product,
    SUM(CASE WHEN month = 'Jan' THEN revenue ELSE 0 END) AS jan,
    SUM(CASE WHEN month = 'Feb' THEN revenue ELSE 0 END) AS feb,
    SUM(CASE WHEN month = 'Mar' THEN revenue ELSE 0 END) AS mar,
    SUM(CASE WHEN month = 'Apr' THEN revenue ELSE 0 END) AS apr
FROM monthly_sales
GROUP BY product
ORDER BY product;
```

```sql
-- Orders by status per country
SELECT
    country,
    COUNT(CASE WHEN status = 'completed'  THEN 1 END) AS completed,
    COUNT(CASE WHEN status = 'pending'    THEN 1 END) AS pending,
    COUNT(CASE WHEN status = 'cancelled'  THEN 1 END) AS cancelled
FROM orders
JOIN customers USING (customer_id)
GROUP BY country
ORDER BY country;
```

---

## PIVOT — SQL Server Native Syntax

SQL Server offers a `PIVOT` operator that is more concise but vendor-specific.

```sql
-- SQL Server PIVOT
SELECT product, [Jan], [Feb], [Mar], [Apr]
FROM (
    SELECT product, month, revenue
    FROM monthly_sales
) AS source
PIVOT (
    SUM(revenue)
    FOR month IN ([Jan], [Feb], [Mar], [Apr])
) AS pivoted
ORDER BY product;
```

### ANSI Equivalent

```sql
-- Equivalent ANSI SQL — portable across all databases
SELECT
    product,
    SUM(CASE WHEN month = 'Jan' THEN revenue END) AS jan,
    SUM(CASE WHEN month = 'Feb' THEN revenue END) AS feb,
    SUM(CASE WHEN month = 'Mar' THEN revenue END) AS mar,
    SUM(CASE WHEN month = 'Apr' THEN revenue END) AS apr
FROM monthly_sales
GROUP BY product;
```

| | ANSI (CASE WHEN) | SQL Server PIVOT |
|---|---|---|
| Portability | ✅ All databases | ❌ SQL Server / Oracle only |
| Dynamic columns | ❌ Manual | ❌ Manual (static) |
| Readability | Medium | Cleaner for simple cases |
| Aggregation control | ✅ Full | Limited |

---

## UNPIVOT — ANSI SQL Approach (UNION ALL)

Unpivoting converts columns back into rows. The ANSI approach uses `UNION ALL`.

```sql
-- Wide format: one row per product, columns for each month
-- Convert to tall format: one row per product per month

SELECT product, 'Jan' AS month, jan AS revenue FROM product_monthly WHERE jan IS NOT NULL
UNION ALL
SELECT product, 'Feb' AS month, feb AS revenue FROM product_monthly WHERE feb IS NOT NULL
UNION ALL
SELECT product, 'Mar' AS month, mar AS revenue FROM product_monthly WHERE mar IS NOT NULL
UNION ALL
SELECT product, 'Apr' AS month, apr AS revenue FROM product_monthly WHERE apr IS NOT NULL
ORDER BY product, month;
```

---

## UNPIVOT — SQL Server Native Syntax

```sql
-- SQL Server UNPIVOT
SELECT product, month, revenue
FROM product_monthly
UNPIVOT (
    revenue FOR month IN ([Jan], [Feb], [Mar], [Apr])
) AS unpivoted
ORDER BY product, month;
```

### ANSI Equivalent

```sql
-- Portable UNION ALL approach
SELECT product, 'Jan' AS month, jan AS revenue FROM product_monthly
UNION ALL
SELECT product, 'Feb' AS month, feb AS revenue FROM product_monthly
UNION ALL
SELECT product, 'Mar' AS month, mar AS revenue FROM product_monthly
UNION ALL
SELECT product, 'Apr' AS month, apr AS revenue FROM product_monthly
ORDER BY product, month;
```

---

## PostgreSQL — Using crosstab (tablefunc)

PostgreSQL offers a `crosstab` function for pivoting, but it requires the `tablefunc` extension.

```sql
-- Enable the extension
CREATE EXTENSION IF NOT EXISTS tablefunc;

-- Pivot using crosstab
SELECT *
FROM crosstab(
    'SELECT product, month, revenue FROM monthly_sales ORDER BY product, month',
    'SELECT DISTINCT month FROM monthly_sales ORDER BY month'
) AS ct (product TEXT, jan NUMERIC, feb NUMERIC, mar NUMERIC, apr NUMERIC);
```

!!! tip
    For most use cases, the `CASE WHEN` approach is simpler and more portable than `crosstab`. Use `crosstab` only when dealing with a large number of dynamic columns.

---

## Choosing the Right Approach

| Scenario | Recommendation |
|---|---|
| Portable SQL (multi-database) | ANSI `CASE WHEN` / `UNION ALL` |
| SQL Server only | Native `PIVOT` / `UNPIVOT` for readability |
| Migrating from SQL Server | Rewrite as `CASE WHEN` / `UNION ALL` |
| Dynamic columns (unknown at query time) | Dynamic SQL with string building |

!!! note
    Dynamic pivots — where the column values are not known at query write time — cannot be expressed in static SQL. The two main approaches are: (1) build the SQL string dynamically in a stored procedure using `EXEC` (SQL Server) or `EXECUTE` (PostgreSQL), or (2) handle the transformation in the application layer or pipeline tool (Python, dbt, Spark) where column lists can be computed at runtime. This is the recommended approach for production pipelines where the list of pivot values changes over time.

---

## Best Practices

- Use `CASE WHEN` aggregation for pivot when writing ANSI SQL — it is fully portable
- Use `UNION ALL` for unpivot — it is clear, portable, and easy to extend
- When migrating from SQL Server, replace `PIVOT`/`UNPIVOT` with the ANSI equivalents
- Add `WHERE column IS NOT NULL` in `UNION ALL` unpivots to avoid rows with no data
- For truly dynamic pivots (unknown column values at query time), use dynamic SQL or handle in the application layer