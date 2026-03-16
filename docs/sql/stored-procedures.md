---
title: Stored Procedures
description: Reusable procedural logic stored in the database with SQL stored procedures
tags: [sql, database-objects, stored-procedures]
---

# Stored Procedures

A stored procedure is a named, reusable block of SQL and procedural logic stored in the database. Unlike functions, procedures can execute multiple statements, control transactions, modify data, and optionally return results — making them suitable for complex multi-step operations.

---

## Stored Procedures vs Functions

| | Stored Procedure | Function |
|---|---|---|
| Returns value | Optional | ✅ Always |
| Used in SELECT | ❌ | ✅ |
| Can modify data | ✅ | Limited |
| Transaction control | ✅ | ❌ (usually) |
| Called with | `CALL` / `EXEC` | Inside SQL expressions |
| ANSI SQL standard | SQL:1999 (`CREATE PROCEDURE`) | ✅ |

---

## Basic Syntax

```sql
-- ANSI SQL (SQL:1999)
CREATE PROCEDURE procedure_name (parameter1 type, parameter2 type)
LANGUAGE SQL
BEGIN
    -- SQL statements
END;
```

---

## Creating a Stored Procedure

```sql
-- PostgreSQL
CREATE OR REPLACE PROCEDURE update_customer_status(
    p_customer_id INT,
    p_status      VARCHAR
)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE customers
    SET
        status     = p_status,
        updated_at = CURRENT_TIMESTAMP
    WHERE customer_id = p_customer_id;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'Customer % not found', p_customer_id;
    END IF;
END;
$$;

-- Execute it
CALL update_customer_status(42, 'inactive');
```

!!! note
    `RAISE EXCEPTION` is PL/pgSQL (PostgreSQL) syntax for throwing errors. The equivalent in SQL Server is `THROW` or `RAISERROR`. MySQL uses `SIGNAL SQLSTATE`. There is no ANSI SQL standard for raising exceptions in procedural code — always use the syntax specific to your platform.

---

## Procedures with Transaction Control

One of the key advantages of procedures over functions — they can manage transactions internally.

```sql
-- PostgreSQL — order fulfillment procedure
CREATE OR REPLACE PROCEDURE fulfill_order(p_order_id INT)
LANGUAGE plpgsql
AS $$
DECLARE
    v_product_id INT;
    v_quantity   INT;
BEGIN
    -- Get the order details
    SELECT product_id, quantity
    INTO v_product_id, v_quantity
    FROM order_items
    WHERE order_id = p_order_id;

    -- Deduct from inventory
    UPDATE inventory
    SET stock = stock - v_quantity
    WHERE product_id = v_product_id;

    -- Check for negative stock
    IF (SELECT stock FROM inventory WHERE product_id = v_product_id) < 0 THEN
        ROLLBACK;
        RAISE EXCEPTION 'Insufficient stock for product %', v_product_id;
    END IF;

    -- Mark order as fulfilled
    UPDATE orders
    SET status = 'fulfilled', updated_at = CURRENT_TIMESTAMP
    WHERE order_id = p_order_id;

    COMMIT;
END;
$$;
```

---

## Procedures for ETL Pipelines

Stored procedures are commonly used to encapsulate data loading steps.

```sql
-- SQL Server style
CREATE PROCEDURE load_monthly_summary
    @load_month DATE
AS
BEGIN
    SET NOCOUNT ON;

    -- Clear existing data for the month
    DELETE FROM monthly_summary
    WHERE month = @load_month;

    -- Load fresh aggregation
    INSERT INTO monthly_summary (month, country, total_revenue, total_orders)
    SELECT
        @load_month,
        c.country,
        SUM(o.amount),
        COUNT(*)
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id
    WHERE YEAR(o.order_date)  = YEAR(@load_month)
      AND MONTH(o.order_date) = MONTH(@load_month)
      AND o.status = 'completed'
    GROUP BY c.country;
END;

-- Execute
EXEC load_monthly_summary @load_month = '2024-01-01';
```

---

## Output Parameters

Procedures can return values through output parameters.

```sql
-- PostgreSQL — return a value via INOUT parameter
CREATE OR REPLACE PROCEDURE get_customer_total(
    p_customer_id  INT,
    INOUT p_total  DECIMAL DEFAULT 0
)
LANGUAGE plpgsql
AS $$
BEGIN
    SELECT COALESCE(SUM(amount), 0)
    INTO p_total
    FROM orders
    WHERE customer_id = p_customer_id
      AND status = 'completed';
END;
$$;

-- Call with output
CALL get_customer_total(42, NULL);
```

---

## Dropping Procedures

```sql
-- ANSI SQL
DROP PROCEDURE update_customer_status;

-- Vendor extension — supported in PostgreSQL, MySQL, SQL Server (2016+)
DROP PROCEDURE IF EXISTS update_customer_status;

-- With parameter types when overloaded (PostgreSQL)
DROP PROCEDURE update_customer_status(INT, VARCHAR);
```

---

## Vendor Notes

Stored procedures are one of the least standardized areas of SQL. Syntax and capabilities differ significantly across databases.

| Feature | PostgreSQL | SQL Server | MySQL | Oracle |
|---|---|---|---|---|
| Create | `CREATE PROCEDURE` | `CREATE PROCEDURE` | `CREATE PROCEDURE` | `CREATE PROCEDURE` |
| Language | `plpgsql`, `SQL` | `T-SQL` | `SQL` | `PL/SQL` |
| Call | `CALL` | `EXEC` / `EXECUTE` | `CALL` | `EXEC` / `CALL` |
| Transaction in proc | ✅ | ✅ | ✅ | ✅ |
| Replace existing | `CREATE OR REPLACE` | `ALTER PROCEDURE` | `CREATE OR REPLACE` | `CREATE OR REPLACE` |

---

## When to Use (and Avoid) Stored Procedures

**Use stored procedures when:**
- Encapsulating complex multi-step data operations
- Running ETL logic that must be atomic
- Enforcing consistent business logic at the database level
- Reducing network round trips for multi-statement operations

**Avoid stored procedures when:**
- The logic is simple enough for a view or function
- You need to version control and test logic in an application layer
- The team uses a migration tool (dbt, Flyway) that manages transformations externally
- The procedure would be tightly coupled to a specific database vendor

!!! tip
    Modern data engineering tends to favor application-layer logic (dbt models, Python pipelines) over stored procedures for transformations — they are easier to test, version, and maintain. Use procedures where database-native execution and transaction control genuinely add value.

---

## Best Practices

- Name procedures as verb phrases describing what they do — `load_monthly_summary`, `fulfill_order`
- Include error handling and meaningful error messages
- Use transaction control explicitly — do not rely on implicit behavior
- Keep procedures focused — one business operation per procedure
- Version control procedure definitions alongside application code
- Document input and output parameters clearly