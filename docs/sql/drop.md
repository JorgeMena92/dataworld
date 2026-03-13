---
title: DROP
description: Permanently removing database objects with DROP in ANSI SQL
tags: [sql, ddl, drop]
---

# DROP

`DROP` permanently removes a database object — table, schema, index, view, or constraint. Unlike `DELETE`, which removes rows, `DROP` removes the object itself along with all its data and structure. It cannot be undone outside of a transaction.

---

## DROP TABLE

```sql
-- Drop a table
DROP TABLE orders;

-- Drop only if it exists (safe for scripts)
DROP TABLE IF EXISTS orders;

-- Drop multiple tables
DROP TABLE IF EXISTS staging_orders, staging_customers;
```

!!! danger
    `DROP TABLE` permanently deletes the table and all its data. Always back up data before dropping tables in production.

---

## DROP with CASCADE

`CASCADE` automatically drops all objects that depend on the target — foreign keys, views, indexes.

```sql
-- Drop the table and everything that references it
DROP TABLE customers CASCADE;

-- Drop a schema and all objects inside it
DROP SCHEMA staging CASCADE;
```

### RESTRICT (Default)

```sql
-- Fails if any other object depends on this one
DROP TABLE customers RESTRICT;
-- Error: table is referenced by a foreign key in orders
```

!!! tip
    Use `RESTRICT` (the default) in production — it protects against accidentally dropping a table that other objects depend on. Use `CASCADE` only when you are certain about the full dependency chain.

---

## DROP SCHEMA

```sql
-- Drop an empty schema
DROP SCHEMA staging;

-- Drop a schema and all its contents
DROP SCHEMA staging CASCADE;

-- Safe version
DROP SCHEMA IF EXISTS staging CASCADE;
```

---

## DROP INDEX

```sql
DROP INDEX idx_orders_customer_id;

-- PostgreSQL
DROP INDEX IF EXISTS idx_orders_customer_id;

-- SQL Server (requires table name)
DROP INDEX idx_orders_customer_id ON orders;
```

---

## DROP VIEW

```sql
DROP VIEW active_customers;
DROP VIEW IF EXISTS active_customers;

-- Drop and recreate — common pattern when redeploying views
DROP VIEW IF EXISTS active_customers;
CREATE VIEW active_customers AS
SELECT customer_id, first_name, email
FROM customers
WHERE is_active = TRUE;
```

---

## DROP SEQUENCE

```sql
DROP SEQUENCE customer_id_seq;
DROP SEQUENCE IF EXISTS customer_id_seq;
```

---

## Safe DROP Pattern in Scripts

```sql
-- Always use IF EXISTS in migration scripts
DROP TABLE IF EXISTS temp_staging;
DROP INDEX IF EXISTS idx_temp_customer;
DROP VIEW IF EXISTS stale_report;
```

This makes scripts idempotent — they can be run multiple times without failing on the second run.

---

## Recovering from an Accidental DROP

In databases with transactional DDL (PostgreSQL, SQL Server), a `DROP` inside an uncommitted transaction can be rolled back.

```sql
-- PostgreSQL — DDL inside a transaction can be rolled back
BEGIN;
DROP TABLE customers;
-- Realize this was a mistake
ROLLBACK;
-- Table still exists
```

In databases without transactional DDL (MySQL, Oracle), a `DROP` is immediately permanent. Recovery requires restoring from a backup.

---

## Vendor Notes

| Feature | ANSI SQL | SQL Server | PostgreSQL | MySQL |
|---|---|---|---|---|
| `IF EXISTS` | ✅ | ✅ | ✅ | ✅ |
| `CASCADE` | ✅ | ❌ (manual) | ✅ | ❌ (manual) |
| Transactional DDL | Depends | ✅ | ✅ | ❌ |
| Drop index syntax | Varies | `DROP INDEX name ON table` | `DROP INDEX name` | `DROP INDEX name ON table` |

---

## Best Practices

- Always use `IF EXISTS` in scripts to prevent errors on re-runs
- Use `RESTRICT` (the default) in production — let the database warn you about dependencies
- Never run `DROP` manually in production — use reviewed migration scripts
- In databases with transactional DDL, wrap destructive drops in a transaction
- Back up data before dropping any table that contains records
- Prefer renaming a table over dropping it immediately — gives a recovery window

```sql
-- Safer than immediate DROP — rename first, verify, then drop later
ALTER TABLE customers RENAME TO customers_deprecated_20240101;
-- After confirming no issues:
DROP TABLE customers_deprecated_20240101;
```
