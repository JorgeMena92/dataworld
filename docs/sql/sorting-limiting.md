---
title: Sorting & Limiting
description: ORDER BY, FETCH FIRST, OFFSET, and pagination in ANSI SQL
tags: [sql, dql, sorting, limiting]
---

# Sorting & Limiting

Sorting controls the order of results. Limiting controls how many rows are returned. Together they power rankings, top-N queries, and pagination.

---

## ORDER BY

`ORDER BY` sorts the final result set. It is the last clause to execute — after `SELECT`, `GROUP BY`, and `HAVING`.

```sql
-- Ascending (default)
SELECT * FROM employees ORDER BY last_name;

-- Descending
SELECT * FROM orders ORDER BY amount DESC;

-- Multiple columns — sort by first, then by second within ties
SELECT *
FROM employees
ORDER BY department ASC, salary DESC;
```

### ORDER BY with Expressions

```sql
-- Sort by a calculated value — ANSI SQL, always portable
SELECT
    first_name,
    last_name,
    salary * 12 AS annual_salary
FROM employees
ORDER BY salary * 12 DESC;
```

!!! warning
    Sorting by a column alias (`ORDER BY annual_salary DESC`) is supported in most databases but is not strict ANSI SQL. Repeat the expression in `ORDER BY` for portable queries.

### ORDER BY with CASE WHEN

```sql
-- Custom sort order
SELECT order_id, status
FROM orders
ORDER BY
    CASE status
        WHEN 'urgent'    THEN 1
        WHEN 'pending'   THEN 2
        WHEN 'completed' THEN 3
        ELSE 4
    END;
```

### NULL Ordering

```sql
-- ANSI SQL — NULLs sort last by default in ASC, first in DESC
-- Control explicitly with NULLS FIRST / NULLS LAST
SELECT * FROM employees ORDER BY manager_id ASC NULLS LAST;
SELECT * FROM employees ORDER BY manager_id DESC NULLS FIRST;
```

!!! warning
    `NULLS FIRST` / `NULLS LAST` is supported in PostgreSQL, Oracle, and Snowflake but not in SQL Server or MySQL.

---

## FETCH FIRST — ANSI Standard Row Limiting

`FETCH FIRST` is the ANSI SQL standard way to limit the number of rows returned.

```sql
-- Return only the first 10 rows
SELECT *
FROM orders
ORDER BY created_at DESC
FETCH FIRST 10 ROWS ONLY;

-- Return rows 11 through 20 (with OFFSET)
SELECT *
FROM orders
ORDER BY created_at DESC
OFFSET 10 ROWS
FETCH NEXT 10 ROWS ONLY;
```

!!! tip
    Always pair `FETCH FIRST` with `ORDER BY` — without it, the database can return any rows and the result is non-deterministic.

---

## OFFSET — Skipping Rows

`OFFSET` skips a number of rows before returning results. Combined with `FETCH FIRST`, it enables pagination.

```sql
-- Page 1 — rows 1 to 10
SELECT * FROM products
ORDER BY product_name
OFFSET 0 ROWS FETCH NEXT 10 ROWS ONLY;

-- Page 2 — rows 11 to 20
SELECT * FROM products
ORDER BY product_name
OFFSET 10 ROWS FETCH NEXT 10 ROWS ONLY;

-- Page N
-- OFFSET (page_number - 1) * page_size ROWS FETCH NEXT page_size ROWS ONLY
```

!!! warning
    `OFFSET` pagination becomes slow on large tables — the database still reads and discards all skipped rows. For large datasets, use keyset pagination instead (filter by the last seen ID or timestamp).

---

## Top-N Queries

Return the highest or lowest N values — a very common analytics pattern.

```sql
-- Top 5 orders by amount
SELECT order_id, amount
FROM orders
ORDER BY amount DESC
FETCH FIRST 5 ROWS ONLY;
```

For **Top-N per group** — for example, the top 3 employees by salary within each department — `FETCH FIRST` alone is not enough. That requires `ROW_NUMBER()`, a window function that assigns a sequential number to each row within a partition:

```sql
-- Top 3 employees per department by salary
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
WHERE rn <= 3;
```

!!! note
    `ROW_NUMBER()` is a window function. If you are not familiar with window functions yet, the pattern above assigns a rank number per group (`PARTITION BY department`) and the outer query filters to keep only the top rows. See [Window Functions](window-functions.md) for a full explanation.

---

## Vendor Differences

Row limiting is one of the most inconsistent areas across SQL platforms.

| Database | Syntax |
|---|---|
| ANSI SQL | `FETCH FIRST n ROWS ONLY` |
| PostgreSQL | `LIMIT n` / `FETCH FIRST n ROWS ONLY` |
| SQL Server | `TOP n` / `FETCH FIRST n ROWS ONLY` (2012+) |
| MySQL | `LIMIT n` |
| Oracle | `FETCH FIRST n ROWS ONLY` (12c+) / `ROWNUM` (legacy) |
| Snowflake | `LIMIT n` / `FETCH FIRST n ROWS ONLY` |
| BigQuery | `LIMIT n` |

```sql
-- ANSI SQL (portable)
SELECT * FROM orders ORDER BY created_at DESC
FETCH FIRST 10 ROWS ONLY;

-- SQL Server legacy
SELECT TOP 10 * FROM orders ORDER BY created_at DESC;

-- MySQL / PostgreSQL / Snowflake
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;

-- Oracle legacy
SELECT * FROM orders WHERE ROWNUM <= 10 ORDER BY created_at DESC;
-- Note: ROWNUM filters before ORDER BY in Oracle — use a subquery
```

!!! tip
    Use `FETCH FIRST n ROWS ONLY` when writing portable SQL. It is supported in SQL Server 2012+, PostgreSQL, Oracle 12c+, Snowflake, and BigQuery.

---

## Best Practices

- Always include `ORDER BY` when using `FETCH FIRST` or `LIMIT` — results without ordering are non-deterministic
- Use `FETCH FIRST` over `LIMIT` or `TOP` for cross-platform portability
- Repeat expressions in `ORDER BY` rather than referencing aliases — alias sorting is not ANSI SQL
- Use `ROW_NUMBER()` for Top-N per group — `FETCH FIRST` only limits the overall result set
- Avoid deep `OFFSET` pagination on large tables — switch to keyset pagination for performance
- Use `NULLS FIRST` / `NULLS LAST` explicitly when NULLs in sorted columns matter for correctness