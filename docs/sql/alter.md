---
title: ALTER
description: Modifying existing database objects with ALTER in ANSI SQL
tags: [sql, ddl, alter]
---

# ALTER

`ALTER` modifies an existing database object — adding or removing columns, changing data types, renaming objects, and managing constraints. It is the primary tool for evolving a schema over time.

---

## ALTER TABLE — Columns

### Add a Column

```sql
ALTER TABLE customers
ADD COLUMN phone VARCHAR(20);

-- With a default value
ALTER TABLE orders
ADD COLUMN is_priority BOOLEAN DEFAULT FALSE;
```

### Drop a Column

```sql
-- ANSI SQL
ALTER TABLE customers
DROP COLUMN phone;

-- Vendor extension — supported in PostgreSQL and MySQL 8.0+, not in SQL Server or Oracle
ALTER TABLE customers
DROP COLUMN IF EXISTS phone;
```

!!! note
    `DROP COLUMN IF EXISTS` is not ANSI SQL — it is a vendor extension. In SQL Server, check `INFORMATION_SCHEMA.COLUMNS` first:
    ```sql
    -- SQL Server workaround
    IF EXISTS (
        SELECT 1 FROM INFORMATION_SCHEMA.COLUMNS
        WHERE TABLE_NAME = 'customers' AND COLUMN_NAME = 'phone'
    )
        ALTER TABLE customers DROP COLUMN phone;
    ```

### Rename a Column

```sql
ALTER TABLE customers
RENAME COLUMN phone TO phone_number;
```

### Change a Column's Data Type

```sql
-- ANSI SQL
ALTER TABLE customers
ALTER COLUMN phone SET DATA TYPE VARCHAR(30);
```

!!! note
    `ALTER COLUMN ... SET DATA TYPE` is the ANSI SQL form. PostgreSQL uses `ALTER COLUMN ... TYPE`, SQL Server uses `ALTER COLUMN phone VARCHAR(30)` with no keyword. MySQL uses `MODIFY COLUMN`. Always verify syntax for your platform before running in production.

!!! warning
    Changing a column's data type can fail if existing data is incompatible with the new type. Always check the data before altering types in production.

---

## ALTER TABLE — Constraints

### Add a Constraint

```sql
-- Add a NOT NULL constraint
ALTER TABLE customers
ALTER COLUMN email SET NOT NULL;

-- Add a UNIQUE constraint
ALTER TABLE customers
ADD CONSTRAINT uq_customers_email UNIQUE (email);

-- Add a CHECK constraint
ALTER TABLE orders
ADD CONSTRAINT chk_orders_amount CHECK (amount > 0);

-- Add a FOREIGN KEY
ALTER TABLE orders
ADD CONSTRAINT fk_orders_customer
    FOREIGN KEY (customer_id) REFERENCES customers (customer_id);

-- Add a PRIMARY KEY
ALTER TABLE products
ADD CONSTRAINT pk_products PRIMARY KEY (product_id);
```

### Drop a Constraint

```sql
ALTER TABLE orders
DROP CONSTRAINT chk_orders_amount;

ALTER TABLE orders
DROP CONSTRAINT fk_orders_customer;
```

---

## ALTER TABLE — Table-Level Changes

### Rename a Table

```sql
ALTER TABLE customers
RENAME TO clients;
```

### Set or Drop a Default Value

```sql
-- Set a default
ALTER TABLE orders
ALTER COLUMN status SET DEFAULT 'pending';

-- Remove a default
ALTER TABLE orders
ALTER COLUMN status DROP DEFAULT;
```

---

## ALTER INDEX

`ALTER INDEX` modifies an existing index. The most common operation is renaming — structural index changes typically require dropping and recreating the index instead.

```sql
-- Rename an index
ALTER INDEX idx_orders_customer_id
RENAME TO idx_orders_cust;
```

---

## ALTER SCHEMA

`ALTER SCHEMA` modifies an existing schema. Renaming is the most common use — moving objects between schemas is handled separately by each platform.

```sql
-- Rename a schema
ALTER SCHEMA staging RENAME TO raw;
```

---

## Schema Migration Pattern

In production, schema changes should be applied through versioned migration scripts — not run manually.

```sql
-- Migration: v1.2.0 — add loyalty tier to customers
ALTER TABLE customers
ADD COLUMN loyalty_tier VARCHAR(20) DEFAULT 'standard';

ALTER TABLE customers
ADD CONSTRAINT chk_loyalty_tier
    CHECK (loyalty_tier IN ('standard', 'silver', 'gold', 'platinum'));
```

Each migration script should follow these principles to be safe and maintainable:

- Idempotent where possible (`IF NOT EXISTS`, `IF EXISTS`)
- Version-controlled alongside application code
- Tested in a lower environment before production
- Reversible — paired with a rollback script

---

## Performance Considerations

`ALTER TABLE` on large tables can be slow and may lock the table for the duration of the operation, blocking reads and writes.

```sql
-- Adding a NOT NULL column with no default requires rewriting the entire table
-- This is safe on small tables but dangerous on large ones:
ALTER TABLE orders ADD COLUMN processed BOOLEAN NOT NULL DEFAULT FALSE;

-- Safer approach for large tables:
-- Step 1: add as nullable first (fast)
ALTER TABLE orders ADD COLUMN processed BOOLEAN;

-- Step 2: backfill in batches (controlled)
UPDATE orders SET processed = FALSE WHERE processed IS NULL;

-- Step 3: add NOT NULL constraint after backfill (fast, data already valid)
ALTER TABLE orders ALTER COLUMN processed SET NOT NULL;
```

---

## Vendor Notes

| Feature | ANSI SQL | SQL Server | PostgreSQL | MySQL |
|---|---|---|---|---|
| Add column | ✅ | ✅ | ✅ | ✅ |
| Drop column | ✅ | ✅ | ✅ | ✅ |
| Rename column | `RENAME COLUMN` | `sp_rename` | `RENAME COLUMN` | `CHANGE COLUMN` |
| Change data type | `ALTER COLUMN ... TYPE` | `ALTER COLUMN` | `ALTER COLUMN ... TYPE` | `MODIFY COLUMN` |
| Drop constraint | `DROP CONSTRAINT` | `DROP CONSTRAINT` | `DROP CONSTRAINT` | `DROP CONSTRAINT` / `DROP FOREIGN KEY` |

---

## Best Practices

- Name all constraints explicitly — unnamed constraints are hard to drop later
- Test `ALTER TABLE` on a copy of the table in non-production first
- For large tables, split `ALTER` operations into multiple steps to minimize locking
- Always write a rollback script alongside each migration
- Use a migration tool (Flyway, Liquibase, dbt) to manage schema changes systematically
- Never alter a production table manually — always use a reviewed migration script