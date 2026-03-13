---
title: JOINs
description: Combining rows from multiple tables using SQL JOINs
tags: [sql, dql, joins]
---

# JOINs

JOINs combine rows from two or more tables based on a related column. They are the primary way to work with normalized data — where information is split across multiple tables by design.

---

## JOIN Types Overview

| Type | Returns |
|---|---|
| `INNER JOIN` | Only rows that match in both tables |
| `LEFT JOIN` | All rows from the left table, matching rows from the right |
| `RIGHT JOIN` | All rows from the right table, matching rows from the left |
| `FULL OUTER JOIN` | All rows from both tables |
| `CROSS JOIN` | Every combination of rows from both tables |
| `SELF JOIN` | A table joined to itself |

---

## INNER JOIN

Returns only rows where the join condition matches in **both** tables. The most common join type.

```sql
SELECT
    o.order_id,
    c.first_name,
    c.last_name,
    o.amount
FROM orders o
INNER JOIN customers c
    ON o.customer_id = c.customer_id;
```

Rows without a match on either side are excluded. If a customer has no orders, they do not appear. If an order has no matching customer, it does not appear either.

---

## LEFT JOIN

Returns **all rows from the left table**, plus matching rows from the right. If there is no match, the right side columns return `NULL`.

```sql
-- All customers, with their orders if they have any
SELECT
    c.customer_id,
    c.first_name,
    o.order_id,
    o.amount
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id;
```

### Find rows with no match

```sql
-- Customers who have never placed an order
SELECT c.customer_id, c.first_name
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;
```

!!! tip
    This pattern — `LEFT JOIN` + `WHERE right_table.id IS NULL` — is one of the most useful in SQL. It finds rows in one table that have no corresponding row in another.

---

## RIGHT JOIN

Returns **all rows from the right table**, plus matching rows from the left. In practice, a `RIGHT JOIN` can always be rewritten as a `LEFT JOIN` by swapping the table order — most teams prefer `LEFT JOIN` for consistency.

```sql
-- Equivalent queries
SELECT * FROM orders o RIGHT JOIN customers c ON o.customer_id = c.customer_id;
SELECT * FROM customers c LEFT JOIN orders o ON c.customer_id = o.customer_id;
```

---

## FULL OUTER JOIN

Returns **all rows from both tables**. Where there is no match, the missing side returns `NULL`.

```sql
-- All customers and all orders, matched where possible
SELECT
    c.customer_id,
    c.first_name,
    o.order_id,
    o.amount
FROM customers c
FULL OUTER JOIN orders o
    ON c.customer_id = o.customer_id;
```

Useful for reconciliation — finding rows that exist in one table but not the other, or comparing two datasets for differences.

!!! warning
    `FULL OUTER JOIN` is not supported in MySQL. Use a `LEFT JOIN UNION ALL RIGHT JOIN WHERE left IS NULL` workaround instead.

---

## CROSS JOIN

Returns the **Cartesian product** — every row from the left table combined with every row from the right table. No join condition needed.

```sql
-- Combine every product with every region
SELECT
    p.product_name,
    r.region_name
FROM products p
CROSS JOIN regions r;
```

If `products` has 10 rows and `regions` has 5 rows, the result has 50 rows. Use with care on large tables.

A common use case is generating a complete date or dimension spine to fill gaps in data.

---

## SELF JOIN

A table joined to itself — useful for hierarchical data like org charts or comparing rows within the same table.

```sql
-- Show each employee alongside their manager
SELECT
    e.first_name        AS employee,
    m.first_name        AS manager
FROM employees e
LEFT JOIN employees m
    ON e.manager_id = m.employee_id;
```

---

## Joining on Multiple Columns

```sql
-- Match on more than one column
SELECT *
FROM sales s
JOIN targets t
    ON s.region = t.region
    AND s.year  = t.year
    AND s.month = t.month;
```

---

## Joining More Than Two Tables

```sql
SELECT
    o.order_id,
    c.first_name,
    p.product_name,
    oi.quantity,
    oi.unit_price
FROM orders o
JOIN customers c
    ON o.customer_id = c.customer_id
JOIN order_items oi
    ON o.order_id = oi.order_id
JOIN products p
    ON oi.product_id = p.product_id;
```

!!! tip
    Always alias tables when joining more than two. It makes column references unambiguous and queries far easier to read.

---

## JOIN vs Subquery

Both can produce the same result — choose based on readability and performance.

```sql
-- Subquery approach
SELECT *
FROM orders
WHERE customer_id IN (
    SELECT customer_id
    FROM customers
    WHERE country = 'Peru'
);

-- JOIN approach
SELECT o.*
FROM orders o
JOIN customers c
    ON o.customer_id = c.customer_id
WHERE c.country = 'Peru';
```

The `JOIN` approach is generally preferred — it is more readable and gives the query optimizer more flexibility.

---

## Best Practices

- Always use explicit `JOIN` syntax — never use comma-separated tables in `FROM`
- Always alias tables in multi-table queries
- Use `LEFT JOIN` instead of `RIGHT JOIN` for consistency — swap the table order if needed
- Filter on the joined table in `ON` when possible, not in `WHERE`, to avoid accidentally converting a `LEFT JOIN` into an `INNER JOIN`
- Be careful with `CROSS JOIN` and `FULL OUTER JOIN` on large tables — they can produce very large result sets

```sql
-- This accidentally turns a LEFT JOIN into an INNER JOIN
SELECT *
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed';   -- excludes NULLs from left join

-- Correct — filter in the ON clause to preserve all customers
SELECT *
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id
    AND o.status = 'completed';
```