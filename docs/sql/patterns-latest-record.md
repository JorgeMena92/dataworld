---
title: Latest Record per Group
description: Retrieving the most recent row per entity with SQL window functions
tags: [sql, patterns, latest-record, window-functions]
---

# Latest Record per Group

The latest record per group pattern retrieves the most recent version of each entity from a table that contains multiple rows per key — such as a history table, a changelog, or a staging table with repeated loads.

This is one of the most frequently used patterns in data engineering.

---

## The Problem

A table contains multiple rows per `customer_id` — one for each update. You need the most recent row per customer.

```
customer_id   email                  updated_at
1             jorge@old.com          2024-01-01
1             jorge@new.com          2024-06-15   ← keep this
2             ana@example.com        2024-03-10   ← keep this
3             luis@v1.com            2023-12-01
3             luis@v2.com            2024-02-14
3             luis@v3.com            2024-07-01   ← keep this
```

---

## ROW_NUMBER — The Standard Approach

```sql
SELECT *
FROM (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY updated_at DESC
        ) AS rn
    FROM customers
) t
WHERE rn = 1;
```

`ROW_NUMBER()` assigns 1 to the most recent row per `customer_id`. Filtering `WHERE rn = 1` returns exactly one row per group — even if there are ties.

---

## With a CTE

```sql
WITH latest AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY updated_at DESC
        ) AS rn
    FROM customers
)
SELECT customer_id, email, country, updated_at
FROM latest
WHERE rn = 1;
```

---

## Handling Ties

When two rows have the same `updated_at`, add a tiebreaker to make the result deterministic.

```sql
WITH latest AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY updated_at DESC, customer_id ASC  -- tiebreaker
        ) AS rn
    FROM customers
)
SELECT *
FROM latest
WHERE rn = 1;
```

---

## Latest Record Using DISTINCT ON (PostgreSQL)

PostgreSQL offers a cleaner syntax for this pattern.

```sql
SELECT DISTINCT ON (customer_id)
    customer_id, email, country, updated_at
FROM customers
ORDER BY customer_id, updated_at DESC;
```

`DISTINCT ON` keeps the first row for each distinct value of the specified column — combined with `ORDER BY`, this returns the most recent row per customer.

!!! warning
    `DISTINCT ON` is PostgreSQL-specific and not ANSI SQL. Use `ROW_NUMBER()` for cross-platform queries.

---

## Latest Record with a Correlated Subquery

An alternative approach — less efficient on large tables but readable for small ones.

```sql
SELECT *
FROM customers c
WHERE updated_at = (
    SELECT MAX(updated_at)
    FROM customers
    WHERE customer_id = c.customer_id
);
```

!!! warning
    If two rows share the same `MAX(updated_at)` for a customer, this query returns both rows — not just one. Use `ROW_NUMBER()` when you need exactly one row per group regardless of ties.

!!! tip
    Prefer `ROW_NUMBER()` over the correlated subquery on large tables — the subquery executes once per row of the outer query.

---

## Latest Record per Group vs Deduplication

These patterns are related but serve different purposes:

| | Latest Record per Group | Deduplication |
|---|---|---|
| Goal | Get the current state of each entity | Remove redundant copies |
| Source | History table, changelog, repeated loads | Table with accidental duplicates |
| Rows per key | Expected — it's a design | Unexpected — it's a data quality issue |
| Method | `ROW_NUMBER() ... ORDER BY date DESC` | `ROW_NUMBER()` or `DISTINCT` |

!!! note
    For the full deduplication treatment including pipeline patterns and DELETE strategies, see [Deduplication](patterns-deduplication.md). For building dimension tables where each entity needs a current and historical record, see [Slowly Changing Dimensions](patterns-scd.md) — the latest record pattern is essentially reading SCD Type 1 current state.

---

## Practical Examples

### Latest Order per Customer

```sql
WITH latest_orders AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY order_date DESC
        ) AS rn
    FROM orders
)
SELECT customer_id, order_id, order_date, amount
FROM latest_orders
WHERE rn = 1;
```

### Latest Price per Product

```sql
WITH latest_prices AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY product_id
            ORDER BY effective_date DESC
        ) AS rn
    FROM product_prices
)
SELECT product_id, price, effective_date
FROM latest_prices
WHERE rn = 1;
```

### Latest Status per Order

```sql
WITH latest_status AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY order_id
            ORDER BY changed_at DESC
        ) AS rn
    FROM order_status_history
)
SELECT order_id, status, changed_at
FROM latest_status
WHERE rn = 1;
```

---

## Best Practices

- Use `ROW_NUMBER()` for cross-platform compatibility — not `DISTINCT ON`
- Always add a tiebreaker to `ORDER BY` to make results deterministic when timestamps can repeat
- Materialize the result in a view or table if the same latest-record query is used frequently
- Index the partition key and the order column for performance on large tables