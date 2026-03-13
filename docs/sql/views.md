---
title: Views
description: Creating and using views as reusable virtual tables in ANSI SQL
tags: [sql, database-objects, views]
---

# Views

A view is a named query stored in the database that behaves like a virtual table. It does not store data itself — it executes the underlying query every time it is referenced. Views simplify complex queries, enforce consistent data access, and add a layer of abstraction between the data and its consumers.

> Views are created with DDL (`CREATE VIEW`) but grouped here under Database Objects because they are reusable schema-level objects with their own lifecycle, independent of table structure management.

---

## Basic Syntax

```sql
CREATE VIEW view_name AS
SELECT ...
FROM ...
WHERE ...;
```

---

## Creating a View

```sql
-- Simple filter view
CREATE VIEW active_customers AS
SELECT customer_id, first_name, last_name, email
FROM customers
WHERE is_active = TRUE;

-- Query it like a table
SELECT * FROM active_customers;
SELECT * FROM active_customers WHERE country = 'Peru';
```

```sql
-- View that joins multiple tables
CREATE VIEW order_summary AS
SELECT
    o.order_id,
    c.first_name,
    c.last_name,
    c.country,
    o.amount,
    o.status,
    o.created_at
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;
```

---

## CREATE OR REPLACE VIEW

Update a view definition without dropping and recreating it.

```sql
CREATE OR REPLACE VIEW active_customers AS
SELECT customer_id, first_name, last_name, email, country
FROM customers
WHERE is_active = TRUE;
```

!!! tip
    `CREATE OR REPLACE VIEW` is supported in PostgreSQL, MySQL, and Oracle. SQL Server requires `DROP VIEW` followed by `CREATE VIEW` — or use `ALTER VIEW`.

---

## DROP VIEW

```sql
DROP VIEW active_customers;
DROP VIEW IF EXISTS active_customers;
```

---

## Views for Security and Access Control

Views are a clean way to expose only the columns and rows a user should see — without granting direct table access.

```sql
-- Expose customer data without sensitive columns
CREATE VIEW customers_public AS
SELECT customer_id, first_name, country, segment
FROM customers;
-- email, phone, and payment info are excluded

-- Grant access to the view, not the table
GRANT SELECT ON customers_public TO analyst_role;
```

---

## Views for Reusable Business Logic

Encode complex business rules once in a view — all consumers use the same definition.

```sql
CREATE VIEW high_value_orders AS
SELECT
    o.order_id,
    o.customer_id,
    o.amount,
    c.segment
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.amount > 1000
  AND o.status = 'completed'
  AND c.segment = 'enterprise';
```

---

## Updatable Views

In ANSI SQL, a view is updatable if it meets certain conditions — no aggregations, no `DISTINCT`, no `GROUP BY`, based on a single table.

```sql
-- Simple updatable view
CREATE VIEW active_customers AS
SELECT customer_id, first_name, email
FROM customers
WHERE is_active = TRUE;

-- This UPDATE applies to the underlying table
UPDATE active_customers
SET email = 'new@example.com'
WHERE customer_id = 42;
```

!!! warning
    Complex views with joins, aggregations, or subqueries are generally not updatable. Treat views primarily as read-only objects for safety.

---

## Views vs CTEs

Both provide named, reusable query logic — but they serve different scopes.

| | View | CTE |
|---|---|---|
| Scope | Persistent — available across sessions | Single query only |
| Stored in database | ✅ | ❌ |
| Can be shared with other users | ✅ | ❌ |
| Executes query on every reference | ✅ | ✅ (usually) |
| Best for | Reusable cross-session logic | Step-by-step logic within a query |

---

## Vendor Notes

| Feature | ANSI SQL | SQL Server | PostgreSQL | MySQL |
|---|---|---|---|---|
| Create view | ✅ | ✅ | ✅ | ✅ |
| Replace view | — | `ALTER VIEW` | `CREATE OR REPLACE` | `CREATE OR REPLACE` |
| Updatable views | ✅ | ✅ | ✅ | ✅ |
| Schema-bound views | — | `WITH SCHEMABINDING` | — | — |

---

## Best Practices

- Use views to abstract complexity — consumers should not need to know the underlying joins
- Use views to enforce consistent business logic across reports and queries
- Use views to restrict column and row access without changing table-level permissions
- Name views clearly to distinguish them from tables — some teams use a `v_` prefix (`v_active_customers`)
- Avoid deeply nested views — a view built on a view built on a view becomes hard to debug
- Document what each view represents and who its intended consumers are
