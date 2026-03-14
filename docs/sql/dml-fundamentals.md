---
title: DML Fundamentals
description: Introduction to Data Manipulation Language — writing, modifying, and deleting data
tags: [sql, dml, fundamentals]
---

# DML Fundamentals

Data Manipulation Language (DML) covers the commands that write and modify data inside tables. While DQL focuses on reading, DML focuses on writing — inserting new records, updating existing ones, deleting obsolete ones, and merging sources together.

---

## DML Commands

| Command | Purpose |
|---|---|
| `INSERT` | Add new rows to a table |
| `UPDATE` | Modify existing rows |
| `DELETE` | Remove rows |
| `MERGE` | Insert or update based on a match condition |

!!! note
    `TRUNCATE` removes all rows from a table quickly and is classified as DDL in ANSI SQL — it changes the state of the table rather than manipulating individual rows. In practice it is often used as part of a DML pipeline (full load pattern). See [TRUNCATE](truncate.md) and [Schema Design](schema-design.md) for details.

---

## How DML Fits in the SQL Lifecycle

```
DDL  →  CREATE the tables and define structure
DML  →  INSERT, UPDATE, DELETE, MERGE data
DQL  →  SELECT and query the results
TCL  →  COMMIT or ROLLBACK the changes
DCL  →  GRANT access to the right users
```

DML commands interact with the data, not the structure. They never change the shape of a table — only its contents.

---

## Transactions and DML

Every DML statement runs inside a transaction — either explicitly defined or implicit. This means changes can be committed (saved permanently) or rolled back (undone entirely).

```sql
BEGIN;

INSERT INTO orders (customer_id, amount) VALUES (42, 199.99);
UPDATE inventory SET stock = stock - 1 WHERE product_id = 7;

COMMIT;   -- both changes saved together
-- or
ROLLBACK; -- both changes undone
```

!!! tip
    Wrapping related DML operations in a transaction ensures atomicity — either all changes succeed or none of them do. This is critical for data integrity in multi-step pipelines. See [Transactions](transactions.md) for the full treatment.

---

## DML in Data Engineering

In data engineering, DML is the core of every pipeline. The typical patterns are:

| Pattern | Description |
|---|---|
| Full load | Truncate and reload the table completely |
| Incremental load | Insert only new records since the last run |
| Upsert | Insert new records, update existing ones |
| Soft delete | Mark rows as inactive instead of physically removing them |

Each pattern has tradeoffs between simplicity, performance, and recoverability.

---

## Test Before You Execute

Always validate the scope of a `UPDATE` or `DELETE` by running the same `WHERE` clause as a `SELECT` first — confirm the row count before making any changes.

```sql
-- Test before you delete
SELECT COUNT(*)
FROM orders
WHERE status = 'cancelled'
  AND created_at < '2023-01-01';

-- Then delete
DELETE FROM orders
WHERE status = 'cancelled'
  AND created_at < '2023-01-01';
```

---

## Best Practices

- Always test `UPDATE` and `DELETE` with a `SELECT` using the same `WHERE` clause first
- Always include a `WHERE` clause — without it, every row in the table is affected
- Use transactions for multi-step operations
- Prefer soft deletes over hard deletes when auditability matters