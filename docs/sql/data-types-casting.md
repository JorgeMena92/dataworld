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
| `TEXT` | Unlimited length text (not in all ANSI versions) |

```sql
-- CHAR pads to fixed length — 'Peru' stored as 'Peru      ' in CHAR(10)
-- VARCHAR stores exactly what you insert
CREATE TABLE example (
    country_code CHAR(3),
    description  VARCHAR(255)
);
```

### Date and Time

| Type | Description | Example |
|---|---|---|
| `DATE` | Date only | `2024-01-15` |
| `TIME` | Time only | `14:30:00` |
| `TIMESTAMP` | Date and time | `2024-01-15 14:30:00` |
| `INTERVAL` | Duration between two time points | `INTERVAL '7' DAY` |

### Boolean

```sql
-- ANSI SQL boolean
TRUE, FALSE, UNKNOWN

-- Stored as BOOLEAN type
CREATE TABLE flags (is_active BOOLEAN);
INSERT INTO flags VALUES (TRUE);
```

!!! warning
    SQL Server does not have a native `BOOLEAN` type — it uses `BIT` (0 or 1). When writing cross-platform SQL, avoid assuming `BOOLEAN` exists.

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
SELECT 'Order #' || CAST(order_id AS VARCHAR) AS label
FROM orders;

-- Extract year from a string date
SELECT CAST(CAST('2024-06-15' AS DATE) AS INTEGER);
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
WHERE CAST(order_id AS VARCHAR) = '42'
```

!!! tip
    Always cast explicitly when mixing types. Implicit casting works differently across databases and can silently produce wrong results or kill index usage.

---

## Date and Time Casting

Date handling is one of the biggest sources of cross-platform incompatibility.

```sql
-- ANSI SQL standard — works in PostgreSQL, Snowflake, BigQuery
SELECT CAST('2024-01-15' AS DATE);
SELECT CAST('2024-01-15 14:30:00' AS TIMESTAMP);

-- Extract parts using ANSI EXTRACT
SELECT EXTRACT(YEAR  FROM CURRENT_DATE);
SELECT EXTRACT(MONTH FROM CURRENT_TIMESTAMP);
SELECT EXTRACT(DAY   FROM order_date) FROM orders;
```

---

## Type Handling Across Platforms

| Concept | ANSI SQL | SQL Server | PostgreSQL | MySQL |
|---|---|---|---|---|
| Exact decimal | `DECIMAL` | `DECIMAL` / `MONEY` | `DECIMAL` / `NUMERIC` | `DECIMAL` |
| Large text | `VARCHAR` | `VARCHAR(MAX)` | `TEXT` | `TEXT` |
| Boolean | `BOOLEAN` | `BIT` | `BOOLEAN` | `TINYINT(1)` |
| Auto increment | `GENERATED ALWAYS AS IDENTITY` | `IDENTITY(1,1)` | `SERIAL` / `IDENTITY` | `AUTO_INCREMENT` |
| Type conversion | `CAST()` | `CAST()` / `CONVERT()` | `CAST()` / `::` | `CAST()` / `CONVERT()` |
| Current timestamp | `CURRENT_TIMESTAMP` | `GETDATE()` | `NOW()` / `CURRENT_TIMESTAMP` | `NOW()` |

---

## Best Practices

- Use `DECIMAL(p, s)` over `FLOAT` for any monetary or financial values
- Use `CAST()` over vendor-specific functions like `CONVERT()` or PostgreSQL's `::` operator for portability
- Always cast explicitly when mixing types in comparisons or calculations
- Use `VARCHAR` over `CHAR` for variable-length strings — `CHAR` pads with spaces and can cause comparison issues
- Use `CURRENT_DATE` and `CURRENT_TIMESTAMP` instead of vendor functions like `GETDATE()` or `NOW()`
- Use `EXTRACT()` for date parts instead of `YEAR()`, `MONTH()`, `DAY()` vendor functions
