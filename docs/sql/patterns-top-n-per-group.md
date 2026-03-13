---
title: Top-N per Group
description: Retrieving the top N rows per category or group with SQL window functions
tags: [sql, patterns, top-n, window-functions]
---

# Top-N per Group

The Top-N per group pattern retrieves the highest (or lowest) N rows within each partition of a dataset — the top 3 products per category, the 5 best-performing salespeople per region, the 10 most recent orders per customer.

---

## Basic Top-N — Entire Table

```sql
-- Top 5 orders by amount
SELECT order_id, customer_id, amount
FROM orders
ORDER BY amount DESC
FETCH FIRST 5 ROWS ONLY;
```

---

## Top-N per Group — ROW_NUMBER

Use `ROW_NUMBER()` to rank rows within each group, then filter.

```sql
-- Top 3 employees by salary per department
SELECT *
FROM (
    SELECT
        employee_id,
        department,
        salary,
        ROW_NUMBER() OVER (
            PARTITION BY department
            ORDER BY salary DESC
        ) AS rn
    FROM employees
) ranked
WHERE rn <= 3
ORDER BY department, salary DESC;
```

---

## With a CTE

```sql
WITH ranked AS (
    SELECT
        employee_id,
        department,
        salary,
        ROW_NUMBER() OVER (
            PARTITION BY department
            ORDER BY salary DESC
        ) AS rn
    FROM employees
)
SELECT employee_id, department, salary
FROM ranked
WHERE rn <= 3
ORDER BY department, rn;
```

---

## ROW_NUMBER vs RANK vs DENSE_RANK for Top-N

The choice of ranking function affects how ties are handled.

```sql
WITH ranked AS (
    SELECT
        employee_id,
        department,
        salary,
        ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num,
        RANK()       OVER (PARTITION BY department ORDER BY salary DESC) AS rnk,
        DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rnk
    FROM employees
)
SELECT * FROM ranked WHERE rnk <= 3;
```

| Function | Ties | Result for Top-3 with ties at position 2 |
|---|---|---|
| `ROW_NUMBER()` | Breaks ties arbitrarily | Always exactly 3 rows |
| `RANK()` | Same rank for ties, gaps after | May return 4+ rows (positions 1, 2, 2, 4) |
| `DENSE_RANK()` | Same rank, no gaps | May return 4+ rows (positions 1, 2, 2, 3) |

!!! tip
    Use `ROW_NUMBER()` when you need exactly N rows per group. Use `RANK()` or `DENSE_RANK()` when ties should be included together.

---

## Bottom-N per Group

Reverse the `ORDER BY` direction to get the lowest values.

```sql
-- 3 lowest-paid employees per department
WITH ranked AS (
    SELECT
        employee_id,
        department,
        salary,
        ROW_NUMBER() OVER (
            PARTITION BY department
            ORDER BY salary ASC  -- ascending for bottom-N
        ) AS rn
    FROM employees
)
SELECT employee_id, department, salary
FROM ranked
WHERE rn <= 3;
```

---

## Top-N with Additional Filters

Filter the base data before ranking — only rank within the relevant subset.

```sql
-- Top 5 completed orders per customer in 2024
WITH ranked AS (
    SELECT
        customer_id,
        order_id,
        amount,
        order_date,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY amount DESC
        ) AS rn
    FROM orders
    WHERE status = 'completed'
      AND EXTRACT(YEAR FROM order_date) = 2024
)
SELECT customer_id, order_id, amount, order_date
FROM ranked
WHERE rn <= 5
ORDER BY customer_id, rn;
```

---

## Top-1 per Group — Latest or Best

A special case of Top-N — the single best row per group. Also see the **Latest Record per Group** page.

```sql
-- Best-selling product per category
WITH ranked AS (
    SELECT
        category,
        product_name,
        total_sold,
        ROW_NUMBER() OVER (
            PARTITION BY category
            ORDER BY total_sold DESC
        ) AS rn
    FROM product_sales_summary
)
SELECT category, product_name, total_sold
FROM ranked
WHERE rn = 1;
```

---

## Practical Examples

### Top 3 Products by Revenue per Category

```sql
WITH product_revenue AS (
    SELECT
        p.category,
        p.product_name,
        SUM(oi.quantity * oi.unit_price) AS revenue
    FROM order_items oi
    JOIN products p ON oi.product_id = p.product_id
    GROUP BY p.category, p.product_name
),
ranked AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY category
            ORDER BY revenue DESC
        ) AS rn
    FROM product_revenue
)
SELECT category, product_name, revenue
FROM ranked
WHERE rn <= 3
ORDER BY category, rn;
```

### Most Recent 5 Orders per Customer

```sql
WITH ranked AS (
    SELECT
        customer_id,
        order_id,
        order_date,
        amount,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY order_date DESC
        ) AS rn
    FROM orders
)
SELECT customer_id, order_id, order_date, amount
FROM ranked
WHERE rn <= 5
ORDER BY customer_id, order_date DESC;
```

---

## Best Practices

- Use `ROW_NUMBER()` for strict Top-N (exactly N rows). Use `RANK()` when ties should be included
- Always filter base data before ranking — rank only the relevant subset
- Add a tiebreaker to `ORDER BY` when using `ROW_NUMBER()` to ensure deterministic results
- Index the partition key and order column for performance on large tables
- Aggregate first in a CTE, then rank — avoid ranking raw unaggregated tables when you need summary-based rankings
