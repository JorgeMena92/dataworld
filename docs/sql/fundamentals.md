---
title: Fundamentals
description: Essential clauses to start with SQL
tags: [sql, fundamentals]
---

# Fundamentals

The seven core clauses every SQL query is built on. Learn these well and you can answer most business questions with data.

---

## Order of the Clauses

SQL clauses are written in a specific order, but executed in a different one. Understanding this helps you write correct queries and debug errors faster.

| Coding order | Execution order |
|---|---|
| 1. SELECT | 1. FROM |
| 2. FROM | 2. WHERE |
| 3. WHERE | 3. GROUP BY |
| 4. GROUP BY | 4. HAVING |
| 5. HAVING | 5. SELECT |
| 6. ORDER BY | 6. ORDER BY |
| 7. FETCH / LIMIT | 7. FETCH / LIMIT |

---

## 1. SELECT

Choose which columns to return. You can select specific columns, all columns with `*`, or create new ones with expressions and aliases.

```sql
-- Specific columns
SELECT first_name, last_name, salary
FROM employees;

-- All columns
SELECT *
FROM employees;

-- With alias and expression
SELECT
    first_name,
    salary * 12 AS annual_salary
FROM employees;
```

---

## 2. FROM

Specify the source table or view. This is the starting point of every query — SQL reads from here first.

```sql
SELECT *
FROM orders;

-- You can also alias the table for shorter references
SELECT o.order_id, o.amount
FROM orders o;
```

---

## 3. WHERE

Filter rows **before** any grouping or aggregation. Only rows that match the condition are kept.

```sql
-- Single condition
SELECT *
FROM employees
WHERE department = 'Engineering';

-- Multiple conditions
SELECT *
FROM orders
WHERE amount > 500
  AND status = 'completed';

-- Pattern matching
SELECT *
FROM customers
WHERE email LIKE '%@gmail.com';
```

!!! tip
    `WHERE` runs before `GROUP BY`. You cannot use aggregate functions like `SUM()` or `COUNT()` inside `WHERE` — use `HAVING` for that.

---

## 4. GROUP BY

Group rows that share the same values in one or more columns, usually to apply an aggregate function to each group.

```sql
-- Total sales per department
SELECT
    department,
    SUM(salary) AS total_salary,
    COUNT(*) AS headcount
FROM employees
GROUP BY department;

-- Group by multiple columns
SELECT
    department,
    job_title,
    AVG(salary) AS avg_salary
FROM employees
GROUP BY department, job_title;
```

!!! tip
    Every column in `SELECT` must either be in `GROUP BY` or wrapped in an aggregate function.

---

## 5. HAVING

Filter rows **after** grouping. Use it when you need to filter based on the result of an aggregate function.

```sql
-- Only departments with more than 5 employees
SELECT
    department,
    COUNT(*) AS headcount
FROM employees
GROUP BY department
HAVING COUNT(*) > 5;

-- Departments where total salary exceeds a threshold
SELECT
    department,
    SUM(salary) AS total_salary
FROM employees
GROUP BY department
HAVING SUM(salary) > 100000;
```

---

## 6. ORDER BY

Sort the final result set. Use `ASC` for ascending (default) or `DESC` for descending.

```sql
-- Sort by salary descending
SELECT first_name, salary
FROM employees
ORDER BY salary DESC;

-- Sort by multiple columns
SELECT first_name, department, salary
FROM employees
ORDER BY department ASC, salary DESC;

-- Sort by column position (less readable, avoid in production)
SELECT first_name, salary
FROM employees
ORDER BY 2 DESC;
```

---

## 7. FETCH / LIMIT

Restrict how many rows are returned. Useful for previewing data, building top-N reports, or paginating results.

The ANSI SQL standard uses `FETCH FIRST`:

```sql
-- ANSI SQL — works in SQL Server, PostgreSQL, Oracle, DB2
SELECT first_name, salary
FROM employees
ORDER BY salary DESC
FETCH FIRST 10 ROWS ONLY;
```

`LIMIT` is a widely supported shorthand, but it is not ANSI SQL:

```sql
-- Non-ANSI — works in PostgreSQL, MySQL, SQLite, Snowflake
SELECT first_name, salary
FROM employees
ORDER BY salary DESC
LIMIT 10;
```

For pagination, combine with `OFFSET`:

```sql
-- ANSI SQL — skip the first 10 rows, return the next 10
SELECT first_name, salary
FROM employees
ORDER BY salary DESC
OFFSET 10 ROWS
FETCH NEXT 10 ROWS ONLY;

-- Non-ANSI equivalent
SELECT first_name, salary
FROM employees
ORDER BY salary DESC
LIMIT 10 OFFSET 10;
```

!!! warning
    Always use `ORDER BY` with `FETCH` / `LIMIT`. Without it, the database returns rows in an undefined order and results will be inconsistent across runs.

!!! tip
    Use `FETCH FIRST n ROWS ONLY` when writing portable SQL. Use `LIMIT` only when you know the target platform supports it.

---

## Putting It All Together

A query using all seven clauses:

```sql
SELECT
    department,
    COUNT(*)        AS headcount,
    AVG(salary)     AS avg_salary,
    SUM(salary)     AS total_salary
FROM employees
WHERE status = 'active'
GROUP BY department
HAVING COUNT(*) > 3
ORDER BY total_salary DESC
FETCH FIRST 5 ROWS ONLY;
```

This query reads as: *"For active employees, group by department, keep only departments with more than 3 people, sort by total salary, and return the top 5."*

---

!!! tip
    Master these seven clauses before moving on to JOINs, CTEs, or window functions. Every advanced SQL pattern builds on top of this foundation.