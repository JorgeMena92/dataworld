---
title: Functions
description: User-defined functions in SQL — scalar, table-valued, and aggregate
tags: [sql, database-objects, functions]
---

# Functions

A function is a reusable block of logic that accepts input parameters, performs a computation, and returns a result. Functions can be called from `SELECT`, `WHERE`, `JOIN`, and other SQL clauses — making them a powerful tool for encapsulating business logic close to the data.

---

## Functions vs Stored Procedures

| | Function | Stored Procedure |
|---|---|---|
| Returns value | ✅ Always | Optional |
| Used in SELECT | ✅ | ❌ |
| Can modify data | Limited | ✅ |
| Transaction control | ❌ (usually) | ✅ |
| ANSI SQL standard | ✅ (basic) | ❌ |

---

## Types of Functions

| Type | Returns | Use case |
|---|---|---|
| Scalar | A single value | Calculations, formatting, lookups |
| Table-valued | A result set (table) | Parameterized queries, reusable datasets |
| Aggregate | A single value from a group | Custom aggregation logic |

---

## Scalar Functions

Return a single value. Called like any built-in function in a query.

```sql
-- PostgreSQL
CREATE OR REPLACE FUNCTION full_name(first VARCHAR, last VARCHAR)
RETURNS VARCHAR
LANGUAGE SQL
AS $$
    SELECT TRIM(first) || ' ' || TRIM(last);
$$;

-- Usage
SELECT full_name(first_name, last_name) AS name FROM customers;
```

```sql
-- Calculate age from a birth date
CREATE OR REPLACE FUNCTION calculate_age(birth_date DATE)
RETURNS INT
LANGUAGE SQL
AS $$
    SELECT EXTRACT(YEAR FROM AGE(CURRENT_DATE, birth_date))::INT;
$$;

-- Usage
SELECT customer_id, calculate_age(birth_date) AS age FROM customers;
```

---

## Table-Valued Functions

Return a result set that can be used like a table in `FROM` or `JOIN`.

```sql
-- PostgreSQL
CREATE OR REPLACE FUNCTION orders_by_customer(p_customer_id INT)
RETURNS TABLE (order_id INT, amount DECIMAL, status VARCHAR)
LANGUAGE SQL
AS $$
    SELECT order_id, amount, status
    FROM orders
    WHERE customer_id = p_customer_id;
$$;

-- Usage
SELECT * FROM orders_by_customer(42);

-- Join with it
SELECT c.first_name, o.amount
FROM customers c
JOIN orders_by_customer(c.customer_id) o ON TRUE;
```

---

## Functions with PL/pgSQL (PostgreSQL)

For more complex logic — conditionals, loops, and exception handling.

```sql
CREATE OR REPLACE FUNCTION classify_order(amount DECIMAL)
RETURNS VARCHAR
LANGUAGE plpgsql
AS $$
BEGIN
    IF amount >= 1000 THEN
        RETURN 'High';
    ELSIF amount >= 500 THEN
        RETURN 'Medium';
    ELSE
        RETURN 'Low';
    END IF;
END;
$$;

-- Usage
SELECT order_id, amount, classify_order(amount) AS tier FROM orders;
```

---

## Immutability — STABLE and IMMUTABLE

Marking a function with its volatility helps the query optimizer.

```sql
-- IMMUTABLE — always returns the same result for the same inputs (no DB access)
CREATE FUNCTION format_currency(amount DECIMAL)
RETURNS VARCHAR
LANGUAGE SQL
IMMUTABLE
AS $$
    SELECT '$' || TO_CHAR(amount, 'FM999,999,990.00');
$$;

-- STABLE — returns the same result within a single query (may read DB)
CREATE FUNCTION get_tax_rate(country CHAR(2))
RETURNS DECIMAL
LANGUAGE SQL
STABLE
AS $$
    SELECT rate FROM tax_rates WHERE country_code = country;
$$;
```

| Volatility | Behavior | Optimization |
|---|---|---|
| `VOLATILE` (default) | Can return different results each call | No caching |
| `STABLE` | Same result within a query | Can be optimized |
| `IMMUTABLE` | Same result always, no DB access | Fully cacheable, usable in indexes |

---

## Dropping Functions

```sql
DROP FUNCTION full_name(VARCHAR, VARCHAR);
DROP FUNCTION IF EXISTS full_name(VARCHAR, VARCHAR);
```

---

## Vendor Notes

Function syntax varies significantly — functions are not fully standardized across databases.

| Feature | PostgreSQL | SQL Server | MySQL |
|---|---|---|---|
| Scalar function | `CREATE FUNCTION ... RETURNS type` | `CREATE FUNCTION ... RETURNS type` | `CREATE FUNCTION ... RETURNS type` |
| Table-valued | `RETURNS TABLE (...)` | `RETURNS TABLE` / inline TVF | ❌ |
| Language | `SQL`, `plpgsql`, `python` | `T-SQL` | `SQL` |
| Replace existing | `CREATE OR REPLACE` | `ALTER FUNCTION` | `CREATE OR REPLACE` |

---

## Best Practices

- Keep functions focused — one responsibility per function
- Mark functions `IMMUTABLE` or `STABLE` when possible — the optimizer can use this to improve performance
- Avoid functions that query large tables row-by-row in a scalar context — they run once per row and can be slow
- Use table-valued functions instead of views when you need parameterized queries
- Name functions as verbs or verb phrases — `calculate_age`, `classify_order`, `format_currency`
- Document input parameters and return values — functions are harder to introspect than tables
