---
title: Scalar Functions
description: ANSI SQL scalar functions for strings, numbers, dates, and null handling
tags: [sql, dql, functions, scalar]
---

# Scalar Functions

Scalar functions operate on a single value and return a single value. They transform, format, or extract data inline — inside `SELECT`, `WHERE`, `ORDER BY`, or any expression context.

This page covers the ANSI SQL standard functions. Where vendor-specific alternatives exist, they are noted at the end of each section and in the comparison table.

---

## String Functions

### Case Conversion

```sql
SELECT UPPER('hello');          -- 'HELLO'
SELECT LOWER('HELLO');          -- 'hello'
```

### Trimming Whitespace

```sql
-- Remove leading and trailing spaces
SELECT TRIM('  hello  ');           -- 'hello'

-- Remove only leading spaces
SELECT TRIM(LEADING ' ' FROM '  hello  ');   -- 'hello  '

-- Remove only trailing spaces
SELECT TRIM(TRAILING ' ' FROM '  hello  ');  -- '  hello'
```

### Length

```sql
SELECT CHAR_LENGTH('hello');    -- 5   (ANSI standard)
SELECT CHARACTER_LENGTH('hello'); -- 5  (same, longer form)
```

### Substring Extraction

```sql
-- SUBSTRING(string FROM start FOR length) — ANSI syntax
SELECT SUBSTRING('hello world' FROM 1 FOR 5);  -- 'hello'
SELECT SUBSTRING('hello world' FROM 7 FOR 5);  -- 'world'

-- Omitting FOR returns everything from the start position onward
SELECT SUBSTRING('hello world' FROM 7);        -- 'world'

-- Practical use: extract year from a string date
SELECT SUBSTRING(order_date FROM 1 FOR 4) AS order_year
FROM orders;
```

### Concatenation

```sql
-- ANSI SQL concatenation operator
SELECT 'Hello' || ' ' || 'World';   -- 'Hello World'

-- CONCAT function (widely supported)
SELECT CONCAT(first_name, ' ', last_name) AS full_name
FROM customers;
```

!!! warning
    The `||` operator is ANSI SQL but SQL Server uses `+` instead. Use `CONCAT()` for maximum portability — it is supported across all major platforms.

### String Position

```sql
-- Find the position of a substring
SELECT POSITION('world' IN 'hello world');  -- 7
```

### Replace

```sql
SELECT REPLACE('hello world', 'world', 'SQL');  -- 'hello SQL'
```

### Padding

```sql
-- Left-pad to a fixed width
SELECT LPAD('42', 5, '0');     -- '00042'

-- Right-pad to a fixed width
SELECT RPAD('hello', 8, '.');  -- 'hello...'
```

!!! note
    `LPAD` and `RPAD` are widely supported but not strictly ANSI SQL. They are available in PostgreSQL, MySQL, Oracle, and Snowflake. SQL Server requires a workaround using `RIGHT` and `REPLICATE`.

---

## Numeric Functions

### Rounding

```sql
-- Round to n decimal places
SELECT ROUND(3.14159, 2);    -- 3.14
SELECT ROUND(3.14159, 0);    -- 3.0
SELECT ROUND(3.75, 0);       -- 4.0

-- Negative precision rounds to the left of the decimal point
SELECT ROUND(1234.56, -2);   -- 1200  (round to nearest hundred)
SELECT ROUND(1234.56, -1);   -- 1230  (round to nearest ten)

-- Round down to nearest integer
SELECT FLOOR(3.9);           -- 3
SELECT FLOOR(-3.1);          -- -4

-- Round up to nearest integer
SELECT CEILING(3.1);         -- 4
SELECT CEILING(-3.9);        -- -3
```

### Absolute Value and Sign

```sql
SELECT ABS(-42);             -- 42
SELECT ABS(42);              -- 42

SELECT SIGN(-5);             -- -1
SELECT SIGN(0);              -- 0
SELECT SIGN(5);              -- 1
```

### Modulo

```sql
-- Remainder after division (ANSI)
SELECT MOD(10, 3);           -- 1
SELECT MOD(15, 5);           -- 0
```

### Power and Square Root

```sql
SELECT POWER(2, 10);         -- 1024
SELECT SQRT(144);            -- 12
```

---

## Date and Time Functions

### Current Date and Time

```sql
-- ANSI SQL standard — portable across all platforms
SELECT CURRENT_DATE;         -- current date, e.g. 2024-03-14
SELECT CURRENT_TIME;         -- current time
SELECT CURRENT_TIMESTAMP;    -- current date and time
```

!!! tip
    Always use `CURRENT_DATE` and `CURRENT_TIMESTAMP` over vendor functions like `GETDATE()` (SQL Server) or `NOW()` (MySQL/PostgreSQL) when writing portable SQL.

### Extracting Date Parts

```sql
-- EXTRACT is ANSI SQL
SELECT EXTRACT(YEAR  FROM CURRENT_DATE);
SELECT EXTRACT(MONTH FROM order_date) FROM orders;
SELECT EXTRACT(DAY   FROM order_date) FROM orders;
SELECT EXTRACT(HOUR  FROM CURRENT_TIMESTAMP);
```

Common ANSI-supported date parts: `YEAR`, `MONTH`, `DAY`, `HOUR`, `MINUTE`, `SECOND`.

