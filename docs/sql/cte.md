---
title: CTEs — Common Table Expressions
description: Write readable, maintainable SQL with Common Table Expressions
tags: [sql, cte, intermediate]
---

# CTEs — Common Table Expressions

A Common Table Expression (CTE) is a named, temporary result set defined at the top of a query using the `WITH` keyword. You reference it like a table for the rest of that query.

CTEs do not store data — they exist only for the duration of the statement they belong to.

---

## Why Use CTEs?

CTEs make complex queries readable by breaking them into named steps. Compare these two approaches to the same problem:

```sql
-- Without CTE — nested and hard to follow
SELECT c.first_name, c.email, s.total_spent
FROM customers c
JOIN (
    SELECT customer_id, SUM(amount) AS total_spent
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
    HAVING SUM(amount) > 5000
) s ON c.customer_id = s.customer_id;

-- With CTE — intention is clear
WITH high_value_customers AS (
    SELECT customer_id, SUM(amount) AS total_spent
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
    HAVING SUM(amount) > 5000
)
SELECT c.first_name, c.email, h.total_spent
FROM customers c
JOIN high_value_customers h
    ON c.customer_id = h.customer_id;
```

Both produce the same result. The CTE version reads like a sentence.

---

## Syntax

```sql
WITH cte_name AS (
    -- your query here
    SELECT ...
    FROM ...
    WHERE ...
)
SELECT *
FROM cte_name;
```

For multiple CTEs, separate them with a comma — only one `WITH` keyword:

```sql
WITH
cte_one AS (
    SELECT ...
),
cte_two AS (
    SELECT ...
    FROM cte_one   -- can reference a previous CTE
)
SELECT *
FROM cte_two;
```

---

## Single CTE

The simplest use case — one named step before the final query.

```sql
WITH active_employees AS (
    SELECT employee_id, first_name, department, salary
    FROM employees
    WHERE status = 'active'
)
SELECT
    department,
    COUNT(*)    AS headcount,
    AVG(salary) AS avg_salary
FROM active_employees
GROUP BY department
ORDER BY avg_salary DESC;
```

---

## Chained CTEs

Each CTE can reference any CTE defined before it. This lets you build logic step by step.

```sql
WITH
monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', order_date) AS month,
        SUM(amount) AS revenue
    FROM orders
    WHERE status = 'completed'
    GROUP BY 1
),
revenue_with_growth AS (
    SELECT
        month,
        revenue,
        LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue,
        ROUND(
            100.0 * (revenue - LAG(revenue) OVER (ORDER BY month))
            / NULLIF(LAG(revenue) OVER (ORDER BY month), 0),
            2
        ) AS growth_pct
    FROM monthly_revenue
),
positive_growth_months AS (
    SELECT *
    FROM revenue_with_growth
    WHERE growth_pct > 0
)
SELECT *
FROM positive_growth_months
ORDER BY month;
```

Each CTE has one clear responsibility. You can read the query top to bottom like a recipe.

---

## CTEs vs Subqueries

Use CTEs when logic is reused or when readability matters. Use subqueries for simple, one-off filters.

| | CTE | Subquery |
|---|---|---|
| Readability | ✅ Named, easy to follow | ❌ Nested, harder to read |
| Reusability | ✅ Can be referenced multiple times | ❌ Must be repeated |
| Debugging | ✅ Test each CTE in isolation | ❌ Hard to isolate inner logic |
| Performance | Similar (optimizer usually treats them the same) | Similar |
| Best for | Complex logic, multi-step transformations | Simple one-off conditions |

```sql
-- Subquery used twice — leads to repetition
SELECT *
FROM orders
WHERE customer_id IN (SELECT customer_id FROM customers WHERE country = 'Peru')
  AND amount > (SELECT AVG(amount) FROM orders WHERE customer_id IN (SELECT customer_id FROM customers WHERE country = 'Peru'));

-- CTE used twice — cleaner
WITH peruvian_customers AS (
    SELECT customer_id FROM customers WHERE country = 'Peru'
)
SELECT *
FROM orders
WHERE customer_id IN (SELECT customer_id FROM peruvian_customers)
  AND amount > (SELECT AVG(amount) FROM orders WHERE customer_id IN (SELECT customer_id FROM peruvian_customers));
```

---

## CTEs for Deduplication

