---
title: Intermediate
description: Subqueries, CTEs, window functions, and real-world patterns
tags: [sql, intermediate]
---

# Intermediate

Patterns that separate good SQL writers from basic ones. These are the techniques used daily in data engineering and analytics work.

---

## Subqueries

A query nested inside another query. Useful when one query depends on the result of another.

```sql
-- Filter using a subquery result
SELECT *
FROM employees
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
);

-- Subquery in FROM (derived table)
SELECT department, avg_salary
FROM (
    SELECT
        department,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
) dept_summary
WHERE avg_salary > 50000;

-- Correlated subquery (references outer query)
SELECT
    e.employee_id,
    e.first_name,
    e.salary
FROM employees e
WHERE e.salary = (
    SELECT MAX(salary)
    FROM employees
    WHERE department = e.department
);
```

---

## CTEs — Common Table Expressions

CTEs break complex queries into readable, named steps. They are defined with `WITH` and referenced like temporary tables.

```sql
-- Single CTE
WITH high_value_customers AS (
    SELECT
        customer_id,
        SUM(amount) AS total_spent
    FROM orders
    GROUP BY customer_id
    HAVING SUM(amount) > 5000
)
SELECT c.first_name, c.email, h.total_spent
FROM customers c
JOIN high_value_customers h
    ON c.customer_id = h.customer_id;

-- Multiple CTEs chained together
WITH monthly_sales AS (
    SELECT
        DATE_TRUNC('month', order_date) AS month,
        SUM(amount) AS total
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
    Prefer CTEs over deeply nested subqueries. They are easier to read, debug, and maintain.

---

## Window Functions

Perform calculations across a set of rows related to the current row — without collapsing the result like GROUP BY does.

```sql
-- Syntax
function() OVER (
    PARTITION BY column   -- group rows (optional)
    ORDER BY column       -- define order within partition
    ROWS/RANGE BETWEEN... -- define frame (optional)
)
```

### Ranking functions

```sql
SELECT
    employee_id,
    department,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num,
    RANK()       OVER (PARTITION BY department ORDER BY salary DESC) AS rank,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rank
FROM employees;
```

| Function | Ties | Gaps |
|---|---|---|
| `ROW_NUMBER()` | Unique numbers always | — |
| `RANK()` | Same rank for ties | Gaps after ties |
| `DENSE_RANK()` | Same rank for ties | No gaps |

### Aggregate window functions

```sql
SELECT
    employee_id,
    department,
    salary,
    SUM(salary)   OVER (PARTITION BY department) AS dept_total,
    AVG(salary)   OVER (PARTITION BY department) AS dept_avg,
    COUNT(*)      OVER (PARTITION BY department) AS dept_count,
    salary / SUM(salary) OVER (PARTITION BY department) AS pct_of_dept
FROM employees;
```

### LAG and LEAD

```sql
SELECT
    order_date,
    amount,
    LAG(amount)  OVER (ORDER BY order_date) AS prev_amount,
    LEAD(amount) OVER (ORDER BY order_date) AS next_amount,
    amount - LAG(amount) OVER (ORDER BY order_date) AS change
FROM daily_sales;
```

### Running totals

```sql
SELECT
    order_date,
    amount,
    SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM daily_sales;
```

---

## Deduplication with ROW_NUMBER

One of the most common patterns in data engineering — keep only one row per business key.

```sql
SELECT *
FROM (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY updated_at DESC
        ) AS rn
    FROM customers
) t
WHERE rn = 1;
```

---

## EXISTS vs IN

Both filter rows based on a related table, but `EXISTS` is often more efficient on large datasets.

```sql
-- IN
SELECT * FROM customers
WHERE customer_id IN (
    SELECT customer_id FROM orders
);

-- EXISTS (stops as soon as a match is found)
SELECT * FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

---

## UNION vs UNION ALL

```sql
-- UNION removes duplicates (slower)
SELECT city FROM customers
UNION
SELECT city FROM suppliers;

-- UNION ALL keeps all rows (faster)
SELECT city FROM customers
UNION ALL
SELECT city FROM suppliers;
```

!!! tip
    Use `UNION ALL` by default. Only use `UNION` when you specifically need deduplication — it adds a sorting step that slows down the query.

---

## Best Practices

- Break complex queries into CTEs instead of nesting subqueries
- Use window functions instead of self-joins for running totals and rankings
- Use `EXISTS` over `IN` for large subqueries
- Always use `UNION ALL` unless you need deduplication
- Name CTEs and aliases clearly — they are the documentation of your query logic
