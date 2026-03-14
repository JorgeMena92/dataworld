---
title: Introduction
description: What is SQL, scope, types, and tools
tags: [sql, introduction]
---

# Introduction

SQL (Structured Query Language) is the standard language for working with relational databases. It is one of the most valuable and enduring skills in data — used by analysts, engineers, developers, and scientists across virtually every industry.

---

## What is SQL?

SQL is a declarative language — you describe **what** you want, not **how** to get it. The database engine figures out the most efficient way to retrieve or manipulate the data.

```sql
-- You say what you want
SELECT name, salary
FROM employees
WHERE department = 'Engineering'
ORDER BY salary DESC;

-- The database figures out how to do it
```

SQL was developed in the 1970s at IBM, standardized by ANSI in 1986, and has been the dominant language for relational databases ever since.

---

## Scope

SQL can do much more than just query data. It covers the full lifecycle of working with a relational database:

- **Define** the structure of tables and schemas
- **Manipulate** data — insert, update, delete
- **Query** data — retrieve and aggregate information
- **Control** access — manage permissions and roles
- **Transact** — ensure data integrity with commits and rollbacks

---

## Types of SQL Commands

SQL is divided into five categories based on what they do:

| Category | Full Name | Purpose | Key Commands |
|---|---|---|---|
| DQL | Data Query Language | Read and retrieve data | `SELECT` |
| DML | Data Manipulation Language | Write and modify data | `INSERT`, `UPDATE`, `DELETE`, `MERGE` |
| DDL | Data Definition Language | Define database structure | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` |
| DCL | Data Control Language | Manage access and permissions | `GRANT`, `REVOKE` |
| TCL | Transaction Control Language | Manage transactions | `COMMIT`, `ROLLBACK`, `SAVEPOINT` |

See [SQL Command Categories](sql-command-categories.md) for a full breakdown with examples of each.

---

## Relational Databases

SQL works with relational databases — systems that store data in structured tables with defined relationships between them.

Popular relational database systems:

| Database | Owner | Best for |
|---|---|---|
| SQL Server | Microsoft | Enterprise, Azure ecosystem |
| PostgreSQL | Open source | General purpose, advanced features |
| MySQL | Oracle / Open source | Web applications |
| Oracle DB | Oracle | Enterprise, large scale |
| SQLite | Open source | Embedded, lightweight apps |
| BigQuery | Google | Cloud analytics at scale |
| Snowflake | Snowflake | Cloud data warehousing |
| Redshift | Amazon | AWS data warehousing |

---

## SQL vs NoSQL

SQL databases store data in structured tables with a fixed schema. NoSQL databases use flexible formats like documents, key-value pairs, or graphs.

| | SQL | NoSQL |
|---|---|---|
| Structure | Tables with rows and columns | Documents, key-value, graph |
| Schema | Fixed | Flexible |
| Relationships | Strong, enforced | Weak or none |
| Best for | Structured business data | Unstructured or rapidly changing data |
| Examples | SQL Server, PostgreSQL | MongoDB, Redis, Cassandra |

For most data analytics and BI work, SQL databases are the right choice.

---

## Why Learn SQL?

- It is the most used data skill across all data roles
- Every major data tool (Power BI, Tableau, dbt, Spark) supports SQL
- It transfers across platforms — core SQL works in any relational database
- It is readable and close to plain English
- It has been around for 50 years and is not going anywhere

!!! tip
    Learn ANSI SQL first — the standard that works everywhere. Then learn the specific extensions of the platform you use most (T-SQL for SQL Server, PL/pgSQL for PostgreSQL, etc.).