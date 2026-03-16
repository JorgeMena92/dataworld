---
title: Constraints
description: Enforcing data integrity with PRIMARY KEY, FOREIGN KEY, UNIQUE, CHECK, and NOT NULL
tags: [sql, ddl, constraints]
---

# Constraints

Constraints enforce rules on the data stored in a table. They are defined at the schema level and applied automatically by the database engine — preventing invalid data from being inserted or updated without any application-level code.

---

## Types of Constraints

| Constraint | Purpose |
|---|---|
| `PRIMARY KEY` | Uniquely identifies each row |
| `FOREIGN KEY` | Enforces referential integrity between tables |
| `UNIQUE` | Ensures all values in a column are distinct |
| `NOT NULL` | Prevents null values in a column |
| `CHECK` | Validates values against a condition |
| `DEFAULT` | Provides a default value when none is supplied |

!!! note
    `DEFAULT` is included here for completeness but behaves differently from the other constraint types — it does not prevent invalid data, it only supplies a fallback value when `INSERT` does not provide one. The remaining five constraint types actively enforce rules and reject invalid data.

---

## PRIMARY KEY

Uniquely identifies each row. Implicitly `NOT NULL` and `UNIQUE`. A table can have only one primary key.

```sql
-- Single column primary key
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    email       VARCHAR(255)
);

-- Named primary key (recommended)
CREATE TABLE customers (
    customer_id INT,
    email       VARCHAR(255),
    CONSTRAINT pk_customers PRIMARY KEY (customer_id)
);

-- Composite primary key
CREATE TABLE order_items (
    order_id   INT,
    product_id INT,
    quantity   INT,
    CONSTRAINT pk_order_items PRIMARY KEY (order_id, product_id)
);
```

---

## FOREIGN KEY

Enforces referential integrity — ensures a value in one table exists in another.

```sql
CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    customer_id INT NOT NULL,

    CONSTRAINT fk_orders_customer
        FOREIGN KEY (customer_id)
        REFERENCES customers (customer_id)
);
```

### Referential Actions

```sql
CONSTRAINT fk_orders_customer
    FOREIGN KEY (customer_id)
    REFERENCES customers (customer_id)
    ON DELETE CASCADE    -- delete orders when customer is deleted
    ON UPDATE CASCADE;   -- update customer_id in orders if it changes in customers
```

| Action | Behavior |
|---|---|
| `CASCADE` | Propagate the change to child rows |
| `SET NULL` | Set child column to NULL |
| `SET DEFAULT` | Set child column to its default value |
| `RESTRICT` | Prevent the change if child rows exist |
| `NO ACTION` | Same as RESTRICT (default) |

---

## UNIQUE

Ensures all values in a column (or combination of columns) are distinct.

```sql
-- Single column
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    email       VARCHAR(255) UNIQUE
);

-- Named unique constraint
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    email       VARCHAR(255),
    CONSTRAINT uq_customers_email UNIQUE (email)
);

-- Composite unique constraint
CREATE TABLE employee_roles (
    employee_id INT,
    role_id     INT,
    CONSTRAINT uq_employee_role UNIQUE (employee_id, role_id)
);
```

!!! tip
    A `UNIQUE` constraint allows `NULL` values — and most databases allow multiple `NULL` values in a `UNIQUE` column since `NULL` is not equal to `NULL`. If you need to prevent nulls, combine `UNIQUE` with `NOT NULL`.

---

## NOT NULL

Prevents a column from storing null values.

```sql
CREATE TABLE customers (
    customer_id INT          PRIMARY KEY,
    first_name  VARCHAR(100) NOT NULL,
    last_name   VARCHAR(100) NOT NULL,
    email       VARCHAR(255) NOT NULL
);

-- Add NOT NULL to an existing column
ALTER TABLE customers
ALTER COLUMN email SET NOT NULL;
```

---

## CHECK

