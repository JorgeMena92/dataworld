---
title: DDL — Data Definition Language
description: CREATE, ALTER, DROP, TRUNCATE with examples
tags: [sql, ddl]
---

# DDL — Data Definition Language

DDL commands define and manage the **structure** of a database — tables, schemas, indexes, and constraints. DDL changes affect the schema, not the data itself.

---

## CREATE

Creates a new database object — table, schema, index, or view.

### CREATE TABLE

```sql
CREATE TABLE customers (
    customer_id   INT           PRIMARY KEY,
    first_name    VARCHAR(100)  NOT NULL,
    last_name     VARCHAR(100)  NOT NULL,
    email         VARCHAR(255)  UNIQUE,
    created_at    DATE          DEFAULT CURRENT_DATE
);
```

### Data Types

| Type | Description |
|---|---|
| `INT` / `INTEGER` | Whole numbers |
| `BIGINT` | Large whole numbers |
| `DECIMAL(p, s)` | Exact decimal numbers |
| `FLOAT` / `REAL` | Approximate decimal numbers |
| `VARCHAR(n)` | Variable-length text up to n characters |
| `CHAR(n)` | Fixed-length text |
| `TEXT` | Unlimited text |
| `DATE` | Date only (YYYY-MM-DD) |
| `DATETIME` / `TIMESTAMP` | Date and time |
| `BOOLEAN` | True / False |

### Constraints

```sql
CREATE TABLE orders (
    order_id      INT           PRIMARY KEY,       -- unique, not null
    customer_id   INT           NOT NULL,          -- cannot be null
    amount        DECIMAL(10,2) CHECK (amount > 0), -- must pass condition
    status        VARCHAR(20)   DEFAULT 'pending', -- default value
    created_at    TIMESTAMP     DEFAULT NOW(),

    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

### CREATE SCHEMA

```sql
CREATE SCHEMA sales;
CREATE TABLE sales.orders (...);
```

### CREATE INDEX

```sql
-- Speed up queries that filter or sort by this column
CREATE INDEX idx_orders_customer_id
ON orders(customer_id);

-- Unique index
CREATE UNIQUE INDEX idx_customers_email
ON customers(email);
```

### CREATE VIEW

```sql
CREATE VIEW active_customers AS
SELECT customer_id, first_name, last_name
FROM customers
WHERE status = 'active';
```

---

## ALTER

Modifies an existing table structure — add, modify, or remove columns and constraints.

```sql
-- Add a column
ALTER TABLE customers
ADD COLUMN phone VARCHAR(20);

-- Modify a column type
ALTER TABLE customers
ALTER COLUMN phone TYPE VARCHAR(30);

-- Rename a column
ALTER TABLE customers
RENAME COLUMN phone TO phone_number;

-- Drop a column
ALTER TABLE customers
DROP COLUMN phone_number;

-- Add a constraint
ALTER TABLE orders
ADD CONSTRAINT fk_customer
FOREIGN KEY (customer_id) REFERENCES customers(customer_id);

-- Drop a constraint
ALTER TABLE orders
DROP CONSTRAINT fk_customer;
```

!!! warning
    `ALTER TABLE` on large tables can be slow and lock the table. In production, plan schema changes carefully and consider running them during maintenance windows.

---

## DROP

Permanently deletes a database object and all its data.

```sql
-- Drop a table
DROP TABLE orders;

-- Drop only if it exists (avoids error)
DROP TABLE IF EXISTS orders;

-- Drop a view
DROP VIEW active_customers;

-- Drop an index
DROP INDEX idx_orders_customer_id;

-- Drop a schema and everything in it
DROP SCHEMA sales CASCADE;
```

!!! danger
    `DROP` is irreversible. Always back up data before dropping tables in production.

---

## TRUNCATE

Removes all rows from a table but keeps the table structure intact. Faster than `DELETE` for clearing large tables.

```sql
TRUNCATE TABLE orders;

-- Some databases support truncating multiple tables
TRUNCATE TABLE orders, order_items;
```

| | TRUNCATE | DELETE |
|---|---|---|
| Removes all rows | ✅ | ✅ (without WHERE) |
| Can filter rows | ❌ | ✅ |
| Resets identity/sequence | ✅ (usually) | ❌ |
| Can be rolled back | Depends on DB | ✅ |
| Speed on large tables | Fast | Slow |

---

## Best Practices

- Always use `IF EXISTS` / `IF NOT EXISTS` to avoid errors in scripts
- Define constraints at table creation — not as an afterthought
- Use schemas to organize tables by domain or team
- Name constraints explicitly — makes them easier to drop and troubleshoot
- Version control your DDL scripts — treat schema changes like code
