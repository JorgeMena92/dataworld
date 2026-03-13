---
title: CTEs
description: Common Table Expressions — structure, chaining, and recursive patterns
tags: [sql, dql, ctes]
---

# CTEs — Common Table Expressions

A CTE is a named, temporary result set defined at the top of a query using the `WITH` keyword. It exists only for the duration of the query and can be referenced like a table in the main statement.

CTEs make complex queries readable by breaking logic into named, sequential steps — one responsibility per block.

---

## Basic Syntax

```sql
WITH cte_name AS (
    SELECT ...
    FROM ...
    WHERE ...
)
SELECT *
FROM cte_name;
```

---

## Single CTE

```sql
-- Customers who have spent more than 5000
WITH high_value_customers AS (
    SELECT
        customer_id,
        SUM(amount) AS total_spent
    FROM orders
    GROUP BY customer_id
    HAVING SUM(amount) > 5000
)
SELECT
    c.first_name,
    c.email,
    h.total_spent
FROM customers c
JOIN high_value_customers h
    ON c.customer_id = h.customer_id
ORDER BY h.total_spent DESC;
```

---

## Chaining Multiple CTEs

Multiple CTEs are separated by commas. Each one can reference the CTEs defined before it.

```sql
WITH monthly_sales AS (
    SELECT
        DATE_TRUNC('month', order_date) AS month,
        SUM(amount)                     AS total
    FROM orders
    GROUP BY 1
),
ranked_months AS (
    SELECT
        month,
        total,
        RANK() OVER (ORDER BY total DESC) AS rank
    FROM monthly_sales
)
SELECT *
FROM ranked_months
WHERE rank <= 3;
```

!!! tip
    Think of chained CTEs as a pipeline — each step transforms the data and passes it to the next. Name each CTE after what it represents, not what it does: `high_value_customers`, not `filtered_customers`.

---

## CTE vs Subquery vs Derived Table

All three can produce the same result. CTEs win on readability for anything beyond a single level of nesting.

```sql
-- Subquery (hard to read when nested)
SELECT department, avg_salary
FROM (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
) dept_summary
WHERE avg_salary > 50000;

-- CTE (easier to read and debug)
WITH dept_summary AS (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
)
SELECT department, avg_salary
FROM dept_summary
WHERE avg_salary > 50000;
```

---

## CTE for Deduplication

One of the most common uses in data engineering — keep the latest record per business key.

```sql
WITH ranked AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY updated_at DESC
        ) AS rn
    FROM customers
)
SELECT *
FROM ranked
WHERE rn = 1;
```

---

## CTE for Step-by-Step Transformations

CTEs shine when a query requires multiple transformation steps.

```sql
WITH raw_orders AS (
    SELECT *
    FROM orders
    WHERE status = 'completed'
),
enriched AS (
    SELECT
        o.order_id,
        o.amount,
        c.country,
        c.segment
    FROM raw_orders o
    JOIN customers c ON o.customer_id = c.customer_id
),
aggregated AS (
    SELECT
        country,
        segment,
        COUNT(*)    AS total_orders,
        SUM(amount) AS total_revenue
    FROM enriched
    GROUP BY country, segment
)
SELECT *
FROM aggregated
WHERE total_revenue > 10000
ORDER BY total_revenue DESC;
```

---

## Recursive CTEs

A recursive CTE references itself. It repeats until a termination condition is met — making it the standard tool for hierarchical and graph problems.

### Syntax

```sql
WITH RECURSIVE cte_name AS (
    -- Anchor: the starting point (runs once)
    SELECT ...

    UNION ALL

    -- Recursive: references cte_name, runs repeatedly until no rows returned
    SELECT ...
    FROM cte_name
    WHERE ...  -- termination condition
)
SELECT * FROM cte_name;
```

### Org Chart — Walking a Hierarchy

```sql
WITH RECURSIVE org_chart AS (
    -- Anchor: top-level employees (no manager)
    SELECT
        employee_id,
        first_name,
        manager_id,
        0 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: find direct reports of the current level
    SELECT
        e.employee_id,
        e.first_name,
        e.manager_id,
        o.level + 1
    FROM employees e
    JOIN org_chart o
        ON e.manager_id = o.employee_id
)
SELECT
    employee_id,
    first_name,
    level,
    REPEAT('  ', level) || first_name AS indented_name
FROM org_chart
ORDER BY level, employee_id;
```

### Category Tree — Parent-Child Navigation

```sql
WITH RECURSIVE category_tree AS (
    -- Anchor: root categories
    SELECT
        category_id,
        name,
        parent_id,
        name AS path
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- Recursive: child categories
    SELECT
        c.category_id,
        c.name,
        c.parent_id,
        ct.path || ' > ' || c.name
    FROM categories c
    JOIN category_tree ct
        ON c.parent_id = ct.category_id
)
SELECT *
FROM category_tree
ORDER BY path;
```

### Generating a Date Spine

```sql
WITH RECURSIVE date_spine AS (
    SELECT DATE '2024-01-01' AS dt

    UNION ALL

    SELECT dt + INTERVAL '1 day'
    FROM date_spine
    WHERE dt < DATE '2024-12-31'
)
SELECT dt
FROM date_spine;
```

A date spine is useful for filling gaps in time series data — join it to your fact table to ensure every date appears even if no events occurred.

!!! warning
    Recursive CTEs can run indefinitely if the termination condition is never met. Always verify your anchor and recursive parts produce a finite sequence. Use `FETCH FIRST n ROWS` during development to test safely.

---

## Common Use Cases

| Use Case | CTE Type |
|---|---|
| Breaking complex logic into steps | Regular |
| Deduplication with ROW_NUMBER | Regular |
| Reusing a subquery multiple times | Regular |
| Org charts and reporting hierarchies | Recursive |
| Category trees and parent-child structures | Recursive |
| Bill of materials | Recursive |
| Generating date or number sequences | Recursive |
| Graph traversal | Recursive |

---

## Best Practices

- Name CTEs after what they represent, not what they do — `completed_orders`, not `filtered_data`
- One responsibility per CTE — keep each step focused and small
- Prefer CTEs over nested subqueries for anything with more than one level of nesting
- Test recursive CTEs with a row limit first to avoid runaway queries
- Use `UNION ALL` inside recursive CTEs — `UNION` adds unnecessary deduplication overhead
- Keep the anchor query simple — complex logic belongs in subsequent CTEs built on top of it