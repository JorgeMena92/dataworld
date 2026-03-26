---
title: Pipelines
description: Data Factory pipelines, activities, orchestration, and scheduling patterns in Microsoft Fabric
tags: [fabric, pipelines, data-factory, orchestration]
---

# Pipelines

Pipelines in Microsoft Fabric are the orchestration layer for data movement and transformation. Built on Data Factory, they connect sources to destinations, sequence activities, handle errors, and schedule workloads — all without writing infrastructure code.

---

## What is a Pipeline?

A pipeline is a logical grouping of activities that together perform a data integration task. Activities execute sequentially or in parallel, pass outputs to downstream steps, and can branch based on success, failure, or custom conditions.

Common uses:

- Ingest data from external sources into a Lakehouse
- Orchestrate a sequence of notebook runs (Bronze → Silver → Gold)
- Copy data between Fabric items or from external systems
- Trigger refreshes of semantic models after data loads complete
- Schedule and monitor recurring data workflows

---

## Activities

Activities are the building blocks of a pipeline. Each activity performs one unit of work.

### Copy Data

The most commonly used activity — moves data from a source to a destination using one of 200+ native connectors.

| Source | Destination |
|--------|-------------|
| SQL Server, Azure SQL | Lakehouse (Delta table) |
| REST API | Lakehouse (Files section) |
| SharePoint List | Warehouse |
| Blob / ADLS Gen2 | Lakehouse (Delta table) |
| Salesforce, SAP | Lakehouse (Files section) |

Key settings:

- **Source** — connection, query or table, incremental filter
- **Sink** — destination, write behavior (overwrite, append, upsert)
- **Mapping** — column mapping between source and destination
- **Performance** — copy parallelism, batch size

### Notebook

Runs a Fabric notebook as a pipeline step. Use it to execute Spark transformations, run data quality checks, or trigger any Python or PySpark logic. Parameters can be passed from the pipeline to the notebook, making notebooks reusable across pipeline runs.

### Dataflow Gen2

Runs a Dataflow Gen2 (Power Query-based transformation) as a pipeline step. Useful for low-code transformations that feed into a Lakehouse or Warehouse.

### Stored Procedure

Executes a stored procedure in a Warehouse or SQL database. Use it for SQL-based transformations, post-load aggregations, or data quality rules written in T-SQL.

### Script

Runs an arbitrary SQL script against a Warehouse or SQL analytics endpoint. More flexible than Stored Procedure for one-off queries or dynamic SQL.

### Get Metadata

Retrieves metadata about a file or table — row count, last modified date, schema — and passes it to downstream activities. Useful for validating that a file arrived before attempting to process it.

### Delete

Deletes files or folders from a Lakehouse Files section or external storage. Use it to clean up staging files after successful ingestion.

### If Condition / Switch

Branches the pipeline based on a boolean expression or a value. Use it to handle different source formats, skip processing when no new data exists, or route records to different destinations.

### ForEach / Until

- **ForEach** — iterates over a list and runs a set of activities for each item. Use it to process multiple files, tables, or regions in a single pipeline run.
- **Until** — loops until a condition is met. Useful for polling patterns.

### Fail

Explicitly fails the pipeline with a custom error message. Use it at the end of a conditional branch to surface meaningful errors when unexpected conditions occur.

---

## Parameters

Parameters make pipelines reusable across different inputs without duplicating the pipeline definition.

Common parameters:

- `load_date` — the date to filter incremental data
- `source_table` — which table to load
- `environment` — dev / test / prod, used to switch connection strings
- `file_path` — path to the input file in the Files section

Reference parameters in activity settings using `@pipeline().parameters.parameter_name`.

---

## Expressions and Dynamic Content

Pipeline settings support dynamic expressions using the Data Factory expression language, written in `@` syntax. Expressions can reference parameters, variables, system functions, and activity outputs.

```
-- Today's date as a string
@formatDateTime(utcNow(), 'yyyy-MM-dd')

-- Parameter reference
@pipeline().parameters.source_table

-- Output from a previous activity
@activity('Get Row Count').output.firstRow.row_count

-- Conditional expression
@if(equals(pipeline().parameters.environment, 'prod'), 'prod-lakehouse', 'dev-lakehouse')
```

