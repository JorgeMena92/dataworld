---
title: Lakehouse
description: Delta Lake storage, SQL analytics endpoint, shortcuts, and table management in Microsoft Fabric
tags: [fabric, lakehouse, delta, onelake]
---

# Lakehouse

A Lakehouse in Microsoft Fabric is the primary storage and compute unit for data engineering workloads. It combines the flexibility of a data lake (files in any format) with the structure of a data warehouse (Delta tables with SQL access) — all stored in OneLake and accessible by every Fabric workload without copying data.

---

## What is a Lakehouse?

A Lakehouse is a Fabric item that provisions two storage locations in OneLake automatically:

- **Tables** — managed Delta tables. Registered in the metastore, queryable via SQL and Spark.
- **Files** — unmanaged storage for raw files in any format (CSV, Parquet, JSON, images, etc.). Not queryable directly via SQL.

Every Lakehouse also comes with a built-in **SQL analytics endpoint** — a read-only T-SQL interface over the Delta tables, usable from any SQL tool without any additional setup.

```
Lakehouse
├── Tables/          ← Delta tables (managed, SQL-queryable)
│   ├── orders
│   ├── customers
│   └── products
└── Files/           ← Raw files (unmanaged, any format)
    ├── raw/
    │   └── orders_2024_01.csv
    └── archive/
```

---

## Delta Lake

All tables in a Fabric Lakehouse are stored in Delta Lake format — an open storage layer built on Parquet files with a transaction log.

Delta Lake delivers:

- **ACID transactions** — reads and writes are consistent even under concurrent access
- **Time travel** — query historical versions of a table using `VERSION AS OF` or `TIMESTAMP AS OF`
- **Schema enforcement** — writes that don't match the table schema are rejected
- **Schema evolution** — controlled addition of new columns without breaking existing queries
- **Optimize and vacuum** — compact small files and remove old versions to maintain performance

```python
# Time travel in PySpark
df = spark.read.format("delta") \
    .option("versionAsOf", 5) \
    .load("Tables/orders")

# Or using SQL
spark.sql("SELECT * FROM orders VERSION AS OF 5")
```

!!! note
    Delta Lake is the default and only supported table format in Fabric Lakehouses. You cannot create non-Delta tables in the managed Tables section — all tables are Delta, whether created via Spark, SQL, or dataflow.

---

## SQL Analytics Endpoint

Every Lakehouse has a built-in SQL analytics endpoint — a T-SQL read-only interface that exposes all Delta tables as SQL views. It requires no setup and is always available.

Use it to:
- Query Lakehouse tables from SQL Server Management Studio, Azure Data Studio, or any JDBC/ODBC tool
- Connect Power BI in Direct Query mode without a semantic model
- Run cross-lakehouse SQL queries using three-part naming

```sql
-- Query a lakehouse table via SQL analytics endpoint
SELECT
    customer_id,
    SUM(amount) AS total_sales
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY customer_id
ORDER BY total_sales DESC;
```

!!! warning
    The SQL analytics endpoint is **read-only**. You cannot INSERT, UPDATE, DELETE, or CREATE TABLE through it. All writes must go through Spark or the Lakehouse UI. Use a Fabric Warehouse if you need full DML support via T-SQL.

---

## Shortcuts

Shortcuts are virtual links to data stored outside the Lakehouse — in other OneLake locations, ADLS Gen2, Amazon S3, Google Cloud Storage, or Dataverse. They appear as folders or tables inside the Lakehouse without copying or moving the data.

```
Lakehouse
├── Tables/
│   ├── orders          ← managed Delta table
│   └── external_ref    ← shortcut to another lakehouse
└── Files/
    ├── raw/
    └── s3_data/        ← shortcut to Amazon S3 bucket
```

### When to use shortcuts

| Scenario | Use shortcut |
|----------|-------------|
| Data already exists in ADLS Gen2 | ✅ Avoid duplication |
| Sharing a table between two lakehouses | ✅ Single source of truth |
| Reading from S3 or GCS without moving data | ✅ Zero-copy access |
| Data needs transformation before use | ❌ Ingest and transform instead |

!!! tip
    Use shortcuts for Bronze layer ingestion when the source already lands data in ADLS Gen2 or S3. This avoids a full copy into OneLake and keeps the raw data in its original location while making it accessible to all Fabric workloads.