!!! warning
    `EXTRACT(QUARTER FROM ...)` is widely used but not part of ANSI SQL. It works in PostgreSQL, MySQL, and Snowflake but not in SQL Server. For portable queries, derive the quarter from the month:
    ```sql
    CASE
        WHEN EXTRACT(MONTH FROM order_date) <= 3  THEN 1
        WHEN EXTRACT(MONTH FROM order_date) <= 6  THEN 2
        WHEN EXTRACT(MONTH FROM order_date) <= 9  THEN 3
        ELSE 4
    END AS quarter
    ```

### Date Arithmetic with INTERVAL

```sql
-- Add an interval to a date
SELECT CURRENT_DATE + INTERVAL '7' DAY;
SELECT CURRENT_DATE + INTERVAL '3' MONTH;
SELECT CURRENT_DATE - INTERVAL '1' YEAR;

-- Find orders placed in the last 30 days
SELECT * FROM orders
WHERE order_date >= CURRENT_DATE - INTERVAL '30' DAY;
```

!!! warning
    `INTERVAL` syntax varies across platforms. The ANSI form `INTERVAL '7' DAY` is the safest. SQL Server uses `DATEADD(day, 7, CURRENT_TIMESTAMP)`. Always test date arithmetic when migrating between databases.

---

## NULL-Handling Functions

These functions are covered in detail on the [Filtering & Conditions](filtering-conditions.md) page. Summary for quick reference:

### COALESCE

Returns the first non-NULL value in the list. ANSI SQL standard.

```sql
SELECT COALESCE(phone, email, 'No contact') AS contact
FROM customers;
```

### NULLIF

Returns `NULL` if both arguments are equal. Returns the first argument otherwise.

```sql
-- Prevent division by zero
SELECT revenue / NULLIF(units, 0) AS revenue_per_unit
FROM orders;
```

---

## Combining Functions

Functions can be nested and combined inside any expression.

```sql
-- Normalize customer name for comparison
SELECT *
FROM customers
WHERE UPPER(TRIM(last_name)) = 'MENA';

-- Build a formatted label — capitalize first letter, lowercase the rest
-- SUBSTRING(name FROM 1 FOR 1) extracts the first character
-- SUBSTRING(name FROM 2) extracts everything from the second character onward
SELECT
    CONCAT(
        UPPER(SUBSTRING(first_name FROM 1 FOR 1)),
        LOWER(SUBSTRING(first_name FROM 2))
    ) AS formatted_name
FROM customers;

-- Round revenue to the nearest 100
SELECT
    order_id,
    ROUND(amount, -2) AS amount_bucket   -- negative precision rounds left of decimal
FROM orders;
```

---

## Vendor Comparison

| Function | ANSI SQL | SQL Server | PostgreSQL | MySQL / MariaDB |
|---|---|---|---|---|
| String length | `CHAR_LENGTH` | `LEN` | `CHAR_LENGTH` / `LENGTH` | `CHAR_LENGTH` / `LENGTH` |
| Substring | `SUBSTRING(s FROM n FOR len)` | `SUBSTRING(s, n, len)` | Both forms | `SUBSTRING(s, n, len)` |
| Concatenate | `\|\|` or `CONCAT()` | `+` or `CONCAT()` | Both | `CONCAT()` |
| String position | `POSITION(x IN s)` | `CHARINDEX(x, s)` | `POSITION` / `STRPOS` | `POSITION` / `LOCATE` |
| Current date | `CURRENT_DATE` | `CAST(GETDATE() AS DATE)` | `CURRENT_DATE` | `CURRENT_DATE` |
| Current timestamp | `CURRENT_TIMESTAMP` | `GETDATE()` | `NOW()` / `CURRENT_TIMESTAMP` | `NOW()` |
| Date parts | `EXTRACT(YEAR FROM d)` | `YEAR(d)` / `DATEPART` | `EXTRACT` / `DATE_PART` | `YEAR(d)` / `EXTRACT` |
| Date add | `d + INTERVAL '7' DAY` | `DATEADD(day, 7, d)` | `d + INTERVAL '7 days'` | `DATE_ADD(d, INTERVAL 7 DAY)` |
| Round | `ROUND` | `ROUND` | `ROUND` | `ROUND` |
| Floor | `FLOOR` | `FLOOR` | `FLOOR` | `FLOOR` |
| Ceiling | `CEILING` | `CEILING` | `CEIL` / `CEILING` | `CEIL` / `CEILING` |
| Modulo | `MOD(a, b)` | `a % b` | `MOD` / `%` | `MOD` / `%` |

---

## Best Practices

- Use `CURRENT_DATE` and `CURRENT_TIMESTAMP` over vendor-specific date functions
- Use `EXTRACT()` for date parts instead of `YEAR()`, `MONTH()`, `DAY()` vendor functions
- Use `CONCAT()` over the `||` operator for string concatenation — SQL Server does not support `||`
- Use `CHAR_LENGTH()` over `LENGTH()` for character counts — `LENGTH()` returns bytes in some platforms
- Use `COALESCE()` and `NULLIF()` over `ISNULL()`, `NVL()`, or `IFNULL()` — they are ANSI standard
- Avoid nesting more than 2–3 functions in a single expression — use a CTE to break complex transformations into readable steps