Validates that column values satisfy a condition. Evaluated on every `INSERT` and `UPDATE`.

```sql
CREATE TABLE orders (
    order_id INT            PRIMARY KEY,
    amount   DECIMAL(10, 2) CHECK (amount > 0),
    status   VARCHAR(20)    CHECK (status IN ('pending', 'completed', 'cancelled'))
);

-- Named CHECK constraint (recommended)
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    price      DECIMAL(10, 2),
    discount   DECIMAL(5, 2),

    CONSTRAINT chk_price_positive   CHECK (price > 0),
    CONSTRAINT chk_discount_range   CHECK (discount BETWEEN 0 AND 100)
);
```

!!! warning
    A `CHECK` constraint evaluates to UNKNOWN (not FALSE) when any column in the condition is NULL — which means NULL values silently pass the check. If a column must satisfy a condition and never be NULL, combine `CHECK` with `NOT NULL`:
    ```sql
    price DECIMAL(10, 2) NOT NULL CHECK (price > 0)
    ```

---

## DEFAULT

Provides a value when `INSERT` does not supply one.

```sql
CREATE TABLE orders (
    order_id   INT           PRIMARY KEY,
    status     VARCHAR(20)   DEFAULT 'pending',
    created_at TIMESTAMP     DEFAULT CURRENT_TIMESTAMP,
    is_active  BOOLEAN       DEFAULT TRUE
);

-- Add a default to an existing column
ALTER TABLE orders
ALTER COLUMN status SET DEFAULT 'pending';
```

---

## Naming Constraints

Always name constraints explicitly. Unnamed constraints get auto-generated names that are different across databases and hard to reference in `ALTER` and error messages.

```sql
-- Recommended naming pattern
CONSTRAINT pk_customers       PRIMARY KEY (customer_id)
CONSTRAINT fk_orders_customer FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
CONSTRAINT uq_customers_email UNIQUE (email)
CONSTRAINT chk_orders_amount  CHECK (amount > 0)
```

Prefix convention:
- `pk_` — primary key
- `fk_` — foreign key
- `uq_` — unique
- `chk_` — check

---

## Adding and Dropping Constraints

```sql
-- Add constraints after table creation
ALTER TABLE orders
ADD CONSTRAINT fk_orders_customer
    FOREIGN KEY (customer_id) REFERENCES customers (customer_id);

ALTER TABLE products
ADD CONSTRAINT chk_price_positive CHECK (price > 0);

-- Drop a constraint
ALTER TABLE orders
DROP CONSTRAINT fk_orders_customer;

ALTER TABLE products
DROP CONSTRAINT chk_price_positive;
```

---

## Deferrable Constraints

ANSI SQL supports deferring constraint checks to the end of a transaction — useful when inserting rows with circular references or when the order of inserts would temporarily violate a constraint.

```sql
-- ANSI SQL — deferrable foreign key
ALTER TABLE orders
ADD CONSTRAINT fk_orders_customer
    FOREIGN KEY (customer_id) REFERENCES customers (customer_id)
    DEFERRABLE INITIALLY DEFERRED;
```

When set to `INITIALLY DEFERRED`, the constraint is checked at `COMMIT` rather than at each statement. When set to `INITIALLY IMMEDIATE` (the default), it is checked after each statement but can be deferred explicitly within a transaction.

!!! note
    `DEFERRABLE` constraints are supported in PostgreSQL and Oracle. SQL Server does not support deferrable constraints — referential integrity is always checked immediately after each statement in SQL Server.

---

## Best Practices

- Define constraints at table creation — not as an afterthought
- Name every constraint explicitly using a consistent prefix convention
- Use foreign keys to enforce referential integrity at the database level — do not rely on application code alone
- Combine `UNIQUE` with `NOT NULL` when a column must be both distinct and present
- Use `CHECK` constraints to enforce business rules close to the data
- Choose referential actions (`CASCADE`, `RESTRICT`) deliberately — understand the full impact on child data