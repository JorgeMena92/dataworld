---
title: Data Types & Casting
description: ANSI SQL data types, CAST, and cross-platform type handling
tags: [sql, dql, data-types, casting]
---

# Data Types & Casting

Data types define what kind of value a column can store. Casting converts a value from one type to another. Understanding both is essential when migrating queries across platforms — type names and implicit conversion rules vary significantly between databases.

---

## ANSI SQL Data Types

These are the standard types defined by ANSI SQL, supported across most major databases.

### Numeric

| Type | Description | Example |
|---|---|---|
| `INTEGER` / `INT` | Whole numbers | `42` |
| `BIGINT` | Large whole numbers | `9000000000` |
| `SMALLINT` | Small whole numbers | `100` |
| `DECIMAL(p, s)` | Exact decimal — p digits total, s after decimal | `DECIMAL(10, 2)` |
| `NUMERIC(p, s)` | Identical to `DECIMAL` in ANSI SQL | `NUMERIC(8, 4)` |
| `FLOAT` | Approximate decimal (double precision) | `3.14159` |
| `REAL` | Approximate decimal (single precision) | `3.14` |

!!! tip
    Use `DECIMAL` or `NUMERIC` for financial and monetary data — `FLOAT` and `REAL` are approximate and will introduce rounding errors in calculations.

### Character

| Type | Description |
|---|---|
| `CHAR(n)` | Fixed-length string, padded with spaces |
| `VARCHAR(n)` | Variable-length string up to n characters |

```sql
-- CHAR pads to fixed length — 'Peru' stored as 'Peru      ' in CHAR(10)
-- VARCHAR stores exactly what you insert
CREATE TABLE example (
    country_code CHAR(3),
    description  VARCHAR(255)
);
```

!!! tip
    `TEXT` is widely used across platforms but is not part of core ANSI SQL. Use `VARCHAR(n)` with a large value (e.g. `VARCHAR(4000)`) for portable queries. Reserve `TEXT` for platform-specific work where unlimited length is required.

### Date and Time

| Type | Description | Example |
|---|---|---|
| `DATE` | Date only | `2024-01-15` |
| `TIME` | Time only | `14:30:00` |
| `TIMESTAMP` | Date and time | `2024-01-15 14:30:00` |
| `INTERVAL` | Duration between two time points | `INTERVAL '7' DAY` |

```sql
-- INTERVAL used in date arithmetic
SELECT CURRENT_DATE + INTERVAL '30' DAY;   -- 30 days from today
SELECT CURRENT_DATE - INTERVAL '1' MONTH;  -- one month ago
```

!!! note
    Date arithmetic with `INTERVAL` and date functions like `EXTRACT` are covered in [Scalar Functions](scalar-functions.md).

### Boolean

```sql
-- ANSI SQL boolean
TRUE, FALSE, UNKNOWN

-- Stored as BOOLEAN type
CREATE TABLE flags (is_active BOOLEAN);
INSERT INTO flags VALUES (TRUE);
```

!!! warning
    `BOOLEAN` support varies across platforms. SQL Server uses `BIT` (0 or 1) and MySQL uses `TINYINT(1)`. When writing cross-platform SQL, avoid assuming `BOOLEAN` exists — consider using `CHAR(1)` with values `'Y'`/`'N'` or a small integer instead.

---

## CAST — Type Conversion

`CAST` is the ANSI SQL standard for converting values between types.

```sql
-- Basic syntax
CAST(value AS target_type)

-- String to integer
SELECT CAST('42' AS INTEGER);

-- Integer to decimal
SELECT CAST(price AS DECIMAL(10, 2)) FROM products;

-- String to date
SELECT CAST('2024-01-15' AS DATE);

-- Number to string
SELECT CAST(order_id AS VARCHAR(20)) FROM orders;
```

### CAST in Practice

```sql
-- Safe division — cast to avoid integer truncation
SELECT
    total_revenue,
    total_orders,
    CAST(total_revenue AS DECIMAL(12, 2)) / NULLIF(total_orders, 0) AS avg_order_value
FROM summary;

-- Concatenate a number with a string
SELECT 'Order #' || CAST(order_id AS VARCHAR(20)) AS label
FROM orders;

-- Extract year from a string date using ANSI EXTRACT
SELECT EXTRACT(YEAR FROM CAST('2024-06-15' AS DATE));
```

---

## Implicit vs Explicit Casting

Databases often convert types automatically — but implicit casting rules differ across platforms and can cause subtle bugs.

```sql
-- This may work in some databases (implicit cast)
WHERE order_id = '42'  -- string compared to integer

-- This is always safe (explicit cast)
WHERE order_id = CAST('42' AS INTEGER)
-- or
WHERE CAST(order_id AS VARCHAR(20)) = '42'
```

!!! tip
    Always cast explicitly when mixing types. Implicit casting works differently across databases and can silently produce wrong results or kill index usage.

---

## Date and Time Casting

Date handling is one of the biggest sources of cross-platform incompatibility.

```sql
-- ANSI SQL standard — works in PostgreSQL, Snowflake, BigQuery, SQL Server
SELECT CAST('2024-01-15' AS DATE);
SELECT CAST('2024-01-15 14:30:00' AS TIMESTAMP);
```

!!! note
    For extracting date parts and date arithmetic, use `EXTRACT()` and `INTERVAL` — both ANSI SQL and covered in [Scalar Functions](scalar-functions.md).

---

## Type Handling Across Platforms

| Concept | ANSI SQL | SQL Server | PostgreSQL | MySQL |
|---|---|---|---|---|
| Exact decimal | `DECIMAL` | `DECIMAL` / `MONEY` | `DECIMAL` / `NUMERIC` | `DECIMAL` |
| Large text | `VARCHAR(n)` | `VARCHAR(MAX)` | `TEXT` | `TEXT` |
| Boolean | `BOOLEAN` | `BIT` | `BOOLEAN` | `TINYINT(1)` |
| Auto increment | `GENERATED ALWAYS AS IDENTITY` | `IDENTITY(1,1)` | `SERIAL` / `IDENTITY` | `AUTO_INCREMENT` |
| Type conversion | `CAST()` | `CAST()` / `CONVERT()` | `CAST()` / `::` | `CAST()` / `CONVERT()` |
| Current timestamp | `CURRENT_TIMESTAMP` | `GETDATE()` | `NOW()` / `CURRENT_TIMESTAMP` | `NOW()` |

---

## Best Practices

- Use `DECIMAL(p, s)` over `FLOAT` for any monetary or financial values
- Use `CAST()` over vendor-specific functions like `CONVERT()` or PostgreSQL's `::` operator for portability
- Always cast explicitly when mixing types in comparisons or calculations
- Use `VARCHAR(n)` over `TEXT` for portable queries — `TEXT` is not core ANSI SQL
- Use `VARCHAR` over `CHAR` for variable-length strings — `CHAR` pads with spaces and can cause comparison issues
- Use `EXTRACT()` to get date parts rather than casting dates to integers — integer conversion of dates is not reliable across platforms
- Use `CURRENT_DATE` and `CURRENT_TIMESTAMP` instead of vendor functions like `GETDATE()` or `NOW()`