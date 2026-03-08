---
title: ANSI SQL
description: Standards, portability, and best practices
tags: [sql, ansi]
---

# ANSI SQL

ANSI SQL is the standardized version of SQL defined by the American National Standards Institute. It is the common foundation that all major relational databases implement, with their own extensions on top.

---

## What is ANSI SQL?

ANSI SQL defines the core syntax and behavior that SQL engines agree to support. When you write ANSI-compliant SQL, your queries are portable — they work across PostgreSQL, SQL Server, MySQL, Oracle, Snowflake, and others with minimal changes.

The standard has been updated several times:

| Version | Year | Key additions |
|---|---|---|
| SQL-86 | 1986 | First standard |
| SQL-92 | 1992 | JOINs, subqueries, CASE |
| SQL:1999 | 1999 | Recursive CTEs, triggers, OOP types |
| SQL:2003 | 2003 | Window functions, MERGE |
| SQL:2008 | 2008 | TRUNCATE, FETCH |
| SQL:2011 | 2011 | Temporal tables |
| SQL:2016 | 2016 | JSON support |

---

## ANSI SQL vs Vendor Extensions

Each database adds its own extensions on top of the standard. These are powerful but reduce portability.

| Feature | ANSI SQL | SQL Server (T-SQL) | PostgreSQL | MySQL |
|---|---|---|---|---|
| Limit rows | `FETCH FIRST n ROWS` | `TOP n` | `LIMIT n` | `LIMIT n` |
| String concat | `\|\|` | `+` | `\|\|` | `CONCAT()` |
| Current date | `CURRENT_DATE` | `GETDATE()` | `CURRENT_DATE` | `CURDATE()` |
| Auto increment | `GENERATED ALWAYS AS IDENTITY` | `IDENTITY(1,1)` | `SERIAL` | `AUTO_INCREMENT` |
| If/else inline | `CASE WHEN` | `CASE WHEN` / `IIF()` | `CASE WHEN` | `CASE WHEN` |

---

## Core ANSI SQL Features

### CASE WHEN
```sql
SELECT
    order_id,
    CASE
        WHEN amount > 1000 THEN 'High'
        WHEN amount > 500  THEN 'Medium'
        ELSE 'Low'
    END AS order_tier
FROM orders;
```

### CTEs (Common Table Expressions)
```sql
WITH monthly_sales AS (
    SELECT
        DATE_TRUNC('month', order_date) AS month,
        SUM(amount) AS total
    FROM orders
    GROUP BY 1
)
SELECT *
FROM monthly_sales
ORDER BY month DESC;
```

### Window Functions
```sql
SELECT
    employee_id,
    department,
    salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees;
```

### FETCH FIRST (ANSI standard for row limiting)
```sql
SELECT *
FROM orders
ORDER BY created_at DESC
FETCH FIRST 10 ROWS ONLY;
```

### NULLIF and COALESCE
```sql
-- COALESCE returns the first non-null value
SELECT COALESCE(phone, email, 'No contact') AS contact
FROM customers;

-- NULLIF returns null if both values are equal
SELECT NULLIF(status, 'unknown') AS status
FROM orders;
```

---

## Best Practices

- **Write ANSI SQL first** — use vendor-specific syntax only when necessary for performance or features unavailable in the standard
- **Use `CASE WHEN`** instead of vendor-specific if/else functions like `IIF()` or `DECODE()`
- **Use `COALESCE`** instead of `ISNULL()` or `NVL()`
- **Use standard JOINs** — avoid old comma-separated join syntax (`FROM a, b WHERE a.id = b.id`)
- **Use `CURRENT_DATE` and `CURRENT_TIMESTAMP`** instead of database-specific date functions when possible
- **Qualify all column names** with table aliases in multi-table queries to avoid ambiguity
- **Use `FETCH FIRST`** for row limiting when writing code that needs to run on multiple platforms

!!! tip
    When you join a new project or company, always check which database they use and what version. Features like window functions, CTEs, and JSON support depend on the database version, not just the platform.
