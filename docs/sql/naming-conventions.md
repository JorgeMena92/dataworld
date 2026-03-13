---
title: Naming Conventions
description: Consistent naming standards for tables, columns, constraints, and indexes
tags: [sql, ddl, naming, conventions]
---

# Naming Conventions

Consistent naming is one of the highest-leverage practices in database design. Good names make schemas self-documenting — you can understand the purpose of a table or column without reading any external documentation.

---

## General Rules

- Use **lowercase** with **underscores** as separators — `customer_id`, not `CustomerID` or `customerId`
- Be **descriptive but concise** — `order_date`, not `dt` or `the_date_the_order_was_placed`
- Use **English** — even in multilingual teams, English is the standard for code and schemas
- Avoid **reserved words** — `order`, `date`, `name`, `value` are reserved in SQL; prefix them to be safe
- Be **consistent** — the same concept should always have the same name across all tables

---

## Tables

- Use **plural nouns** for tables — `customers`, `orders`, `products`
- Prefix by layer or domain when using schemas — but avoid redundant prefixes inside a schema

```sql
-- Good
CREATE TABLE customers (...);
CREATE TABLE orders (...);
CREATE TABLE order_items (...);

-- Avoid
CREATE TABLE tbl_customers (...);    -- redundant 'tbl' prefix
CREATE TABLE Customer (...);         -- singular, mixed case
CREATE TABLE ORDERS (...);           -- all caps
```

---

## Columns

- Use **singular nouns** — `customer_id`, `first_name`, `created_at`
- Suffix primary keys with `_id` — `customer_id`, `order_id`, `product_id`
- Use the same name for foreign keys as the primary key they reference — `orders.customer_id` references `customers.customer_id`
- Use consistent suffixes for common patterns:

| Suffix | Meaning | Example |
|---|---|---|
| `_id` | Identifier / primary or foreign key | `customer_id` |
| `_at` | Timestamp | `created_at`, `updated_at`, `deleted_at` |
| `_date` | Date only (no time) | `order_date`, `birth_date` |
| `_count` | Integer count | `item_count`, `retry_count` |
| `_amount` | Monetary value | `total_amount`, `discount_amount` |
| `_flag` / `is_` | Boolean | `is_active`, `is_verified` |
| `_name` | Human-readable name | `first_name`, `product_name` |
| `_code` | Short code or identifier | `country_code`, `currency_code` |

```sql
CREATE TABLE orders (
    order_id      INT              PRIMARY KEY,
    customer_id   INT              NOT NULL,       -- FK matches PK name
    order_date    DATE             NOT NULL,
    total_amount  DECIMAL(10, 2)   NOT NULL,
    item_count    INT              DEFAULT 0,
    is_priority   BOOLEAN          DEFAULT FALSE,
    created_at    TIMESTAMP        DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP
);
```

---

## Constraints

Name constraints explicitly using a consistent prefix pattern.

| Prefix | Constraint type | Example |
|---|---|---|
| `pk_` | Primary key | `pk_customers` |
| `fk_` | Foreign key | `fk_orders_customer` |
| `uq_` | Unique | `uq_customers_email` |
| `chk_` | Check | `chk_orders_amount_positive` |

```sql
CREATE TABLE orders (
    order_id    INT           NOT NULL,
    customer_id INT           NOT NULL,
    amount      DECIMAL(10,2) NOT NULL,

    CONSTRAINT pk_orders             PRIMARY KEY (order_id),
    CONSTRAINT fk_orders_customer    FOREIGN KEY (customer_id) REFERENCES customers (customer_id),
    CONSTRAINT chk_orders_amount     CHECK (amount > 0)
);
```

---

## Indexes

```
idx_<table>_<column(s)>
```

```sql
CREATE INDEX idx_orders_customer_id   ON orders (customer_id);
CREATE INDEX idx_orders_status_date   ON orders (status, created_at);
CREATE UNIQUE INDEX idx_customers_email ON customers (email);
```

---

## Schemas

Use short, lowercase names that describe the layer or domain.

| Schema | Purpose |
|---|---|
| `raw` / `landing` | Raw data as-is from source systems |
| `staging` | Cleaned and transformed intermediate data |
| `analytics` / `mart` | Final tables for reporting and BI |
| `sales` / `finance` | Domain-specific schemas |
| `public` | Default schema (avoid using in production) |

```sql
CREATE SCHEMA staging;
CREATE SCHEMA analytics;

CREATE TABLE staging.orders (...);
CREATE TABLE analytics.monthly_revenue (...);
```

---

## Common Pitfalls

```sql
-- Avoid ambiguous names
date        → order_date, created_at, birth_date
name        → first_name, product_name, country_name
value       → total_amount, score, rating
status      → order_status, customer_status (qualify if multiple)
type        → product_type, account_type

-- Avoid inconsistent casing
CustomerID   → customer_id
orderDate    → order_date
PRODUCT_NAME → product_name

-- Avoid abbreviations that are not universally understood
qty  → quantity
amt  → amount
cust → customer
```

---

## Best Practices

- Agree on conventions with your team before starting a project — consistency matters more than which specific convention you choose
- Document your conventions in a README or wiki alongside the schema
- Use a linter or migration tool (dbt, Liquibase) to enforce naming rules automatically
- When working with an existing schema, follow its conventions rather than mixing styles
