---
title: Advanced
description: Advanced SQL patterns for experienced engineers
tags: [sql, advanced]
---

# Advanced

Complex SQL patterns used in real data engineering work. These go beyond querying — they cover performance, maintainability, and solving hard data problems elegantly.

---

## Recursive CTEs

Recursive CTEs solve hierarchical problems — org charts, category trees, bill of materials, graph traversal. They work by referencing themselves until a termination condition is met.

```sql
WITH RECURSIVE org_chart AS (
    -- Anchor: start with top-level managers
    SELECT
        employee_id,
        manager_id,
        first_name,
        0 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: join each employee to their manager
    SELECT
        e.employee_id,
        e.manager_id,
        e.first_name,
        o.level + 1
    FROM employees e
    JOIN org_chart o
        ON e.manager_id = o.employee_id
)
SELECT *
FROM org_chart
ORDER BY level, employee_id;
```

!!! tip
    Always include a termination condition (the anchor query returns no rows eventually) to avoid infinite loops.

---

## Advanced Window Frames

Window frames control exactly which rows are included in each calculation relative to the current row.

```sql
-- Default: all rows in partition
SUM(amount) OVER (PARTITION BY customer_id)

-- Rows from start of partition to current row (running total)
SUM(amount) OVER (
    ORDER BY order_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
)

-- 7-day rolling average
AVG(amount) OVER (
    ORDER BY order_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
)

-- 3-row centered moving average
AVG(amount) OVER (
    ORDER BY order_date
    ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
)
```

---

## PIVOT and Conditional Aggregation

Transform rows into columns — useful for creating crosstab reports.

```sql
-- Conditional aggregation (ANSI SQL, works everywhere)
SELECT
    product_id,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 1 THEN amount ELSE 0 END) AS jan,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 2 THEN amount ELSE 0 END) AS feb,
    SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 3 THEN amount ELSE 0 END) AS mar
FROM orders
WHERE EXTRACT(YEAR FROM order_date) = 2024
GROUP BY product_id;
```

---

## MERGE (Upsert)

Insert new rows or update existing ones in a single statement — the standard pattern for loading dimension tables and slowly changing dimensions.

```sql
MERGE INTO customers AS target
USING staging_customers AS source
    ON target.customer_id = source.customer_id

WHEN MATCHED THEN
    UPDATE SET
        target.email      = source.email,
        target.updated_at = CURRENT_TIMESTAMP

WHEN NOT MATCHED THEN
    INSERT (customer_id, first_name, last_name, email, created_at)
    VALUES (source.customer_id, source.first_name, source.last_name, source.email, CURRENT_TIMESTAMP);
```

---

## Gaps and Islands

Identify consecutive sequences or breaks in sequences — common in time series, session analysis, and scheduling.

```sql
-- Find gaps in order sequences
WITH numbered AS (
    SELECT
        order_id,
        LAG(order_id) OVER (ORDER BY order_id) AS prev_id
    FROM orders
)
SELECT
    prev_id + 1 AS gap_start,
    order_id - 1 AS gap_end
FROM numbered
WHERE order_id - prev_id > 1;

-- Island detection — group consecutive active days
SELECT
    user_id,
    MIN(activity_date) AS island_start,
    MAX(activity_date) AS island_end,
    COUNT(*) AS days_active
FROM (
    SELECT
        user_id,
        activity_date,
        activity_date - INTERVAL '1 day' * ROW_NUMBER() OVER (
            PARTITION BY user_id ORDER BY activity_date
        ) AS island_group
    FROM user_activity
) t
GROUP BY user_id, island_group
ORDER BY user_id, island_start;
```

---

## Query Optimization

### Use indexes wisely

```sql
-- Queries that benefit from indexes
WHERE customer_id = 42             -- equality on indexed column
WHERE created_at >= '2024-01-01'   -- range on indexed column
ORDER BY customer_id               -- sort on indexed column

-- Queries that bypass indexes
WHERE UPPER(email) = 'JORGE@...'   -- function on column
WHERE amount + 10 > 100            -- expression on column
WHERE email LIKE '%gmail.com'      -- leading wildcard
```

### Avoid SELECT *

```sql
-- Bad — fetches all columns, prevents index-only scans
SELECT * FROM orders WHERE customer_id = 42;

-- Good — fetch only what you need
SELECT order_id, amount, status FROM orders WHERE customer_id = 42;
```

### Filter early with CTEs

```sql
-- Pre-filter before joining reduces rows processed
WITH recent_orders AS (
    SELECT * FROM orders
    WHERE created_at >= CURRENT_DATE - INTERVAL '90 days'
)
SELECT c.first_name, r.amount
FROM recent_orders r
JOIN customers c ON r.customer_id = c.customer_id;
```

### EXISTS vs COUNT for existence checks

```sql
-- Bad — counts all rows just to check if any exist
SELECT * FROM customers
WHERE (SELECT COUNT(*) FROM orders WHERE customer_id = customers.customer_id) > 0;

-- Good — stops at first match
SELECT * FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

---

## Temporal Queries

Work with time-based data — trends, period comparisons, and date math.

```sql
-- Month over month comparison
WITH monthly AS (
    SELECT
        DATE_TRUNC('month', order_date) AS month,
        SUM(amount) AS total
    FROM orders
    GROUP BY 1
)
SELECT
    month,
    total,
    LAG(total) OVER (ORDER BY month) AS prev_month,
    ROUND(
        100.0 * (total - LAG(total) OVER (ORDER BY month))
        / NULLIF(LAG(total) OVER (ORDER BY month), 0),
        2
    ) AS growth_pct
FROM monthly
ORDER BY month;
```

---

## Best Practices

- Use `EXPLAIN` / `EXPLAIN ANALYZE` to understand query execution before optimizing
- Avoid correlated subqueries on large tables — rewrite as JOINs or CTEs
- Use `NULLIF` to avoid division by zero in calculations
- Break complex logic into CTEs — one responsibility per CTE
- Test recursive CTEs with a row limit first (`FETCH FIRST 100 ROWS`) to avoid runaway queries
- Document non-obvious logic with inline comments

```sql
-- Use EXPLAIN to see what the database actually does
EXPLAIN ANALYZE
SELECT *
FROM orders
WHERE customer_id = 42
  AND status = 'completed';
```