---

## Schemas

Lakehouses support schemas — a logical grouping of tables within a single Lakehouse, similar to schemas in a SQL database. This allows you to organize tables by domain or medallion layer within one Lakehouse instead of creating separate Lakehouses per layer.

```
Lakehouse (with schemas enabled)
├── bronze/
│   ├── raw_orders
│   └── raw_customers
├── silver/
│   ├── orders
│   └── customers
└── gold/
    ├── fact_sales
    └── dim_customer
```

Enable schemas when creating the Lakehouse — it cannot be enabled after creation.

!!! note
    Schemas in Lakehouses are a relatively recent feature. Older Lakehouses without schemas enabled have all tables in a flat namespace. For new projects, always enable schemas — it makes the medallion architecture much cleaner to implement in a single Lakehouse.

---

## Loading Data

### From files (Spark)

```python
# Read a CSV from the Files section and write as a Delta table
df = spark.read.option("header", True).csv("Files/raw/orders_2024_01.csv")
df.write.format("delta").mode("overwrite").saveAsTable("bronze.raw_orders")
```

### Append vs overwrite

```python
# Overwrite — full reload
df.write.format("delta").mode("overwrite").saveAsTable("silver.orders")

# Append — incremental load
df.write.format("delta").mode("append").saveAsTable("silver.orders")

# Merge — upsert pattern (see Delta merge below)
```

### Delta merge (upsert)

```python
from delta.tables import DeltaTable

target = DeltaTable.forName(spark, "silver.orders")

target.alias("t").merge(
    source=df.alias("s"),
    condition="t.order_id = s.order_id"
).whenMatchedUpdateAll() \
 .whenNotMatchedInsertAll() \
 .execute()
```

---

## Table Maintenance

Delta tables accumulate small files over time, especially with incremental writes. Regular maintenance keeps query performance healthy.

### Optimize

Compacts small Parquet files into larger ones — improves read performance significantly on large tables:

```python
spark.sql("OPTIMIZE silver.orders")

# With Z-ordering — co-locates related data for faster filtered reads
spark.sql("OPTIMIZE silver.orders ZORDER BY (customer_id, order_date)")
```

### Vacuum

Removes old file versions that are no longer needed — reclaims storage:

```python
# Default retention is 7 days — do not reduce below 7 days
spark.sql("VACUUM silver.orders")

# Check what would be removed before running
spark.sql("VACUUM silver.orders DRY RUN")
```

!!! warning
    Never set VACUUM retention below 7 days. Delta Lake needs those old files to guarantee read consistency for long-running queries. Setting retention to 0 can cause active queries to fail.

### V-Order

Fabric applies V-Order optimization automatically when writing Delta tables — a write-time optimization that improves read performance across all Fabric engines (Spark, T-SQL, Direct Lake). It is enabled by default and does not require manual configuration.

---

## Lakehouse vs Warehouse

Both store data in OneLake as Delta tables, but they serve different purposes:

| | Lakehouse | Warehouse |
|--|-----------|-----------|
| Primary language | PySpark / SQL | T-SQL |
| Write access via SQL | ❌ Read-only endpoint | ✅ Full DML |
| Unstructured files | ✅ Files section | ❌ Tables only |
| Best for | Ingestion, transformation, Bronze/Silver | Serving layer, Gold, BI consumption |
| Spark access | ✅ Native | ✅ Via shortcut |

→ See [Warehouse](warehouse.md) for T-SQL warehouse patterns and when to use one over a Lakehouse.

---

## Best Practices

- Enable schemas when creating a Lakehouse — organize tables by medallion layer (`bronze`, `silver`, `gold`)
- Use shortcuts for external data sources instead of copying data into Fabric
- Always write Delta tables in the Tables section — avoid writing raw files to Files and treating them as tables
- Run `OPTIMIZE` regularly on large, frequently updated tables — schedule it after bulk loads
- Run `VACUUM` weekly on active tables — keeps storage costs under control
- Use Z-ordering on columns that are frequently used in WHERE filters or JOIN conditions
- Use the SQL analytics endpoint for BI tools and ad-hoc SQL — reserve Spark for transformation workloads
- Never reduce VACUUM retention below 7 days