A common ETL pattern — use a CTE to rank rows, then filter in the outer query.

```sql
WITH ranked_customers AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY email
            ORDER BY updated_at DESC
        ) AS rn
    FROM customers
)
SELECT *
FROM ranked_customers
WHERE rn = 1;
```

This keeps the most recently updated record per email address.

---

## CTEs for Incremental Logic

Break a complex business rule into stages — each CTE handles one condition.

```sql
WITH
completed_orders AS (
    SELECT customer_id, amount, order_date
    FROM orders
    WHERE status = 'completed'
),
orders_this_year AS (
    SELECT customer_id, SUM(amount) AS ytd_revenue
    FROM completed_orders
    WHERE EXTRACT(YEAR FROM order_date) = EXTRACT(YEAR FROM CURRENT_DATE)
    GROUP BY customer_id
),
customer_segments AS (
    SELECT
        customer_id,
        ytd_revenue,
        CASE
            WHEN ytd_revenue >= 10000 THEN 'Platinum'
            WHEN ytd_revenue >= 5000  THEN 'Gold'
            WHEN ytd_revenue >= 1000  THEN 'Silver'
            ELSE 'Bronze'
        END AS segment
    FROM orders_this_year
)
SELECT
    c.first_name,
    c.email,
    s.ytd_revenue,
    s.segment
FROM customers c
JOIN customer_segments s
    ON c.customer_id = s.customer_id
ORDER BY s.ytd_revenue DESC;
```

---

## Recursive CTEs

Recursive CTEs reference themselves to process hierarchical or sequential data. They have two required parts connected by `UNION ALL`:

1. **Anchor query** — the starting point (runs once)
2. **Recursive query** — joins back to the CTE, adding rows each iteration

```sql
-- Org chart — find all reports under a given manager
WITH RECURSIVE reports AS (
    -- Anchor: start with the manager
    SELECT
        employee_id,
        first_name,
        manager_id,
        0 AS depth
    FROM employees
    WHERE employee_id = 5   -- starting point

    UNION ALL

    -- Recursive: find each employee's direct reports
    SELECT
        e.employee_id,
        e.first_name,
        e.manager_id,
        r.depth + 1
    FROM employees e
    JOIN reports r
        ON e.manager_id = r.employee_id
)
SELECT *
FROM reports
ORDER BY depth, employee_id;
```

```sql
-- Generate a date series without a calendar table
WITH RECURSIVE date_series AS (
    SELECT CAST('2024-01-01' AS DATE) AS dt

    UNION ALL

    SELECT dt + INTERVAL '1 day'
    FROM date_series
    WHERE dt < '2024-12-31'
)
SELECT dt
FROM date_series;
```

!!! warning
    Always make sure recursive CTEs have a termination condition — a `WHERE` clause that eventually returns no rows. Without it, the query will run forever.

---

## Debugging CTEs

Because each CTE is a named block, you can test them individually — just run the CTE definition with a direct `SELECT *` at the end.

```sql
-- Run just this CTE to verify its output before building on top of it
WITH monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', order_date) AS month,
        SUM(amount) AS revenue
    FROM orders
    GROUP BY 1
)
SELECT * FROM monthly_revenue ORDER BY month;
```

Build your query one CTE at a time. Confirm each step before adding the next.

---

## Best Practices

- **One responsibility per CTE** — if a CTE does two things, split it
- **Name CTEs for what they contain**, not what they do: `high_value_customers` not `get_customers_with_high_value`
- **Build step by step** — test each CTE individually before chaining
- **Prefer CTEs over nested subqueries** for anything more than one level of nesting
- **Don't over-CTE** — a simple filter or join doesn't need its own CTE
- **Add a short comment above each CTE** in complex queries — future you will be grateful

```sql
WITH
-- Step 1: filter to completed orders in the current year
base_orders AS (
    SELECT customer_id, amount
    FROM orders
    WHERE status = 'completed'
      AND EXTRACT(YEAR FROM order_date) = EXTRACT(YEAR FROM CURRENT_DATE)
),
-- Step 2: aggregate to one row per customer
customer_totals AS (
    SELECT customer_id, SUM(amount) AS total
    FROM base_orders
    GROUP BY customer_id
)
SELECT *
FROM customer_totals
ORDER BY total DESC;
```