---
title: INSERT
description: Adding rows to tables with INSERT in ANSI SQL
tags: [sql, dml, insert]
---

# INSERT

`INSERT` adds new rows to a table. It is the entry point for all data — whether loading from a source system, seeding reference data, or copying records between tables.

---

## Basic Syntax

```sql
INSERT INTO table_name (column1, column2, column3)
VALUES (value1, value2, value3);
```

Always specify the column list explicitly — it makes the statement resilient to schema changes and avoids inserting values into the wrong columns.

---

## Insert a Single Row

```sql
INSERT INTO customers (first_name, last_name, email, created_at)
VALUES ('Jorge', 'Mena', 'jorge@example.com', CURRENT_TIMESTAMP);
```

---

## Insert Multiple Rows

```sql
INSERT INTO customers (first_name, last_name, email)
VALUES
    ('Ana',  'García', 'ana@example.com'),
    ('Luis', 'Pérez',  'luis@example.com'),
    ('María','López',  'maria@example.com');
```

Multi-row inserts are more efficient than running individual `INSERT` statements in a loop — fewer round trips to the database.

---

## INSERT ... SELECT

Insert rows from the result of a query. The most common pattern in ETL and data pipelines.

```sql
-- Copy completed orders into an archive table
INSERT INTO orders_archive (order_id, customer_id, amount, created_at)
SELECT order_id, customer_id, amount, created_at
FROM orders
WHERE status = 'completed'
  AND created_at < '2024-01-01';
```

```sql
-- Load transformed data from a staging table
INSERT INTO customers (customer_id, first_name, last_name, email, country)
SELECT
    id,
    TRIM(first_name),
    TRIM(last_name),
    LOWER(email),
    UPPER(country_code)
FROM staging_customers
WHERE email IS NOT NULL;
```

!!! tip
    `INSERT ... SELECT` is pure ANSI SQL and the standard way to move or transform data between tables in a pipeline.

---

## INSERT with DEFAULT Values

```sql
-- Omit columns that have defaults defined
INSERT INTO orders (customer_id, amount)
VALUES (42, 199.99);
-- status defaults to 'pending', created_at defaults to CURRENT_TIMESTAMP

-- Explicitly use DEFAULT keyword
INSERT INTO orders (customer_id, amount, status)
VALUES (42, 199.99, DEFAULT);
```

---

## Conditional Insert — INSERT if Not Exists

Insert a row only if it does not already exist. The ANSI approach uses `MERGE` or a `NOT EXISTS` check.

```sql
-- Insert only if no matching row exists
INSERT INTO customers (customer_id, email)
SELECT 42, 'jorge@example.com'
WHERE NOT EXISTS (
    SELECT 1 FROM customers WHERE customer_id = 42
);
```

---

## Returning Inserted Values

Some databases support returning the values of inserted rows — useful for capturing auto-generated IDs.

```sql
-- PostgreSQL
INSERT INTO orders (customer_id, amount)
VALUES (42, 199.99)
RETURNING order_id, created_at;
```

!!! warning
    `RETURNING` is not ANSI SQL — it is specific to PostgreSQL and some other databases. SQL Server uses `OUTPUT INSERTED.*`, Oracle uses `RETURNING ... INTO`.

---

## Vendor Notes

| Feature | ANSI SQL | SQL Server | PostgreSQL | MySQL |
|---|---|---|---|---|
| Single row insert | ✅ | ✅ | ✅ | ✅ |
| Multi-row insert | ✅ | ✅ | ✅ | ✅ |
| Insert from SELECT | ✅ | ✅ | ✅ | ✅ |
| Return inserted values | — | `OUTPUT INSERTED.*` | `RETURNING` | `LAST_INSERT_ID()` |
| Insert or ignore | — | — | `ON CONFLICT DO NOTHING` | `INSERT IGNORE` |

---

## Best Practices

- Always specify the column list in `INSERT` — never rely on column position
- Use multi-row inserts or `INSERT ... SELECT` over looping single inserts for performance
- Apply transformations (TRIM, LOWER, UPPER) inside `INSERT ... SELECT` rather than in a separate step
- Use `NOT EXISTS` or `MERGE` for conditional inserts instead of checking in application code
- Wrap large inserts in transactions so they can be rolled back if something fails midway
