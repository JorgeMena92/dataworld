---
title: DCL — Data Control Language
description: GRANT, REVOKE, and access control with examples
tags: [sql, dcl]
---

# DCL — Data Control Language

DCL commands manage **access and permissions** in a database — who can read, write, or modify data and objects. This is essential for security and governance in any production environment.

---

## GRANT

Gives a user or role permission to perform actions on database objects.

```sql
-- Grant SELECT on a table
GRANT SELECT ON customers TO analyst_user;

-- Grant multiple privileges
GRANT SELECT, INSERT, UPDATE ON orders TO app_user;

-- Grant on all tables in a schema
GRANT SELECT ON ALL TABLES IN SCHEMA sales TO analyst_role;

-- Grant to a role (recommended over individual users)
GRANT SELECT ON customers TO analyst_role;

-- Grant with the ability to pass the permission to others
GRANT SELECT ON customers TO manager_user WITH GRANT OPTION;
```

### Common Privileges

| Privilege | Description |
|---|---|
| `SELECT` | Read data from a table or view |
| `INSERT` | Add new rows |
| `UPDATE` | Modify existing rows |
| `DELETE` | Remove rows |
| `EXECUTE` | Run stored procedures or functions |
| `ALL PRIVILEGES` | All available permissions |
| `REFERENCES` | Create foreign key references |
| `CREATE` | Create new objects in a schema |

---

## REVOKE

Removes a previously granted permission.

```sql
-- Revoke a specific privilege
REVOKE INSERT ON orders FROM app_user;

-- Revoke multiple privileges
REVOKE INSERT, UPDATE ON orders FROM app_user;

-- Revoke all privileges
REVOKE ALL PRIVILEGES ON customers FROM analyst_user;

-- Revoke from a role
REVOKE SELECT ON customers FROM analyst_role;
```

---

## Roles

Roles are named groups of permissions. Granting permissions to roles instead of individual users makes access management much easier — especially in large teams.

```sql
-- Create a role
CREATE ROLE analyst_role;

-- Grant permissions to the role
GRANT SELECT ON ALL TABLES IN SCHEMA sales TO analyst_role;

-- Assign the role to a user
GRANT analyst_role TO jorge_mena;

-- Remove the role from a user
REVOKE analyst_role FROM jorge_mena;

-- Drop a role
DROP ROLE analyst_role;
```

!!! tip
    Always manage permissions through roles, not individual users. When someone joins or leaves the team, you only need to assign or revoke the role — not update every individual permission.

---

## Row-Level Security (RLS)

Some databases support row-level security — restricting which rows a user can see based on their identity.

```sql
-- PostgreSQL example
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY orders_by_region
ON orders
FOR SELECT
USING (region = current_setting('app.current_region'));
```

In Power BI, RLS is implemented at the semantic model level using DAX filters — the same concept applied to the BI layer.

---

## Best Practices

- Use roles instead of granting permissions directly to users
- Follow the **principle of least privilege** — grant only what is needed
- Regularly audit permissions and remove unused access
- Never grant `ALL PRIVILEGES` to application users
- Use schemas to separate data by sensitivity level and apply permissions at the schema level
- Document who has access to what, especially for sensitive data

```sql
-- Audit current permissions (PostgreSQL)
SELECT grantee, table_name, privilege_type
FROM information_schema.role_table_grants
WHERE table_schema = 'sales'
ORDER BY grantee, table_name;
```
