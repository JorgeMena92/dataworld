---
title: Triggers
description: Automatic actions on data events with SQL triggers
tags: [sql, database-objects, triggers]
---

# Triggers

A trigger is a database object that automatically executes a defined action when a specific event occurs on a table — an `INSERT`, `UPDATE`, or `DELETE`. Triggers fire without being called explicitly — they are attached to the table and respond to data changes.

---

## How Triggers Work

```
Data event (INSERT / UPDATE / DELETE)
    ↓
Trigger fires automatically
    ↓
Trigger body executes
    ↓
Original operation continues (or is cancelled)
```

---

## Trigger Timing

| Timing | Fires | Use case |
|---|---|---|
| `BEFORE` | Before the row is modified | Validate or modify data before it is saved |
| `AFTER` | After the row is modified | Audit logging, cascading updates |
| `INSTEAD OF` | Replaces the operation | Make views updatable |

---

## Trigger Scope

| Scope | Fires |
|---|---|
| `FOR EACH ROW` | Once per affected row |
| `FOR EACH STATEMENT` | Once per SQL statement, regardless of rows affected |

---

## Creating a Trigger

### Audit Log Trigger (AFTER INSERT/UPDATE/DELETE)

```sql
-- PostgreSQL — log all changes to customers
CREATE TABLE customers_audit (
    audit_id    INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id INT,
    operation   VARCHAR(10),
    changed_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    changed_by  VARCHAR(100)
);

CREATE OR REPLACE FUNCTION log_customer_changes()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO customers_audit (customer_id, operation, changed_by)
    VALUES (
        COALESCE(NEW.customer_id, OLD.customer_id),
        TG_OP,
        CURRENT_USER
    );
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_customers_audit
AFTER INSERT OR UPDATE OR DELETE
ON customers
FOR EACH ROW
EXECUTE FUNCTION log_customer_changes();
```

### Validation Trigger (BEFORE INSERT/UPDATE)

```sql
-- Prevent inserting orders with a future date
CREATE OR REPLACE FUNCTION validate_order_date()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF NEW.order_date > CURRENT_DATE THEN
        RAISE EXCEPTION 'Order date cannot be in the future: %', NEW.order_date;
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_validate_order_date
BEFORE INSERT OR UPDATE
ON orders
FOR EACH ROW
EXECUTE FUNCTION validate_order_date();
```

### Auto-Update Timestamp Trigger

One of the most common trigger use cases — keep `updated_at` current automatically.

```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_customers_updated_at
BEFORE UPDATE
ON customers
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
```

---

## NEW and OLD

Inside a trigger body, `NEW` and `OLD` give access to the row before and after the change.

| | INSERT | UPDATE | DELETE |
|---|---|---|---|
| `NEW` | ✅ New row | ✅ New values | ❌ |
| `OLD` | ❌ | ✅ Old values | ✅ Row being deleted |

```sql
-- Log what changed
IF NEW.email <> OLD.email THEN
    INSERT INTO email_change_log (customer_id, old_email, new_email, changed_at)
    VALUES (OLD.customer_id, OLD.email, NEW.email, CURRENT_TIMESTAMP);
END IF;
```

---

## Disabling and Dropping Triggers

```sql
-- PostgreSQL — disable a trigger
ALTER TABLE customers DISABLE TRIGGER trg_customers_audit;

-- Re-enable
ALTER TABLE customers ENABLE TRIGGER trg_customers_audit;

-- Drop
DROP TRIGGER trg_customers_audit ON customers;
DROP TRIGGER IF EXISTS trg_customers_audit ON customers;
```

---

## Vendor Notes

Triggers are supported across most databases but syntax varies significantly.

| Feature | PostgreSQL | SQL Server | MySQL | Oracle |
|---|---|---|---|---|
| BEFORE trigger | ✅ | ❌ (INSTEAD OF only for views) | ✅ | ✅ |
| AFTER trigger | ✅ | ✅ | ✅ | ✅ |
| INSTEAD OF | ✅ (views) | ✅ (views) | ❌ | ✅ |
| FOR EACH ROW | ✅ | Always row-level | ✅ | ✅ |
| FOR EACH STATEMENT | ✅ | ✅ | ❌ | ✅ |

```sql
-- SQL Server syntax
CREATE TRIGGER trg_customers_audit
ON customers
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    INSERT INTO customers_audit (customer_id, operation, changed_at)
    SELECT customer_id, 'CHANGE', GETDATE()
    FROM inserted;  -- SQL Server uses 'inserted' and 'deleted' instead of NEW/OLD
END;
```

---

## When to Use (and Avoid) Triggers

**Good use cases:**
- Audit logging — capture who changed what and when
- Auto-maintaining `updated_at` timestamps
- Enforcing complex constraints that `CHECK` cannot express
- Keeping denormalized summary columns in sync

**Avoid triggers when:**
- The logic can be handled in application code or a pipeline
- The trigger modifies other tables — cascading triggers are hard to debug
- Performance is critical — row-level triggers fire once per row and add overhead on bulk operations
- The team is not aware they exist — hidden logic is dangerous

!!! warning
    Triggers are invisible to most query tools and application developers. Undocumented triggers cause hard-to-diagnose bugs — unexpected data changes with no obvious cause. Always document triggers in the schema README and name them clearly.

---

## Naming Convention

```
trg_<table>_<timing>_<action>
```

```sql
trg_customers_before_insert
trg_orders_after_update
trg_customers_audit
trg_orders_updated_at
```

---

## Best Practices

- Use triggers sparingly — prefer application logic or pipeline logic when possible
- Name triggers clearly following a consistent convention
- Keep trigger logic simple — complex logic belongs in a stored procedure called by the trigger
- Always document triggers — make them visible to the whole team
- Disable triggers during bulk loads to avoid per-row overhead — re-enable after
- Test trigger behavior explicitly — they are easy to forget and hard to discover
