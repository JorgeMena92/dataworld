---
title: DDL Fundamentals
description: Introduction to Data Definition Language — defining and managing database structure
tags: [sql, ddl, fundamentals]
---

# DDL Fundamentals

Data Definition Language (DDL) defines and manages the **structure** of a database — tables, schemas, indexes, constraints, and other objects. DDL changes affect the schema itself, not the data inside it.

---

## DDL Commands

| Command | Purpose |
|---|---|
| `CREATE` | Create a new database object |
| `ALTER` | Modify an existing object |
| `DROP` | Permanently delete an object |
| `TRUNCATE` | Remove all rows from a table, keeping its structure |

---

## How DDL Fits in the SQL Lifecycle

```
DDL  →  CREATE tables, schemas, indexes, constraints
DML  →  INSERT, UPDATE, DELETE, MERGE data
DQL  →  SELECT and query the results
TCL  →  COMMIT or ROLLBACK changes
DCL  →  GRANT access to the right users
```

DDL defines the container. DML fills it with data.

---

## DDL and Transactions

DDL behavior inside transactions varies by database — this is one of the most important cross-platform differences.

| Database | DDL in transactions | Auto-commit |
|---|---|---|
| PostgreSQL | ✅ Transactional DDL | No |
| SQL Server | ✅ Transactional DDL | No |
| MySQL | ❌ DDL causes implicit commit | Yes |
| Oracle | ❌ DDL causes implicit commit | Yes |

```sql
-- PostgreSQL — DDL can be rolled back
BEGIN;
CREATE TABLE test (id INT);
ROLLBACK;
-- Table was never created

-- MySQL — DDL commits immediately, cannot be rolled back
BEGIN;
CREATE TABLE test (id INT);
ROLLBACK;
-- Table still exists — CREATE committed implicitly
```

!!! tip
    When writing migration scripts that need to run on multiple platforms, treat DDL as non-transactional for safety. Test schema changes in a non-production environment first.

---

## DDL Scope — What It Manages

```
Database
└── Schema
    ├── Tables
    │   ├── Columns
    │   └── Constraints (PK, FK, UNIQUE, CHECK, NOT NULL)
    ├── Indexes
    ├── Views
    ├── Materialized Views
    ├── Functions
    ├── Stored Procedures
    └── Triggers
```

All of these objects are created, modified, and dropped using DDL commands.

---

## Safety Rules for DDL

- Always use `IF EXISTS` / `IF NOT EXISTS` in scripts to avoid errors on re-runs
- Never run `DROP` or destructive `ALTER` directly in production without a backup
- Version control all DDL scripts — treat schema changes like application code
- Test migrations in a lower environment before applying to production
- For large tables, plan `ALTER TABLE` operations carefully — they can lock the table

```sql
-- Safe script pattern
CREATE TABLE IF NOT EXISTS customers (
    customer_id INT PRIMARY KEY,
    email       VARCHAR(255) UNIQUE NOT NULL
);

DROP TABLE IF EXISTS temp_staging;
```
