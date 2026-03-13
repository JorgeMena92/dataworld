---
title: SQL Command Categories
description: Understanding DQL, DML, DDL, DCL, and TCL command groups
tags: [sql, commands]
---

# SQL Command Categories

SQL commands are grouped into five categories based on what they do. Understanding this structure helps you navigate SQL systematically and know which commands belong to which layer of database work.

---

## Overview

| Category | Full Name | Purpose | Key Commands |
|---|---|---|---|
| DQL | Data Query Language | Read and retrieve data | `SELECT` |
| DML | Data Manipulation Language | Write and modify data | `INSERT`, `UPDATE`, `DELETE`, `MERGE` |
| DDL | Data Definition Language | Define database structure | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` |
| DCL | Data Control Language | Manage access and permissions | `GRANT`, `REVOKE` |
| TCL | Transaction Control Language | Manage transactions | `COMMIT`, `ROLLBACK`, `SAVEPOINT` |

---

## DQL — Data Query Language

DQL is the most used category. It covers everything related to **reading data** from a database without modifying it.

```sql
-- Retrieve all active customers
SELECT customer_id, first_name, email
FROM customers
WHERE status = 'active'
ORDER BY first_name;
```

!!! tip
    Some sources group `SELECT` under DML since it is part of the SQL standard's data manipulation section. In practice, DQL is widely used as a separate category because querying is the dominant day-to-day activity.

---

## DML — Data Manipulation Language

DML commands **write, modify, and delete data** inside tables. These are the commands used in ETL pipelines, application backends, and data loading processes.

```sql
-- Insert a new record
INSERT INTO customers (first_name, last_name, email)
VALUES ('Jorge', 'Mena', 'jorge@example.com');

-- Update an existing record
UPDATE customers
SET email = 'new@example.com'
WHERE customer_id = 42;

-- Delete a record
DELETE FROM customers
WHERE customer_id = 42;

-- Upsert — insert or update in one statement
MERGE INTO customers AS target
USING staging AS source
    ON target.customer_id = source.customer_id
WHEN MATCHED THEN
    UPDATE SET target.email = source.email
WHEN NOT MATCHED THEN
    INSERT (customer_id, first_name, email)
    VALUES (source.customer_id, source.first_name, source.email);
```

!!! warning
    Always include a `WHERE` clause in `UPDATE` and `DELETE`. Without it, every row in the table is affected.

---

## DDL — Data Definition Language

DDL commands define and manage the **structure** of a database — tables, schemas, indexes, constraints, and views. DDL changes affect the schema, not the data itself.

```sql
-- Create a table
CREATE TABLE orders (
    order_id    INT           PRIMARY KEY,
    customer_id INT           NOT NULL,
    amount      DECIMAL(10,2) CHECK (amount > 0),
    created_at  TIMESTAMP     DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- Add a column to an existing table
ALTER TABLE orders
ADD COLUMN status VARCHAR(20) DEFAULT 'pending';

-- Remove a table permanently
DROP TABLE IF EXISTS orders;

-- Remove all rows but keep the structure
TRUNCATE TABLE orders;
```

!!! tip
    Treat DDL scripts like code — version control them, review them, and never run them directly in production without testing first.

---

## DCL — Data Control Language

DCL commands manage **who can access what** in a database. This is the security and governance layer.

```sql
-- Grant read access to a role
GRANT SELECT ON customers TO analyst_role;

-- Grant write access to an application user
GRANT INSERT, UPDATE ON orders TO app_user;

-- Revoke access
REVOKE INSERT ON orders FROM app_user;
```

!!! tip
    Always manage permissions through **roles**, not individual users. When someone joins or leaves the team, you assign or revoke the role — not every individual permission.

---

## TCL — Transaction Control Language

TCL commands manage **transactions** — groups of operations that must all succeed or all fail together. This is how SQL guarantees data integrity.

```sql
BEGIN;

UPDATE accounts SET balance = balance - 500 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 500 WHERE account_id = 2;

-- If both succeed, commit
COMMIT;

-- If anything fails, roll back everything
ROLLBACK;
```

A `SAVEPOINT` lets you roll back to a specific point without undoing the entire transaction:

```sql
BEGIN;

INSERT INTO orders (...) VALUES (...);
SAVEPOINT after_insert;

UPDATE inventory SET stock = stock - 1 WHERE product_id = 10;

-- Only undo the UPDATE, keep the INSERT
ROLLBACK TO after_insert;

COMMIT;
```

!!! tip
    Wrap multi-step DML operations in a transaction. If one step fails, the whole operation rolls back and your data stays consistent.

---

## How the Categories Work Together

A typical data pipeline touches all five categories:

```
DDL  →  create the tables and define the schema
DML  →  load and transform the data
DQL  →  query the results for reporting
DCL  →  grant analysts access to the tables
TCL  →  ensure each load step is atomic and recoverable
```

Understanding which category a command belongs to helps you reason about what it does, what permissions it requires, and what risks it carries.