---
title: Warehouse
description: T-SQL analytics warehouse, SQL patterns, and cross-lakehouse queries in Microsoft Fabric
tags: [fabric, warehouse, sql, t-sql]
---

# Warehouse

The Fabric Warehouse is a fully managed, serverless SQL analytics warehouse that stores data natively in Delta Lake format on OneLake. It provides full T-SQL support — including DML, DDL, and stored procedures — and integrates with every other Fabric workload without data movement.

---

## What is a Warehouse?

Unlike the Lakehouse SQL analytics endpoint (which is read-only), the Fabric Warehouse supports full read-write T-SQL operations. It is purpose-built for the Gold layer of the medallion architecture — structured, business-ready data optimized for SQL queries and Power BI consumption.

| | Lakehouse SQL endpoint | Warehouse |
|--|----------------------|-----------|
| Read (SELECT) | ✅ | ✅ |
| Write (INSERT, UPDATE, DELETE) | ❌ | ✅ |
| Stored procedures | ❌ | ✅ |
| Spark access | ✅ Native | ✅ Via shortcut |
| Unstructured files | ❌ | ❌ |
| Best for | Ad-hoc SQL on Lakehouse tables | Gold layer, BI serving, T-SQL transformations |

---

## Creating Tables

### CREATE TABLE

```sql
CREATE TABLE gold.dim_customer (
    customer_key  INT           NOT NULL,
    customer_id   VARCHAR(50)   NOT NULL,
    customer_name VARCHAR(200),
    email         VARCHAR(200),
    region        VARCHAR(100),
    segment       VARCHAR(50),
    created_date  DATE,
    updated_date  DATE
);
```

### CREATE TABLE AS SELECT (CTAS)

The fastest way to create and populate a table from a query — useful for building Gold layer tables from Silver sources:

```sql
CREATE TABLE gold.fact_sales
AS
SELECT
    o.order_id,
    o.customer_id,
    o.product_id,
    o.order_date,
    o.quantity,
    o.amount            AS revenue,
    p.cost              AS cogs,
    o.amount - p.cost   AS gross_profit
FROM silver_lakehouse.silver.orders o
LEFT JOIN silver_lakehouse.silver.products p ON o.product_id = p.product_id;
```

!!! note
    CTAS creates a Delta table stored in OneLake. The data is immediately available to all other Fabric workloads — Spark notebooks, semantic models, and pipelines — without any additional export or copy step.

---

## Writing Data

### INSERT

```sql
INSERT INTO gold.dim_customer (customer_key, customer_id, customer_name, region)
SELECT
    ROW_NUMBER() OVER (ORDER BY customer_id) AS customer_key,
    customer_id,
    customer_name,
    region
FROM silver_lakehouse.silver.customers;
```

### UPDATE

```sql
UPDATE gold.dim_customer
SET
    region       = s.region,
    updated_date = GETDATE()
FROM gold.dim_customer t
INNER JOIN silver_lakehouse.silver.customers s ON t.customer_id = s.customer_id
WHERE t.region <> s.region;
```

### DELETE

```sql
DELETE FROM gold.dim_customer
WHERE customer_id NOT IN (
    SELECT DISTINCT customer_id
    FROM silver_lakehouse.silver.orders
    WHERE order_date >= DATEADD(YEAR, -2, GETDATE())
);
```

### MERGE (upsert)

```sql
MERGE INTO gold.dim_customer AS target
USING silver_lakehouse.silver.customers AS source
ON target.customer_id = source.customer_id

WHEN MATCHED AND (
    target.customer_name <> source.customer_name OR
    target.region        <> source.region
) THEN UPDATE SET
    target.customer_name = source.customer_name,
    target.region        = source.region,
    target.updated_date  = GETDATE()

WHEN NOT MATCHED BY TARGET THEN INSERT (
    customer_id, customer_name, region, created_date, updated_date
) VALUES (
    source.customer_id, source.customer_name, source.region,
    GETDATE(), GETDATE()
);
```

---

## Cross-Database Queries

One of the most powerful features of the Fabric Warehouse is the ability to query tables across Lakehouses and Warehouses in the same workspace using three-part naming — no linked servers, no data movement.

```sql
-- Query a Lakehouse table from the Warehouse
SELECT *
FROM silver_lakehouse.silver.orders
WHERE order_date >= '2024-01-01';

-- Join across a Lakehouse and the Warehouse
SELECT
    o.order_id,
    o.amount,
    c.customer_name,
    c.region
FROM silver_lakehouse.silver.orders o
LEFT JOIN gold.dim_customer c ON o.customer_id = c.customer_id;
```

!!! tip
    Use cross-database queries to build Gold layer tables in the Warehouse from Silver tables in the Lakehouse — without copying data. The Warehouse reads directly from OneLake regardless of which Fabric item owns the table.

---

## Stored Procedures

Stored procedures allow you to encapsulate transformation logic, control flow, and error handling in reusable T-SQL routines — callable from pipelines or ad-hoc.

