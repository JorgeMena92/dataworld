---
title: SQL
description: 12 ANSI SQL features every strong SQL engineer should know
# tags: [sql, fundamentals]
hide:
  - toc
---

# SQL

12 ANSI SQL Features Every Strong SQL Engineer Should Know.

This document is written like a `.sql` study sheet — each section uses SQL comments for headers and descriptions, followed by a short example query.

---

## Most people know

### 1. JOINs

Combine rows from related tables using matching keys. Master `INNER JOIN` and `LEFT JOIN` first, then understand `RIGHT`, `FULL`, and `CROSS JOIN`.

```sql
SELECT
    o.order_id,
    c.customer_name
FROM orders o
JOIN customers c
    ON o.customer_id = c.customer_id;
```

### 2. Aggregations

Summarize data with functions like `SUM`, `COUNT`, `AVG`, `MIN`, and `MAX`.

```sql
SELECT
    department,
    SUM(salary) AS total_salary
FROM employees
GROUP BY department;
```

### 3. GROUP BY + HAVING

Use `GROUP BY` to aggregate and `HAVING` to filter aggregated results after the grouping step.

```sql
SELECT
    customer_id,
    SUM(amount) AS total_sales
FROM sales
GROUP BY customer_id
HAVING SUM(amount) > 1000;
```

### 4. CASE expressions

Add conditional logic directly inside SQL for bucketing, flags, labels, and business rules.

```sql
SELECT
    order_id,
    CASE
        WHEN amount > 1000 THEN 'High'
        WHEN amount > 500  THEN 'Medium'
        ELSE 'Low'
    END AS order_category
FROM orders;
```

---

## Good engineers know

### 5. Common Table Expressions (CTEs)

Use CTEs to break complex logic into readable steps. They improve maintainability and make debugging easier.

```sql
WITH sales_per_customer AS (
    SELECT
        customer_id,
        SUM(amount) AS total_sales
    FROM sales
    GROUP BY customer_id
)
SELECT *
FROM sales_per_customer
WHERE total_sales > 1000;
```

### 6. Window functions

Perform calculations across a related set of rows without collapsing the result set.

```sql
SELECT
    employee_id,
    department,
    salary,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg_salary
FROM employees;
```

### 7. ROW_NUMBER() for deduplication

Use `ROW_NUMBER()` to keep the latest or best record per business key. This is one of the most common ETL patterns.

```sql
SELECT *
FROM (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY user_id
            ORDER BY login_date DESC
        ) AS rn
    FROM logins
) t
WHERE rn = 1;
```

### 8. Subqueries

Use subqueries when you need one query to depend on the result of another query.

```sql
SELECT *
FROM employees
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
);
```

### 9. EXISTS

Use `EXISTS` to test whether related rows are present. It is often a clean and efficient way to filter large datasets.

```sql
SELECT *
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

---

## Senior engineers know

### 10. UNION vs UNION ALL

Use `UNION` when you need duplicate removal. Use `UNION ALL` when you want all rows and better performance.

```sql
SELECT city FROM customers
UNION ALL
SELECT city FROM suppliers;
```

### 11. Set operations

Use `INTERSECT` and `EXCEPT` to compare result sets cleanly without forcing everything into joins.

```sql
-- Users who logged in AND made a purchase
SELECT user_id FROM logins
INTERSECT
SELECT user_id FROM purchases;

-- Users who logged in but did NOT make a purchase
SELECT user_id FROM logins
EXCEPT
SELECT user_id FROM purchases;
```

### 12. Recursive CTEs

Use recursive CTEs for hierarchies such as org charts, category trees, and parent-child navigation.

```sql
WITH RECURSIVE org_chart AS (
    SELECT employee_id, manager_id
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.employee_id, e.manager_id
    FROM employees e
    JOIN org_chart o
        ON e.manager_id = o.employee_id
)
SELECT *
FROM org_chart;
```

---

!!! tip
    Write ANSI SQL first, then learn each platform's extensions only when needed for performance, procedural logic, or engine-specific features.