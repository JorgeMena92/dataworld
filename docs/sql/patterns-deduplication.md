---
title: Deduplication
description: Identifying and removing duplicate rows with SQL
tags: [sql, patterns, deduplication, window-functions]
---

# Deduplication

Deduplication removes redundant rows from a dataset — keeping only one representative row per group. It is one of the most common data quality operations in ETL pipelines and data warehouses. For the broader context of why deduplication matters, see [Data Integrity](data-integrity.md).

---

## Types of Duplicates

| Type | Description |
|---|---|
| Exact duplicates | Every column is identical |
| Key duplicates | Same business key, different attribute values |
| Near duplicates | Similar but not identical — require fuzzy matching |

!!! note
    Near duplicates (fuzzy matching) are beyond standard SQL — they require string similarity functions like Levenshtein distance, which are available as extensions in PostgreSQL (`pg_trgm`) or as user-defined functions in other platforms. This page covers exact and key duplicates only.

---

## Exact Duplicates — DISTINCT

Use `DISTINCT` to remove rows where every selected column is identical.

```sql
-- Remove exact duplicates
SELECT DISTINCT customer_id, email, country
FROM customers;

-- Count duplicates
SELECT customer_id, email, COUNT(*) AS occurrences
FROM customers
GROUP BY customer_id, email
HAVING COUNT(*) > 1;
```

---

## Key Duplicates — ROW_NUMBER

The most common deduplication pattern — keep one row per business key based on a defined rule (latest, highest, first).

### Keep the Latest Record

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

### Keep the First Record

```sql
SELECT *
FROM (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY created_at ASC
        ) AS rn
    FROM customers
) t
WHERE rn = 1;
```

### Keep the Record with Highest Value

```sql
SELECT *
FROM (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY product_id
            ORDER BY sale_price DESC
        ) AS rn
    FROM product_prices
) t
WHERE rn = 1;
```

---

## Deduplication with CTE

Cleaner syntax using a CTE — easier to read and debug.

```sql
WITH ranked AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY updated_at DESC
        ) AS rn
    FROM customers
)
SELECT *
FROM ranked
WHERE rn = 1;
```

---

## Delete Duplicates from a Table

Remove duplicate rows from a physical table using a primary key to identify which rows to keep.

```sql
-- ANSI SQL — delete duplicates using ROW_NUMBER and the primary key
WITH ranked AS (
    SELECT
        customer_id,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY updated_at DESC
        ) AS rn
    FROM customers
)
DELETE FROM customers
WHERE customer_id IN (
    SELECT customer_id FROM ranked WHERE rn > 1
);
```

!!! note
    Avoid using platform-specific row identifiers like `ctid` (PostgreSQL) or `ROWID` (Oracle) for deduplication — they are internal and not portable. Use the table's primary key or a surrogate key instead to identify rows to delete.

---

## Deduplication on Load — INSERT with NOT EXISTS

Prevent duplicates from being inserted in the first place.

```sql
-- Only insert rows that don't already exist
INSERT INTO customers (customer_id, email, country)
SELECT source.customer_id, source.email, source.country
FROM staging_customers source
WHERE NOT EXISTS (
    SELECT 1 FROM customers target
    WHERE target.customer_id = source.customer_id
);
```

!!! note
    For a more explicit upsert pattern that handles both inserts and updates, see [MERGE / UPSERT](merge.md).

---

## ROW_NUMBER vs RANK vs DISTINCT

| Approach | Use when |
|---|---|
| `DISTINCT` | All columns are identical |
| `ROW_NUMBER()` | Keep exactly one row per key, with a defined tiebreaker |
| `RANK()` | Keep all rows tied for the top position (may return multiple) |
| `GROUP BY + MAX/MIN` | You only need aggregated values, not the full row |

```sql
-- GROUP BY approach — when you only need aggregated columns
SELECT
    customer_id,
    MAX(updated_at) AS last_updated,
    MAX(email)      AS latest_email
FROM customers
GROUP BY customer_id;
```

---

## Deduplication in Pipelines

In data pipelines, deduplication typically happens at one of three points:

```
Source → [Staging] → [Dedup] → [Target]
```

1. **On ingest** — prevent duplicates entering staging with `NOT EXISTS` or `MERGE`
2. **In staging** — deduplicate in staging before loading to the target
3. **On read** — use a view or CTE with `ROW_NUMBER()` to present deduplicated data

The further upstream you deduplicate, the less work downstream queries have to do.

---

## Best Practices

- Always define a clear rule for which row to keep — latest, first, highest value
- Use `ROW_NUMBER()` rather than `RANK()` for deduplication — it always returns exactly one row per group
- Use the table's primary key to identify rows to delete — avoid internal row identifiers like `ctid`
- Deduplicate as early in the pipeline as possible
- Log how many duplicates were removed — unexpected counts indicate upstream data quality issues
- Add a `UNIQUE` constraint on the business key after deduplication to prevent future duplicates at the database level