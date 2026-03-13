---
title: Filtering & Conditions
description: WHERE, CASE WHEN, NULL handling, and conditional logic in ANSI SQL
tags: [sql, dql, filtering]
---

# Filtering & Conditions

Filtering controls which rows a query returns. Conditional logic transforms values inline. Together they handle most of the business rules and data quality patterns you will encounter in real queries.

---

## WHERE Clause

The `WHERE` clause filters rows before any grouping or aggregation. Only rows that satisfy the condition are kept.

```sql
-- Comparison operators
SELECT * FROM orders WHERE amount > 100;
SELECT * FROM orders WHERE status = 'completed';
SELECT * FROM orders WHERE created_at >= '2024-01-01';

-- Multiple conditions
SELECT * FROM orders
WHERE status = 'completed'
  AND amount > 500;

SELECT * FROM orders
WHERE status = 'pending'
   OR status = 'processing';
```

### IN and NOT IN

```sql
-- Match against a list
SELECT * FROM orders
WHERE country IN ('Peru', 'Chile', 'Argentina');

-- Exclude a list
SELECT * FROM orders
WHERE status NOT IN ('cancelled', 'refunded');
```

!!! warning
    Avoid `NOT IN` when the list comes from a subquery that might return `NULL` — it causes the entire condition to return no rows. Use `NOT EXISTS` instead.

### BETWEEN

```sql
-- Inclusive range
SELECT * FROM orders
WHERE amount BETWEEN 100 AND 500;

SELECT * FROM orders
WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';
```

### LIKE and Pattern Matching

```sql
-- Starts with
SELECT * FROM customers WHERE email LIKE 'jorge%';

-- Ends with
SELECT * FROM customers WHERE email LIKE '%@gmail.com';

-- Contains
SELECT * FROM customers WHERE first_name LIKE '%or%';

-- Single character wildcard
SELECT * FROM customers WHERE code LIKE 'A_C';
```

!!! tip
    Leading wildcards (`LIKE '%value'`) cannot use indexes and will cause full table scans on large tables. Avoid them in performance-critical queries.

---

## CASE WHEN

`CASE WHEN` adds conditional logic directly inside a query — for bucketing, labeling, flags, and transformations.

### Searched CASE

```sql
SELECT
    order_id,
    amount,
    CASE
        WHEN amount >= 1000 THEN 'High'
        WHEN amount >= 500  THEN 'Medium'
        WHEN amount >= 100  THEN 'Low'
        ELSE 'Micro'
    END AS order_tier
FROM orders;
```

### Simple CASE

```sql
SELECT
    order_id,
    CASE status
        WHEN 'completed'  THEN 'Done'
        WHEN 'pending'    THEN 'Waiting'
        WHEN 'cancelled'  THEN 'Cancelled'
        ELSE 'Unknown'
    END AS status_label
FROM orders;
```

### CASE inside Aggregations

```sql
-- Count by category in a single row
SELECT
    COUNT(*)                                            AS total,
    COUNT(CASE WHEN status = 'completed'  THEN 1 END)  AS completed,
    COUNT(CASE WHEN status = 'cancelled'  THEN 1 END)  AS cancelled,
    SUM(CASE WHEN status = 'completed' THEN amount ELSE 0 END) AS completed_revenue
FROM orders;
```

!!! tip
    `CASE WHEN` is pure ANSI SQL and works identically across all major databases. Prefer it over vendor-specific functions like `IIF()` (SQL Server) or `IF()` (MySQL).

---

## NULL Handling

`NULL` means the value is unknown or absent — not zero, not empty string. It behaves differently from any other value.

```sql
-- Always use IS NULL / IS NOT NULL — never = NULL
SELECT * FROM customers WHERE phone IS NULL;
SELECT * FROM customers WHERE phone IS NOT NULL;

-- NULL comparisons always return NULL (not true or false)
SELECT NULL = NULL;   -- NULL, not TRUE
SELECT NULL != NULL;  -- NULL, not TRUE
```

### COALESCE

Returns the first non-null value in the list. ANSI SQL standard.

```sql
-- Replace NULL with a default
SELECT
    first_name,
    COALESCE(phone, email, 'No contact') AS contact
FROM customers;

-- Safe division — avoid divide by zero via NULL
SELECT
    order_id,
    revenue / NULLIF(units, 0) AS revenue_per_unit
FROM orders;
```

### NULLIF

Returns `NULL` if both arguments are equal, otherwise returns the first argument.

```sql
-- Treat 'unknown' as NULL
SELECT NULLIF(status, 'unknown') AS status FROM orders;

-- Avoid division by zero
SELECT total / NULLIF(quantity, 0) AS unit_price FROM order_items;
```

### NULL in Aggregations

```sql
-- Aggregate functions ignore NULLs (except COUNT(*))
SELECT AVG(rating) FROM reviews;           -- ignores NULL ratings
SELECT COUNT(*) FROM reviews;              -- counts all rows
SELECT COUNT(rating) FROM reviews;         -- counts only non-NULL ratings

-- Count how many NULLs exist
SELECT COUNT(*) - COUNT(rating) AS null_count FROM reviews;
```

!!! tip
    `COALESCE` and `NULLIF` are ANSI SQL and portable across all databases. Avoid `ISNULL()` (SQL Server) and `NVL()` (Oracle) when writing cross-platform queries.

---

## Vendor Notes

| Feature | ANSI SQL | SQL Server | PostgreSQL | MySQL |
|---|---|---|---|---|
| Conditional inline | `CASE WHEN` | `CASE WHEN` / `IIF()` | `CASE WHEN` | `CASE WHEN` / `IF()` |
| First non-null | `COALESCE` | `COALESCE` / `ISNULL()` | `COALESCE` | `COALESCE` / `IFNULL()` |
| Pattern matching | `LIKE` | `LIKE` | `LIKE` / `ILIKE` | `LIKE` |
| Null-safe equality | — | — | `IS NOT DISTINCT FROM` | `<=>` |

---

## Best Practices

- Use `IS NULL` / `IS NOT NULL` — never `= NULL`
- Use `COALESCE` and `NULLIF` over vendor-specific null functions
- Use `CASE WHEN` over vendor-specific conditional functions for portability
- Filter as early as possible — reduce rows before joining or aggregating
- Be explicit about `NULL` behavior in every filter and aggregation
