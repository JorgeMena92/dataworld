---
title: Set Operations
description: UNION, INTERSECT, and EXCEPT for combining and comparing result sets
tags: [sql, dql, set-operations]
---

# Set Operations

Set operations combine the results of two or more `SELECT` queries into a single result set. They work on entire result sets rather than joining row by row — making them ideal for comparing datasets, merging sources, and finding differences.

---

## Rules

All set operations require:

- The same number of columns in each `SELECT`
- Compatible data types in each column position
- Column names come from the **first** query

```sql
-- Valid
SELECT first_name, email FROM customers
UNION
SELECT first_name, email FROM prospects;

-- Invalid — different number of columns
SELECT first_name, email FROM customers
UNION
SELECT first_name FROM prospects;
```

---

## UNION and UNION ALL

Combines rows from two queries vertically.

```sql
-- UNION — removes duplicates (slower)
SELECT city FROM customers
UNION
SELECT city FROM suppliers;

-- UNION ALL — keeps all rows including duplicates (faster)
SELECT city FROM customers
UNION ALL
SELECT city FROM suppliers;
```

| | `UNION` | `UNION ALL` |
|---|---|---|
| Removes duplicates | ✅ | ❌ |
| Performance | Slower (sorts to deduplicate) | Faster |
| Use when | You need distinct results | You want all rows |

!!! tip
    Default to `UNION ALL`. Only use `UNION` when you specifically need deduplication — the sort step it adds is expensive on large datasets.

### Practical Example — Merging Sources

```sql
-- Combine active customers from two regional databases
SELECT customer_id, first_name, 'EMEA' AS region
FROM customers_emea
WHERE status = 'active'

UNION ALL

SELECT customer_id, first_name, 'LATAM' AS region
FROM customers_latam
WHERE status = 'active';
```

---

## INTERSECT

Returns only rows that appear in **both** result sets.

```sql
-- Users who have both logged in and made a purchase
SELECT user_id FROM logins
INTERSECT
SELECT user_id FROM purchases;
```

### Practical Example — Common Records

```sql
-- Products sold in both 2023 and 2024
SELECT product_id FROM sales WHERE EXTRACT(YEAR FROM sale_date) = 2023
INTERSECT
SELECT product_id FROM sales WHERE EXTRACT(YEAR FROM sale_date) = 2024;
```

!!! warning
    `INTERSECT` is not supported in MySQL before version 8.0. If targeting older MySQL versions, use an `INNER JOIN` or `IN` subquery as a workaround.

---

## EXCEPT

Returns rows from the first query that do **not** appear in the second.

```sql
-- Users who logged in but did NOT make a purchase
SELECT user_id FROM logins
EXCEPT
SELECT user_id FROM purchases;
```

### Practical Example — Finding Gaps

```sql
-- Customers who registered but never ordered
SELECT customer_id FROM customers
EXCEPT
SELECT DISTINCT customer_id FROM orders;

-- Products in the catalog but never sold
SELECT product_id FROM products
EXCEPT
SELECT DISTINCT product_id FROM order_items;
```

---

## Sorting Set Operation Results

`ORDER BY` applies to the entire combined result — place it at the end, after all set operations.

```sql
SELECT customer_id, first_name, 'customer' AS source FROM customers
UNION ALL
SELECT prospect_id, first_name, 'prospect' AS source FROM prospects
ORDER BY first_name;
```

---

## Set Operations vs JOINs

Both can answer "what is in common" or "what is missing" — choose based on what you need.

```sql
-- INTERSECT — clean and readable for set comparison
SELECT user_id FROM logins
INTERSECT
SELECT user_id FROM purchases;

-- Equivalent JOIN
SELECT DISTINCT l.user_id
FROM logins l
JOIN purchases p ON l.user_id = p.user_id;

-- EXCEPT — clean for finding missing rows
SELECT customer_id FROM customers
EXCEPT
SELECT customer_id FROM orders;

-- Equivalent LEFT JOIN
SELECT c.customer_id
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.customer_id IS NULL;
```

Use set operations when you are comparing whole result sets. Use JOINs when you need columns from both tables in the result.

---

## Operator Precedence

When chaining multiple set operations, `INTERSECT` has higher precedence than `UNION` and `EXCEPT` in ANSI SQL. This can produce unexpected results if parentheses are not used explicitly.

```sql
-- Without parentheses — INTERSECT binds first
-- Evaluated as: A UNION (B INTERSECT C)
SELECT user_id FROM table_a
UNION
SELECT user_id FROM table_b
INTERSECT
SELECT user_id FROM table_c;

-- With parentheses — explicit and unambiguous
-- Evaluated as: (A UNION B) INTERSECT C
SELECT user_id FROM (
    SELECT user_id FROM table_a
    UNION
    SELECT user_id FROM table_b
) combined
INTERSECT
SELECT user_id FROM table_c;
```

!!! tip
    Always use parentheses or CTEs when combining more than two set operations. Relying on implicit precedence makes queries harder to read and easier to get wrong.

---

## Vendor Notes

| Operation | ANSI SQL | SQL Server | PostgreSQL | MySQL | Oracle |
|---|---|---|---|---|---|
| Combine all rows | `UNION ALL` | ✅ | ✅ | ✅ | ✅ |
| Combine distinct | `UNION` | ✅ | ✅ | ✅ | ✅ |
| Common rows | `INTERSECT` | ✅ | ✅ | ❌ (8.0+: ✅) | ✅ |
| Difference | `EXCEPT` | ✅ | ✅ | ❌ (8.0+: ✅) | `MINUS` |

---

## Best Practices

- Use `UNION ALL` by default — only switch to `UNION` when deduplication is required
- Use `INTERSECT` and `EXCEPT` for clean dataset comparisons instead of forcing everything into joins
- Always place `ORDER BY` at the end of the full set operation, not inside individual queries
- Use parentheses or CTEs when chaining more than two set operations — `INTERSECT` has higher precedence than `UNION` and `EXCEPT`
- Be aware of `INTERSECT` and `EXCEPT` limitations in MySQL before 8.0 and Oracle's `MINUS` naming
- When portability matters, rewrite `INTERSECT` and `EXCEPT` as `JOIN` or `LEFT JOIN + WHERE NULL` patterns