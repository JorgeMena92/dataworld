---
title: Notebooks
description: Spark notebooks, PySpark patterns, and Lakehouse integration in Microsoft Fabric
tags: [fabric, notebooks, spark, pyspark]
---

# Notebooks

Notebooks in Microsoft Fabric are the primary development environment for Spark-based data engineering and data science workloads. They run on Apache Spark, support Python, SQL, Scala, and R, and integrate natively with Lakehouses and other Fabric items — all within the browser, no local setup required.

---

## What is a Notebook?

A notebook is a Fabric item that combines code cells, markdown cells, and output in a single interactive document. Each notebook is backed by a Spark session — a distributed compute engine that can process data at any scale.

Key properties:

- **Multi-language** — switch between PySpark, SQL, Scala, and R within the same notebook using `%%` magic commands
- **Lakehouse attached** — a notebook can have a default Lakehouse, making its tables and files directly accessible without connection strings
- **Pipeline-ready** — notebooks can be run from pipelines with parameters passed at runtime
- **Versioned** — when the workspace is connected to Git, notebooks are stored as `.ipynb` files

---

## Attaching a Lakehouse

Attaching a Lakehouse to a notebook is the standard way to access data. Once attached, the Lakehouse tables and files are available in the left Explorer panel and directly referenceable in code.

```python
# Tables in the attached Lakehouse are available as Spark tables
df = spark.read.table("silver.orders")

# Files are accessible via the Files/ path
df = spark.read.csv("Files/raw/orders_2024_01.csv", header=True)
```

A notebook can have one default Lakehouse and additional Lakehouses mounted as secondary. Reference secondary Lakehouses using their full path:

```python
# Secondary Lakehouse
df = spark.read.table("another_lakehouse.silver.orders")
```

---

## PySpark Basics

PySpark is the Python API for Apache Spark — the most commonly used language in Fabric notebooks.

### Reading data

```python
# Read a Delta table
df = spark.read.table("silver.orders")

# Read a CSV file from the Files section
df = spark.read \
    .option("header", True) \
    .option("inferSchema", True) \
    .csv("Files/raw/orders_2024_01.csv")

# Read a Parquet file
df = spark.read.parquet("Files/raw/orders/")
```

### Transformations

```python
from pyspark.sql import functions as F

# Select, filter, rename
df = df.select("order_id", "customer_id", "amount", "order_date") \
       .filter(F.col("amount") > 0) \
       .withColumnRenamed("amount", "order_amount")

# Add calculated columns
df = df.withColumn("order_year",  F.year(F.col("order_date"))) \
       .withColumn("order_month", F.month(F.col("order_date")))

# Aggregation
summary = df.groupBy("customer_id") \
            .agg(
                F.sum("order_amount").alias("total_sales"),
                F.count("order_id").alias("order_count"),
                F.max("order_date").alias("last_order_date")
            )

# Join
customers = spark.read.table("silver.customers")
result = df.join(customers, on="customer_id", how="left")
```

### Writing data

```python
# Write as a Delta table (overwrite)
df.write.format("delta").mode("overwrite").saveAsTable("gold.fact_sales")

# Write as a Delta table (append)
df.write.format("delta").mode("append").saveAsTable("silver.orders")

# Write with partitioning
df.write \
  .format("delta") \
  .mode("overwrite") \
  .partitionBy("order_year", "order_month") \
  .saveAsTable("silver.orders_partitioned")
```

---

## SQL in Notebooks

SQL cells are the fastest way to query and explore data — no DataFrame API needed.

### The %%sql magic

```sql
%%sql
SELECT
    customer_id,
    SUM(order_amount) AS total_sales,
    COUNT(order_id)   AS order_count
FROM silver.orders
WHERE order_date >= '2024-01-01'
GROUP BY customer_id
ORDER BY total_sales DESC
LIMIT 20
```

### Mixing PySpark and SQL

```python
# Create a temp view from a DataFrame, then query it with SQL
df.createOrReplaceTempView("orders_temp")
```

```sql
%%sql
SELECT order_year, SUM(order_amount) AS revenue
FROM orders_temp
GROUP BY order_year
```

```python
# Capture SQL results back into a DataFrame
result = spark.sql("SELECT * FROM silver.orders WHERE amount > 1000")
```

---

## Notebook Parameters

