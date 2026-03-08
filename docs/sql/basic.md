---
title: Basic
description: Practical SQL patterns for beginners
tags: [sql, basic]
---

# Basic

Practical SQL patterns for everyday querying. These cover the situations you will encounter most often when starting to work with data.

---

## Filtering Data

```sql
-- Basic filter
SELECT * FROM orders WHERE status = 'completed';

-- Multiple conditions with AND / OR
SELECT *
FROM employees
WHERE department = 'Sales'
  AND salary > 35000;

-- Match a list of values
SELECT *
FROM orders
WHERE country IN ('Peru', 'Chile', 'Argentina');

-- Exclude values
SELECT *
FROM orders
WHERE status NOT IN ('cancelled', 'refunded');

-- Range filter
SELECT *
FROM orders
WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';

-- Pattern matching
SELECT *
FROM customers
WHERE email LIKE '%@gmail.com';
```

---

## Sorting Results

```sql
-- Sort ascending (default)
SELECT * FROM employees ORDER BY last_name;

-- Sort descending
SELECT * FROM orders ORDER BY amount DESC;

-- Sort by multiple columns
SELECT *
FROM employees
ORDER BY department ASC, salary DESC;
```

---

## Working with NULLs

NULL means the value is unknown or missing — it is not zero, not empty string, just absent.

```sql
-- Find rows with no phone number
SELECT * FROM customers WHERE phone IS NULL;

-- Find rows that have a phone number
SELECT * FROM customers WHERE phone IS NOT NULL;

-- Replace NULL with a default value
SELECT
    first_name,
    COALESCE(phone, 'No phone') AS phone
FROM customers;
```

!!! tip
    Never use `= NULL` to filter — it always returns nothing. Always use `IS NULL` or `IS NOT NULL`.

---

## Aggregations

```sql
SELECT
    department,
    COUNT(*)        AS headcount,
    SUM(salary)     AS total_salary,
    AVG(salary)     AS avg_salary,
    MIN(salary)     AS min_salary,
    MAX(salary)     AS max_salary
FROM employees
GROUP BY department
ORDER BY total_salary DESC;
```

### COUNT variations

```sql
-- Count all rows
SELECT COUNT(*) FROM orders;

-- Count non-null values in a column
SELECT COUNT(email) FROM customers;

-- Count distinct values
SELECT COUNT(DISTINCT customer_id) FROM orders;
```

---

## Basic JOINs

```sql
-- INNER JOIN — only rows that match in both tables
SELECT
    o.order_id,
    c.first_name,
    o.amount
FROM orders o
INNER JOIN customers c
    ON o.customer_id = c.customer_id;

-- LEFT JOIN — all orders, with customer info if available
SELECT
    o.order_id,
    c.first_name,
    o.amount
FROM orders o
LEFT JOIN customers c
    ON o.customer_id = c.customer_id;
```

---

## CASE WHEN

Add conditional logic directly inside a query.

```sql
SELECT
    order_id,
    amount,
    CASE
        WHEN amount >= 1000 THEN 'High'
        WHEN amount >= 500  THEN 'Medium'
        ELSE 'Low'
    END AS order_tier
FROM orders;
```

---

## Useful String Functions

```sql
-- Uppercase / lowercase
SELECT UPPER(first_name), LOWER(email) FROM customers;

-- Trim whitespace
SELECT TRIM(first_name) FROM customers;

-- Concatenate
SELECT first_name || ' ' || last_name AS full_name FROM customers;

-- String length
SELECT first_name, LENGTH(first_name) AS name_length FROM customers;

-- Substring
SELECT SUBSTRING(email, 1, 5) AS email_prefix FROM customers;
```

---

## Useful Date Functions

```sql
-- Current date and time
SELECT CURRENT_DATE, CURRENT_TIMESTAMP;

-- Extract parts of a date
SELECT
    order_id,
    EXTRACT(YEAR  FROM created_at) AS year,
    EXTRACT(MONTH FROM created_at) AS month,
    EXTRACT(DAY   FROM created_at) AS day
FROM orders;

-- Date difference (days between two dates)
SELECT
    order_id,
    created_at,
    shipped_at,
    shipped_at - created_at AS days_to_ship
FROM orders;
```

---

## Best Practices

- Always alias columns when using expressions or functions
- Use table aliases to keep multi-table queries readable
- Filter data as early as possible to reduce the rows processed
- Format your SQL with consistent indentation — future you will thank you
