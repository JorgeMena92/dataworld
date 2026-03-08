---
title: DML — Data Manipulation Language
description: SELECT, INSERT, UPDATE, DELETE with examples
tags: [sql, dml]
---

# DML — Data Manipulation Language

DML commands work with the **data inside** tables — querying, inserting, modifying, and deleting rows. This is where most day-to-day SQL work happens.

---

## SELECT

Retrieves data from one or more tables. The most used SQL command by far.

```sql
-- All columns
SELECT * FROM customers;

-- Specific columns
SELECT first_name, last_name, email
FROM customers;

-- With alias
SELECT
    first_name AS name,
    salary * 12 AS annual_salary
FROM employees;

-- With filter
SELECT *
FROM orders
WHERE status = 'completed'
  AND amount > 100;

-- With sorting
SELECT *
FROM orders
ORDER BY created_at DESC;

-- With limit
SELECT *
FROM orders
ORDER BY created_at DESC
LIMIT 10;
```

### Filtering with WHERE

```sql
-- Comparison operators
WHERE salary > 50000
WHERE status = 'active'
WHERE created_at >= '2024-01-01'

-- Multiple conditions
WHERE department = 'Sales' AND salary > 40000
WHERE status = 'active' OR status = 'pending'

-- IN — match a list of values
WHERE country IN ('Peru', 'Chile', 'Colombia')

-- BETWEEN — inclusive range
WHERE salary BETWEEN 30000 AND 60000

-- LIKE — pattern matching
WHERE email LIKE '%@gmail.com'
WHERE name LIKE 'Jorge%'

-- IS NULL / IS NOT NULL
WHERE phone IS NULL
WHERE email IS NOT NULL
```

### Aggregations

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
HAVING COUNT(*) > 3
ORDER BY total_salary DESC;
```

### JOINs

```sql
-- INNER JOIN — only matching rows
SELECT o.order_id, c.first_name, o.amount
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id;

-- LEFT JOIN — all rows from left, matching from right
SELECT c.first_name, o.order_id
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id;

-- FULL OUTER JOIN — all rows from both
SELECT c.first_name, o.order_id
FROM customers c
FULL OUTER JOIN orders o ON c.customer_id = o.customer_id;
```

---

## INSERT

Adds new rows to a table.

```sql
-- Insert a single row
INSERT INTO customers (first_name, last_name, email)
VALUES ('Jorge', 'Mena', 'jorge@example.com');

-- Insert multiple rows
INSERT INTO customers (first_name, last_name, email)
VALUES
    ('Ana', 'García', 'ana@example.com'),
    ('Luis', 'Pérez', 'luis@example.com');

-- Insert from a SELECT (copy data from another table)
INSERT INTO customers_archive
SELECT * FROM customers
WHERE created_at < '2023-01-01';
```

---

## UPDATE

Modifies existing rows in a table.

```sql
-- Update a single column
UPDATE customers
SET email = 'new@example.com'
WHERE customer_id = 42;

-- Update multiple columns
UPDATE employees
SET salary = salary * 1.10,
    updated_at = CURRENT_TIMESTAMP
WHERE department = 'Engineering'
  AND performance_rating = 'Excellent';

-- Update using a subquery
UPDATE orders
SET status = 'flagged'
WHERE customer_id IN (
    SELECT customer_id
    FROM customers
    WHERE country = 'Unknown'
);
```

!!! warning
    Always include a `WHERE` clause in `UPDATE`. Without it, every row in the table will be updated.

---

## DELETE

Removes rows from a table.

```sql
-- Delete specific rows
DELETE FROM orders
WHERE status = 'cancelled'
  AND created_at < '2023-01-01';

-- Delete using a subquery
DELETE FROM orders
WHERE customer_id IN (
    SELECT customer_id
    FROM customers
    WHERE status = 'inactive'
);

-- Delete all rows (slower than TRUNCATE)
DELETE FROM temp_log;
```

!!! danger
    Always include a `WHERE` clause in `DELETE`. Without it, every row in the table will be deleted. Test your `WHERE` clause with a `SELECT` first before running the delete.

---

## Best Practices

- Always test `UPDATE` and `DELETE` with a `SELECT` using the same `WHERE` clause first
- Use transactions when making multiple related changes
- Avoid `SELECT *` in production queries — select only the columns you need
- Use aliases to make queries more readable
- Filter early — reduce rows as soon as possible in the query

```sql
-- Safe pattern for destructive operations
BEGIN;

-- Test first
SELECT COUNT(*) FROM orders WHERE status = 'cancelled';

-- Then delete
DELETE FROM orders WHERE status = 'cancelled';

-- Review result, then commit or rollback
COMMIT;
-- or ROLLBACK;
```