Notebooks can accept parameters when triggered from a pipeline. Mark a cell as a **parameter cell** in the notebook UI (`...` menu on any cell → Toggle parameter cell) — this designates it as the cell where pipeline-injected values land.

```python
# Parameter cell — defaults used during interactive runs, overridden by pipeline at runtime
load_date    = "2024-01-01"
source_table = "orders"
environment  = "dev"
```

Pass values to these parameters via the Notebook activity settings in the pipeline under **Base parameters**.

!!! tip
    Always define sensible defaults in the parameter cell. This lets you run the notebook interactively during development without needing a pipeline to supply values.

---

## Lakehouse Integration Patterns

### Bronze → Silver transformation

```python
from pyspark.sql import functions as F
from pyspark.sql.types import DoubleType

# Read raw data from Bronze
raw = spark.read \
    .option("header", True) \
    .csv("Files/raw/orders/")

# Clean and conform
clean = raw \
    .withColumn("order_date", F.to_date(F.col("order_date"), "yyyy-MM-dd")) \
    .withColumn("amount", F.col("amount").cast(DoubleType())) \
    .filter(F.col("order_id").isNotNull()) \
    .dropDuplicates(["order_id"])

# Write to Silver
clean.write.format("delta").mode("overwrite").saveAsTable("silver.orders")
```

### Upsert (merge) pattern

```python
from delta.tables import DeltaTable

target = DeltaTable.forName(spark, "silver.orders")

target.alias("t").merge(
    source=clean.alias("s"),
    condition="t.order_id = s.order_id"
).whenMatchedUpdateAll() \
 .whenNotMatchedInsertAll() \
 .execute()
```

### Silver → Gold aggregation

```python
orders    = spark.read.table("silver.orders")
customers = spark.read.table("silver.customers")
products  = spark.read.table("silver.products")

fact_sales = orders \
    .join(customers, on="customer_id", how="left") \
    .join(products,  on="product_id",  how="left") \
    .select(
        "order_id",
        "customer_id",
        "product_id",
        "order_date",
        F.col("amount").alias("revenue"),
        F.col("cost").alias("cogs"),
        (F.col("amount") - F.col("cost")).alias("gross_profit")
    )

fact_sales.write.format("delta").mode("overwrite").saveAsTable("gold.fact_sales")
```

---

## Optimization Tips

### Cache intermediate DataFrames

```python
# Cache a DataFrame used multiple times in the same notebook
orders.cache()

# Release when done
orders.unpersist()
```

### Filter early

Operations like `groupBy()`, `join()`, and `distinct()` trigger a shuffle — data redistribution across Spark nodes. On large tables this is expensive. Filter and reduce data volume as early as possible before these operations.

### Repartition before writing

```python
# Repartition before writing to avoid many small files
df.repartition(8).write.format("delta").mode("overwrite").saveAsTable("silver.orders")
```

### Use display() not collect()

```python
# Good — streams results to the notebook UI efficiently
display(df)

# Avoid on large DataFrames — pulls all rows to the driver and can crash the session
df.collect()
```

---

## Scheduling Notebooks

Notebooks can be scheduled in two ways:

- **From a pipeline** — use the Notebook activity, pass parameters, chain with other activities, and handle errors centrally
- **Directly** — open the notebook, select **Run → Schedule**, set a recurring trigger

For production workloads, always run notebooks from a pipeline. It gives you dependency management, error handling, monitoring, and the ability to chain multiple notebooks in sequence.

!!! warning
    Scheduling a notebook directly bypasses pipeline error handling and monitoring. If the notebook fails, there is no retry logic and no downstream activity to surface the error. Use direct scheduling only for simple, standalone exploration scripts.

---

## Best Practices

- Attach a default Lakehouse to every notebook — avoid hardcoding paths or connection strings
- Use parameter cells for any value that changes between environments or pipeline runs
- Keep notebooks focused — one notebook per transformation layer, not one notebook for everything
- Name notebooks clearly — `nb_transform_orders_bronze_to_silver`, not `Notebook1`
- Prefer `saveAsTable` over writing to file paths — tables are registered in the metastore and immediately queryable via SQL
- Filter and reduce data volume early — before joins, groupBys, and other shuffle operations
- Use `display()` for exploration, never `collect()` on large DataFrames
- Always run production notebooks from pipelines, not direct schedules
- Use `%%sql` cells for quick exploration — faster to write than constructing DataFrames for simple queries
