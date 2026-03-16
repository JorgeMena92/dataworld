---
title: Schema Design
description: Normalization, keys, relationships, and design decisions for relational schemas
tags: [sql, ddl, schema, design]
---

# Schema Design

Schema design is the process of organizing tables, columns, and relationships to accurately represent a domain, support the required queries, and maintain data integrity over time. Good design prevents data anomalies, reduces redundancy, and makes the schema easier to evolve.

---

## Normalization

Normalization is the process of structuring a schema to reduce redundancy and prevent update anomalies. It is organized into normal forms — each one building on the previous.

### First Normal Form (1NF)

Each column holds a single atomic value. No repeating groups or arrays in columns.

```sql
-- Violates 1NF — multiple values in one column
CREATE TABLE orders (
    order_id  INT,
    products  VARCHAR(500)  -- 'Laptop, Monitor, Keyboard'
);

-- 1NF — separate table for line items
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT
);

CREATE TABLE order_items (
    item_id    INT PRIMARY KEY,
    order_id   INT REFERENCES orders(order_id),
    product_id INT,
    quantity   INT
);
```

### Second Normal Form (2NF)

Satisfies 1NF. Every non-key column depends on the **entire** primary key — not just part of it. Relevant when the primary key is composite.

```sql
-- Violates 2NF — product_name depends only on product_id, not the full key
CREATE TABLE order_items (
    order_id     INT,
    product_id   INT,
    product_name VARCHAR(100),  -- depends only on product_id
    quantity     INT,
    PRIMARY KEY (order_id, product_id)
);

-- 2NF — move product_name to products table
CREATE TABLE products (
    product_id   INT PRIMARY KEY,
    product_name VARCHAR(100)
);

CREATE TABLE order_items (
    order_id   INT,
    product_id INT REFERENCES products(product_id),
    quantity   INT,
    PRIMARY KEY (order_id, product_id)
);
```

### Third Normal Form (3NF)

Satisfies 2NF. No non-key column depends on another non-key column (no transitive dependencies).

```sql
-- Violates 3NF — zip_code determines city, not the primary key directly
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    zip_code    CHAR(5),
    city        VARCHAR(100)  -- depends on zip_code, not customer_id
);

-- 3NF — extract the transitive dependency
CREATE TABLE zip_codes (
    zip_code CHAR(5) PRIMARY KEY,
    city     VARCHAR(100)
);

CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    zip_code    CHAR(5) REFERENCES zip_codes(zip_code)
);
```

!!! tip
    For most OLTP (transactional) systems, 3NF is the target. For analytical systems (data warehouses), controlled denormalization is common for query performance.

---

## Surrogate vs Natural Keys

### Natural Key

A key that uses an existing real-world attribute to uniquely identify a row.

```sql
CREATE TABLE countries (
    country_code CHAR(2) PRIMARY KEY,  -- natural key (ISO 3166)
    country_name VARCHAR(100)
);
```

**Pros:** meaningful, no extra column needed
**Cons:** can change over time, may not be truly unique, can be long

### Surrogate Key

An artificial key with no business meaning — typically an auto-generated integer or UUID.

```sql
CREATE TABLE customers (
    customer_id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- surrogate key
    email       VARCHAR(255) UNIQUE NOT NULL                   -- natural key as unique constraint
);
```

**Pros:** stable, compact, independent of business rules
**Cons:** no inherent meaning, requires a separate unique constraint for the natural key

!!! tip
    Use surrogate keys as primary keys in most cases. Keep natural keys as `UNIQUE NOT NULL` constraints so the business identifier is still enforced without being the join key.

---

## Table Relationships

### One-to-Many

The most common relationship. One customer has many orders.

```sql
CREATE TABLE customers (
    customer_id INT PRIMARY KEY
);

CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    customer_id INT NOT NULL REFERENCES customers(customer_id)
);
```

### Many-to-Many

Requires a junction table to resolve the relationship.

```sql
CREATE TABLE students (
    student_id INT PRIMARY KEY
);

CREATE TABLE courses (
    course_id INT PRIMARY KEY
);

CREATE TABLE enrollments (
    student_id INT REFERENCES students(student_id),
    course_id  INT REFERENCES courses(course_id),
    enrolled_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (student_id, course_id)
);
```

### One-to-One

A row in one table corresponds to exactly one row in another. Used to split a table with many columns or to isolate sensitive data.

```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY
);

CREATE TABLE employee_profiles (
    employee_id INT PRIMARY KEY REFERENCES employees(employee_id),
    bio         VARCHAR(2000),
    photo_url   VARCHAR(500)
);
```

---

## OLTP vs OLAP Schema Design

| | OLTP | OLAP / Data Warehouse |
|---|---|---|
| Goal | Fast writes, data integrity | Fast reads, analytical queries |
| Normalization | High (3NF) | Low (denormalized) |
| Schema pattern | Normalized tables | Star or snowflake schema |
| Joins | Many, complex | Fewer, pre-joined dimensions |
| Key type | Surrogate + natural | Surrogate keys in dimensions |

### Star Schema (OLAP)

In a star schema, a central fact table references multiple dimension tables. The fact table holds measurable events (sales, transactions), and the dimensions hold descriptive context (customer, product, date).

```sql
-- Fact table — transactions
CREATE TABLE fact_sales (
    sale_id      INT PRIMARY KEY,
    date_key     INT REFERENCES dim_date(date_key),
    customer_key INT REFERENCES dim_customer(customer_key),
    product_key  INT REFERENCES dim_product(product_key),
    amount       DECIMAL(10, 2),
    quantity     INT
);

-- Dimension table — descriptive attributes
CREATE TABLE dim_customer (
    customer_key INT PRIMARY KEY,
    customer_id  INT UNIQUE,
    first_name   VARCHAR(100),
    country      VARCHAR(100),
    segment      VARCHAR(50)
);

-- Date dimension — one row per calendar day
CREATE TABLE dim_date (
    date_key    INT PRIMARY KEY,   -- e.g. 20240115
    full_date   DATE NOT NULL,
    year        INT,
    month       INT,
    quarter     INT,
    day_of_week VARCHAR(10),
    is_weekend  BOOLEAN
);
```

!!! note
    A **snowflake schema** is a variation of the star schema where dimension tables are further normalized — for example, splitting `country` out of `dim_customer` into its own `dim_country` table. This reduces redundancy but adds joins. Star schemas are generally preferred in analytical workloads for query simplicity and performance.

---

## Audit Columns

Add standard audit columns to every table that tracks changes over time.

```sql
CREATE TABLE customers (
    customer_id INT       PRIMARY KEY,
    email       VARCHAR(255) NOT NULL,

    -- Audit columns
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP,
    created_by  VARCHAR(100),
    updated_by  VARCHAR(100),
    is_active   BOOLEAN   DEFAULT TRUE,
    deleted_at  TIMESTAMP  -- soft delete timestamp
);
```

---

## Best Practices

- Start with 3NF for transactional systems, then denormalize only where query performance demands it
- Use surrogate keys as primary keys — keep natural identifiers as unique constraints
- Always define foreign keys to enforce referential integrity at the database level
- Add audit columns (`created_at`, `updated_at`) to every table that changes over time
- Design for the queries you need to run — a schema that cannot answer your questions efficiently is not a good schema
- Separate OLTP and OLAP concerns — do not design a transactional schema and a reporting schema the same way