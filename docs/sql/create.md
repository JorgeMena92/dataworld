---
title: CREATE
description: Creating tables, schemas, indexes, and views with ANSI SQL CREATE
tags: [sql, ddl, create]
---

# CREATE

`CREATE` defines new database objects — tables, schemas, indexes, sequences, and views. It is the starting point of any database structure.

---

## CREATE TABLE

```sql
CREATE TABLE customers (
    customer_id   INT            PRIMARY KEY,
    first_name    VARCHAR(100)   NOT NULL,
    last_name     VARCHAR(100)   NOT NULL,
    email         VARCHAR(255)   UNIQUE,
    country       CHAR(2),
    created_at    TIMESTAMP      DEFAULT CURRENT_TIMESTAMP
);
```

### CREATE TABLE IF NOT EXISTS

Prevents errors when running scripts multiple times.

```sql
CREATE TABLE IF NOT EXISTS customers (
    customer_id INT PRIMARY KEY,
    email       VARCHAR(255) NOT NULL
);
```

### Inline Constraints

```sql
CREATE TABLE orders (
    order_id      INT             PRIMARY KEY,
    customer_id   INT             NOT NULL,
    amount        DECIMAL(10, 2)  CHECK (amount > 0),
    status        VARCHAR(20)     DEFAULT 'pending',
    created_at    TIMESTAMP       DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_orders_customer
        FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

---

## CREATE TABLE AS SELECT (CTAS)

Create a new table from the result of a query. Useful for snapshots, derived tables, and quick copies.

```sql
-- Copy a filtered subset
CREATE TABLE orders_2024 AS
SELECT *
FROM orders
WHERE EXTRACT(YEAR FROM created_at) = 2024;

-- Build a summary table
CREATE TABLE revenue_by_country AS
SELECT
    country,
    COUNT(*)    AS total_orders,
    SUM(amount) AS total_revenue
FROM orders
JOIN customers USING (customer_id)
GROUP BY country;
```

!!! tip
    `CTAS` copies the data but not the constraints or indexes. Add them explicitly after creation if needed.

---

## CREATE SCHEMA

Schemas organize objects into logical namespaces within a database.

```sql
CREATE SCHEMA sales;
CREATE SCHEMA analytics;
CREATE SCHEMA staging;

-- Create a table inside a schema
CREATE TABLE sales.orders (
    order_id INT PRIMARY KEY,
    amount   DECIMAL(10, 2)
);

-- Reference it with schema prefix
SELECT * FROM sales.orders;
```

---

## CREATE INDEX

Indexes speed up queries that filter, sort, or join on specific columns.

```sql
-- Basic index
CREATE INDEX idx_orders_customer_id
ON orders (customer_id);

-- Unique index — enforces uniqueness like a UNIQUE constraint
CREATE UNIQUE INDEX idx_customers_email
ON customers (email);

-- Composite index — for queries filtering on multiple columns
CREATE INDEX idx_orders_status_date
ON orders (status, created_at);
```

> See the **Indexes** page for full coverage of index types, strategies, and DDL patterns.

---

## CREATE SEQUENCE

Sequences generate unique incrementing numbers — commonly used for surrogate primary keys.

```sql
-- Create a sequence
CREATE SEQUENCE customer_id_seq
    START WITH 1
    INCREMENT BY 1
    NO CYCLE;

-- Use it in a table
CREATE TABLE customers (
    customer_id INT DEFAULT NEXT VALUE FOR customer_id_seq PRIMARY KEY,
    email       VARCHAR(255) NOT NULL
);
```

### ANSI Identity Column (preferred)

```sql
-- ANSI SQL:2003 standard — more portable than sequences for auto-increment keys
CREATE TABLE customers (
    customer_id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email       VARCHAR(255) NOT NULL
);
```

---

## CREATE VIEW

```sql
CREATE VIEW active_customers AS
SELECT customer_id, first_name, email
FROM customers
WHERE is_active = TRUE;

-- Query it like a table
SELECT * FROM active_customers;
```

> See the **Views** page under Database Objects for full coverage.

---

## Vendor Notes

| Feature | ANSI SQL | SQL Server | PostgreSQL | MySQL |
|---|---|---|---|---|
| Auto-increment | `GENERATED ALWAYS AS IDENTITY` | `IDENTITY(1,1)` | `SERIAL` / `IDENTITY` | `AUTO_INCREMENT` |
| IF NOT EXISTS | ✅ | ✅ (2016+) | ✅ | ✅ |
| CTAS | ✅ | `SELECT INTO` | ✅ | ✅ |
| Schema creation | ✅ | ✅ | ✅ | ✅ (= database) |

---

## Best Practices

- Always define a `PRIMARY KEY` on every table
- Use `IF NOT EXISTS` in scripts to make them safely re-runnable
- Name constraints explicitly — it makes them easier to drop and troubleshoot later
- Use `GENERATED ALWAYS AS IDENTITY` over vendor-specific auto-increment for portability
- Use schemas to separate concerns — `staging`, `analytics`, `sales`, etc.
- Version control all `CREATE` scripts alongside application code