```sql
CREATE PROCEDURE gold.usp_load_fact_sales
    @load_date DATE = NULL
AS
BEGIN
    SET NOCOUNT ON;

    IF @load_date IS NULL
        SET @load_date = DATEADD(DAY, -1, CAST(GETDATE() AS DATE));

    -- Delete existing records for the load date (idempotent)
    DELETE FROM gold.fact_sales
    WHERE order_date = @load_date;

    -- Insert fresh data
    INSERT INTO gold.fact_sales (order_id, customer_id, product_id, order_date, revenue, cogs, gross_profit)
    SELECT
        o.order_id,
        o.customer_id,
        o.product_id,
        o.order_date,
        o.amount,
        p.cost,
        o.amount - p.cost
    FROM silver_lakehouse.silver.orders o
    LEFT JOIN silver_lakehouse.silver.products p ON o.product_id = p.product_id
    WHERE o.order_date = @load_date;
END;
```

Call it from a pipeline using the Stored Procedure activity:

```sql
EXEC gold.usp_load_fact_sales @load_date = '2024-01-15';
```

---

## Views

Views provide a semantic layer on top of raw tables — useful for simplifying complex joins, standardizing column names, or restricting visible columns for downstream consumers.

```sql
CREATE VIEW gold.vw_sales_summary AS
SELECT
    c.customer_name,
    c.region,
    c.segment,
    YEAR(f.order_date)  AS order_year,
    MONTH(f.order_date) AS order_month,
    SUM(f.revenue)      AS total_revenue,
    SUM(f.gross_profit) AS total_profit,
    COUNT(f.order_id)   AS order_count
FROM gold.fact_sales f
LEFT JOIN gold.dim_customer c ON f.customer_id = c.customer_id
GROUP BY
    c.customer_name, c.region, c.segment,
    YEAR(f.order_date), MONTH(f.order_date);
```

---

## Schemas

Organize tables by domain or function using schemas — keeps the Warehouse navigable as it grows:

```sql
CREATE SCHEMA gold;     -- Business-ready tables for BI consumption
CREATE SCHEMA staging;  -- Intermediate tables used during load
CREATE SCHEMA control;  -- Watermarks, audit logs, pipeline metadata
```

---

## Common Query Patterns

### Running totals

```sql
SELECT
    order_date,
    revenue,
    SUM(revenue) OVER (ORDER BY order_date ROWS UNBOUNDED PRECEDING) AS running_total
FROM gold.fact_sales;
```

### Year over year comparison

```sql
SELECT
    order_year,
    total_revenue,
    LAG(total_revenue) OVER (ORDER BY order_year)                        AS prev_year_revenue,
    total_revenue - LAG(total_revenue) OVER (ORDER BY order_year)        AS yoy_change
FROM (
    SELECT
        YEAR(order_date)  AS order_year,
        SUM(revenue)      AS total_revenue
    FROM gold.fact_sales
    GROUP BY YEAR(order_date)
) yearly;
```

### Top N per group

```sql
SELECT *
FROM (
    SELECT
        region,
        customer_name,
        SUM(revenue) AS total_revenue,
        RANK() OVER (PARTITION BY region ORDER BY SUM(revenue) DESC) AS rnk
    FROM gold.fact_sales f
    LEFT JOIN gold.dim_customer c ON f.customer_id = c.customer_id
    GROUP BY region, customer_name
) ranked
WHERE rnk <= 5;
```

→ See [SQL Patterns](../sql/patterns-top-n-per-group.md) for more query patterns applicable to the Warehouse.

---

## Connecting Power BI

The Fabric Warehouse is the ideal source for Power BI semantic models in large deployments. Three connection options:

| Mode | How it works | Best for |
|------|-------------|---------|
| **Direct Lake** | Semantic model reads Delta files directly from OneLake — no query sent to the Warehouse | Best performance, always fresh |
| **Direct Query** | Semantic model sends T-SQL queries to the Warehouse at query time | Always live, adds latency per visual |
| **Import** | Data loaded into the semantic model's in-memory engine at refresh time | Best report performance, requires scheduled refresh |

For most production deployments, Direct Lake is the recommended path — it combines the freshness of Direct Query with near-Import performance.

→ See [Direct Lake](direct-lake.md) for setup and performance considerations.

---

## Best Practices

- Use the Warehouse for the Gold layer — structured, conformed, business-ready data
- Use CTAS to build Gold tables from Silver sources — simple, fast, and creates Delta tables automatically
- Use MERGE for dimension tables — handles inserts and updates in a single statement
- Use schemas to organize tables — `gold`, `staging`, `control` at minimum
- Write idempotent stored procedures — delete and reload for the target date so re-runs are safe
- Use cross-database queries to read from Lakehouses — avoid copying Silver data into the Warehouse
- Create views for frequently used joins — simplifies semantic model setup and DAX measures
- Never use the Warehouse as a staging area for raw data — that belongs in the Lakehouse Files section
