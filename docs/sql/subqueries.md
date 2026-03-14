---
title: Subqueries
description: Nested queries, derived tables, correlated subqueries, and when to use them
tags: [sql, dql, subqueries]
---

# Subqueries

A subquery is a query nested inside another query. It lets one query depend on the result of another — filtering, computing, or providing values dynamically.

---

## Types of Subqueries

| Type | Location | Description |
|---|---|---|
| Scalar | `SELECT`, `WHERE`, `HAVING` | Returns a single value |
| Row | `WHERE` | Returns a single row |
| Table (derived) | `FROM` | Returns a result set used as a table |
| Correlated | `WHERE`, `SELECT` | References the outer query row by row |

---

## Scalar Subquery

Returns exactly one value — one row, one column. Can be used anywhere a single value is expected.

```sql
-- Filter rows against a single computed value
SELECT *
FROM employees
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
);

-- Use a scalar subquery in SELECT
SELECT
    employee_id,
    salary,
    (SELECT AVG(salary) FROM employees) AS company_avg,
    salary - (SELECT AVG(salary) FROM employees) AS diff_from_avg
FROM employees;
```

!!! warning
    If a scalar subquery returns more than one row, the query will fail. Make sure the inner query is guaranteed to return a single value.

---

## Derived Table (Subquery in FROM)

A subquery in the `FROM` clause acts as a temporary table for the outer query. Must always be aliased.

```sql
-- Summarize first, then filter the summary
SELECT department, avg_salary
FROM (
    SELECT
        department,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
) dept_summary
WHERE avg_salary > 50000
ORDER BY avg_salary DESC;
```

```sql
-- Top customers by total spend
SELECT customer_id, total_spent
FROM (
    SELECT
        customer_id,
        SUM(amount) AS total_spent
    FROM orders
    GROUP BY customer_id
) customer_totals
WHERE total_spent > 5000;
```

!!! tip
    Derived tables are functionally identical to CTEs but harder to read when nested deeply. Prefer CTEs for anything beyond a single level of nesting.

---

## Subquery in WHERE

Filter rows based on the result of another query.

```sql
-- Orders placed by customers from Peru
SELECT *
FROM orders
WHERE customer_id IN (
    SELECT customer_id
    FROM customers
    WHERE country = 'Peru'
);

-- Products that have never been ordered
SELECT *
FROM products
WHERE product_id NOT IN (
    SELECT DISTINCT product_id
    FROM order_items
    WHERE product_id IS NOT NULL  -- important: NOT IN with NULLs returns no rows
);
```

!!! warning
    Avoid `NOT IN` when the subquery might return `NULL` values — it causes the entire condition to return no rows. See [EXISTS and NOT EXISTS](#exists-and-not-exists) below for the safer alternative.

---

## EXISTS and NOT EXISTS

`EXISTS` returns true if the subquery returns at least one row. It stops scanning as soon as a match is found — often faster than `IN` on large datasets. For a conceptual introduction, see [Filtering & Conditions](filtering-conditions.md).

```sql
-- Customers who have placed at least one order
SELECT *
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);

-- Customers who have never placed an order
SELECT *
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

| | `IN` | `EXISTS` |
|---|---|---|
| Returns | A list of values | True / False |
| Stops at first match | ❌ | ✅ |
| Safe with NULLs | ❌ | ✅ |
| Readable for large subqueries | ❌ | ✅ |

---

## Correlated Subquery

A correlated subquery references a column from the outer query. It runs once **per row** of the outer query — making it powerful but potentially slow on large tables.

```sql
-- Highest-paid employee in each department
SELECT
    e.employee_id,
    e.first_name,
    e.department,
    e.salary
FROM employees e
WHERE e.salary = (
    SELECT MAX(salary)
    FROM employees
    WHERE department = e.department  -- references outer query
);
```

```sql
-- Each order with the customer's lifetime total
SELECT
    o.order_id,
    o.amount,
    (
        SELECT SUM(amount)
        FROM orders
        WHERE customer_id = o.customer_id  -- references outer query
    ) AS customer_lifetime_value
FROM orders o;
```

!!! tip
    Correlated subqueries are intuitive to write but can be slow — they execute once per row. On large tables, rewrite them as a `JOIN` or CTE with a window function for better performance.

The highest-paid per department example above can be rewritten using `RANK()` — which scans the table once instead of once per row:

```sql
-- Rewrite using a window function — much faster on large tables
SELECT employee_id, first_name, department, salary
FROM (
    SELECT
        employee_id,
        first_name,
        department,
        salary,
        RANK() OVER (
            PARTITION BY department
            ORDER BY salary DESC
        ) AS rnk
    FROM employees
) ranked
WHERE rnk = 1;
```

!!! note
    `RANK()` assigns the same rank to ties — if two employees share the highest salary in a department, both rows are returned. Use `ROW_NUMBER()` if you want exactly one row per department regardless of ties. See [Window Functions](window-functions.md) for a full explanation.

---

## Subquery vs JOIN vs CTE

All three can often produce the same result. Choose based on readability and performance.

```sql
-- Subquery
SELECT *
FROM orders
WHERE customer_id IN (
    SELECT customer_id FROM customers WHERE country = 'Peru'
);

-- JOIN (preferred for readability and optimizer flexibility)
SELECT o.*
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE c.country = 'Peru';

-- CTE (preferred for complex logic)
WITH peruvian_customers AS (
    SELECT customer_id FROM customers WHERE country = 'Peru'
)
SELECT *
FROM orders
WHERE customer_id IN (SELECT customer_id FROM peruvian_customers);
```

---

## Best Practices

- Use `EXISTS` instead of `IN` when checking for the presence of related rows
- Never use `NOT IN` against a subquery that might return NULLs — use `NOT EXISTS` instead
- Prefer CTEs over deeply nested derived tables for readability
- Avoid correlated subqueries on large tables — rewrite as JOINs or window functions when performance matters
- Always alias derived tables in `FROM` — the query will fail without an alias