---

## Variables

Variables store intermediate values within a pipeline run. Unlike parameters (set at run time and immutable), variables can be updated during execution using the **Set Variable** activity.

Use variables to accumulate a count across a ForEach loop, store the output of a Get Metadata activity for use downstream, or track whether any errors occurred in a parallel branch.

---

## Error Handling

Every activity has three outcome paths — **Success**, **Failure**, and **Completion** (runs regardless of outcome). Connect activities to these paths to build error handling flows.

```
Copy Data ──(on success)──▶ Notebook (transform)
         └──(on failure)──▶ Fail activity (raise error with message)
```

!!! tip
    Always connect a Fail activity to critical steps. Without it, a failed Copy Data activity marks the pipeline as failed with no context. The Fail activity lets you add a meaningful error message that appears in the monitoring view.

### Retry policy

Each activity supports a configurable retry policy — number of retries and interval between them. Set retries on Copy Data activities that connect to external sources prone to transient failures such as throttling or timeouts.

---

## Scheduling and Triggers

### Scheduled trigger

Runs the pipeline on a recurring schedule — hourly, daily, weekly, or a custom cron expression:

```
-- Daily at 06:00 UTC
0 6 * * *

-- Every hour
0 * * * *

-- Monday to Friday at 07:00 UTC
0 7 * * 1-5
```

### Storage event trigger

Runs the pipeline when a file arrives in a storage location (Blob, ADLS Gen2, OneLake). Use it for event-driven ingestion — process a file as soon as it lands rather than polling on a schedule.

### Manual trigger

Runs the pipeline on demand from the UI or via the Fabric REST API. Useful for ad-hoc loads, backfills, and testing.

!!! note
    Fabric pipelines do not yet support tumbling window triggers, which guarantee exactly-once processing per time window. For strict watermark-based incremental loading, implement the watermark logic manually using a pipeline variable or a control table in the Lakehouse.

---

## Incremental Loading Pattern

The most common pipeline pattern — load only new or changed records since the last run.

### Watermark-based

Store the last loaded timestamp in a control table. On each run, read the watermark, load records newer than it, then update the watermark:

```
1. Read watermark from control table
        │
        ▼
2. Copy Data — filter source WHERE updated_at > @watermark
        │
        ▼
3. Update watermark to current run time
```

A simple control table in the Lakehouse Warehouse:

```sql
CREATE TABLE control.watermarks (
    table_name  VARCHAR(100),
    last_loaded_at DATETIME2
);
```

### Partition-based

For sources partitioned by date (files named `orders_2024_01_15.csv`), use a ForEach activity over a list of dates to process each partition independently — each iteration handles one file.

→ See [Lakehouse](lakehouse.md) for the Delta merge upsert pattern used to apply incremental loads into Delta tables.

---

## Monitoring

Every pipeline run is logged in the **Monitor** hub in Fabric. You can see run status, duration per activity, input and output for each activity, error messages, and re-run a failed pipeline from the point of failure.

!!! tip
    Use the Monitor hub during development to inspect activity inputs and outputs — it shows the exact query sent to the source, the number of rows read and written, and the full error message when something fails. This is faster than adding debug logging to notebooks.

---

## Best Practices

- Use parameters for all environment-specific values — never hardcode connection strings or file paths
- Always add a Fail activity to critical steps — silent failures are harder to diagnose than explicit ones
- Set retry policies on Copy Data activities connecting to external APIs or databases prone to throttling
- Use Get Metadata before processing files — validate the file exists and has rows before running a notebook
- Keep pipelines focused — one pipeline per logical workflow, not one pipeline for everything
- Name pipelines descriptively — `pl_ingest_orders_daily`, not `Pipeline1`
- Store watermarks in a control table — do not rely on pipeline run history for incremental logic
- Monitor run duration over time — a pipeline that suddenly takes 3× longer usually means data volume grew or query folding broke
