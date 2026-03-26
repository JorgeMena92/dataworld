---
title: Real-Time Intelligence
description: Eventstream, KQL databases, and real-time analytics patterns in Microsoft Fabric
tags: [fabric, real-time, kql, eventstream, rti]
---

# Real-Time Intelligence

Real-Time Intelligence (RTI) is the Microsoft Fabric workload for streaming data — ingesting, storing, querying, and acting on data as it arrives. It is built for high-velocity, time-series data: IoT sensor readings, application logs, clickstreams, financial ticks, and telemetry.

---

## What is Real-Time Intelligence?

Real-Time Intelligence consists of three core components that work together:

| Component | What it does |
|-----------|-------------|
| **Eventstream** | Ingest and route streaming data — no-code connectors, in-flight transformations, fan-out to multiple destinations |
| **Eventhouse / KQL Database** | Store and query time-series data using Kusto Query Language — optimized for high-volume, append-only, time-ordered data |
| **Real-Time Dashboard** | Visualize streaming data in auto-refreshing dashboards driven by KQL queries |

Together they cover the full streaming pipeline: ingest → store → analyze → visualize.

---

## Eventstream

Eventstream is the ingestion and routing layer. It connects to streaming sources, optionally transforms data in-flight, and fans out to one or more destinations.

### Sources

- Azure Event Hubs, Azure IoT Hub
- Kafka (Azure HDInsight, Confluent)
- Azure Blob Storage (new file trigger)
- Custom endpoint (REST)
- Sample data (built-in for development)

### Destinations

- **Eventhouse (KQL Database)** — primary destination for real-time analytics
- **Lakehouse** — land streaming data as Delta files for batch processing
- **Warehouse** — stream directly into a Fabric Warehouse table
- **Derived stream** — branch the stream for a separate consumer

### In-flight transformations

Eventstream supports real-time transformations before data reaches the destination — no Spark or separate processing job needed:

- **Filter** — drop events that don't match a condition
- **Aggregate** — tumbling, hopping, or session windows
- **Join** — enrich events with a reference table
- **Manage fields** — add, remove, or rename columns

```
IoT Hub
   │
   ▼
Eventstream
   ├── Filter (temperature > 80)
   ├── Aggregate (avg per device, 1-min tumbling window)
   └── Fan-out
        ├── Eventhouse  (real-time analytics)
        └── Lakehouse   (raw archive for batch)
```

---

## Eventhouse and KQL Database

An **Eventhouse** is the container for one or more **KQL Databases**. Each KQL Database stores tables, functions, and materialized views — all optimized for time-series queries.

### When to use KQL over SQL

| Scenario | KQL | SQL |
|----------|-----|-----|
| Time-series queries (last N minutes, rolling windows) | ✅ Natural syntax | ❌ Verbose |
| High-velocity append-only data (logs, telemetry) | ✅ Optimized | ⚠️ Possible but costly |
| Ad-hoc exploration of large datasets | ✅ Fast | ✅ |
| Complex joins across many tables | ⚠️ | ✅ Natural |
| BI reporting with star schema | ❌ | ✅ |

Use RTI and KQL for streaming and time-series workloads. Use the Warehouse for structured Gold layer data and BI consumption.

---

## KQL Basics

KQL uses a pipe-based syntax — each step filters or transforms the result of the previous step. It reads naturally left-to-right and is optimized for time-series data.

### Basic structure

```kql
TableName
| where condition
| summarize aggregation by grouping
| order by column desc
| take N
```

### Filtering

```kql
// Events in the last hour
device_telemetry
| where ingestion_time() > ago(1h)

// Specific device and threshold
device_telemetry
| where device_id == "sensor-42"
    and temperature > 80

// Time range
device_telemetry
| where timestamp between (datetime(2024-01-15) .. datetime(2024-01-16))
```

### Aggregations

```kql
// Average temperature per device in the last hour
device_telemetry
| where ingestion_time() > ago(1h)
| summarize avg_temp = avg(temperature) by device_id
| order by avg_temp desc

// Count events per minute
device_telemetry
| where ingestion_time() > ago(1h)
| summarize event_count = count() by bin(timestamp, 1m)
| order by timestamp asc
```

### Time series

```kql
// Rolling average over 5-minute buckets
device_telemetry
| where ingestion_time() > ago(24h)
| summarize avg_temp = avg(temperature) by bin(timestamp, 5m), device_id
| order by timestamp asc

// Anomaly detection using built-in ML function
device_telemetry
| where ingestion_time() > ago(7d)
| summarize avg_temp = avg(temperature) by bin(timestamp, 1h)
| evaluate anomaly_detection(avg_temp, timestamp, 0.95)
```

### Joins

```kql
// Enrich telemetry with device metadata
device_telemetry
| where ingestion_time() > ago(1h)
| join kind=leftouter (
    device_metadata
    | project device_id, device_name, location, device_type
) on device_id
| project timestamp, device_name, location, temperature, humidity
```

### Render (visualization)

```kql
// Renders a time chart directly in query results
device_telemetry
| where ingestion_time() > ago(6h)
| summarize avg_temp = avg(temperature) by bin(timestamp, 5m), device_id
| render timechart
```

---

## KQL Tables

### Creating a table

```kql
.create table device_telemetry (
    timestamp:   datetime,
    device_id:   string,
    temperature: real,
    humidity:    real,
    pressure:    real,
    status:      string
)
```

### Retention policy

Control how long data is kept — hot cache (fast, in-memory) and cold storage (slower, cheaper):

```kql
// Keep 7 days in hot cache, 365 days total
.alter table device_telemetry policy caching hot = 7d

.alter table device_telemetry policy retention
'{"SoftDeletePeriod": "365.00:00:00", "Recoverability": "Enabled"}'
```

!!! tip
    Set the hot cache period based on your actual dashboard query range. If dashboards only look at the last 7 days, there is no benefit to caching 30 days — and it costs more. Keep cold storage longer for historical analysis.

---

## Materialized Views

Materialized views pre-aggregate data continuously as new records arrive — a live summary table that is always up to date. Use them to avoid re-aggregating large raw tables on every query.

```kql
.create materialized-view with (backfill=true)
    device_telemetry_hourly on table device_telemetry
{
    device_telemetry
    | summarize
        avg_temp     = avg(temperature),
        avg_humidity = avg(humidity),
        event_count  = count()
      by bin(timestamp, 1h), device_id
}
```

Query the materialized view like a table:

```kql
device_telemetry_hourly
| where timestamp > ago(7d)
| order by timestamp desc
```

!!! tip
    Use materialized views for any aggregation that Power BI or dashboards query repeatedly. They eliminate the cost of re-scanning the full raw table on every dashboard refresh.

---

## Real-Time Dashboards

Real-Time Dashboards auto-refresh on a configurable interval — as fast as every 30 seconds — without user interaction.

### Setup

1. Create a new Real-Time Dashboard in Fabric
2. Connect it to a KQL Database
3. Add tiles — each tile is a KQL query with a visualization type
4. Set the auto-refresh interval
5. Add parameters (date pickers, dropdowns) for interactivity

### Common tile types

| Type | Best for |
|------|---------|
| **Time chart** | Trends over time — temperature, error rates, throughput |
| **Stat card** | Single KPI — current value, count, latest reading |
| **Bar chart** | Comparison across categories |
| **Table** | Raw event stream, recent records |
| **Map** | Geo-distributed data — device locations, regional metrics |

---

## OneLake Availability

KQL tables can expose their data to OneLake as Delta files — making streaming data accessible to Spark notebooks, pipelines, and semantic models without copying.

Enable it per table:

```kql
.alter table device_telemetry policy mirroring enabled
```

Once enabled, the table appears in OneLake and can be referenced from a Lakehouse shortcut or read directly by Spark.

!!! note
    OneLake availability adds a small latency — data typically appears within a few minutes of ingestion. For near-real-time queries, always query the KQL Database directly. Use OneLake availability for batch processing, historical analysis, and Power BI Import mode semantic models.

---

## Medallion Architecture for Streaming

Real-Time Intelligence supports the medallion pattern for streaming data:

| Layer | Storage | What happens |
|-------|---------|-------------|
| **Bronze** | Raw KQL table or Eventstream → Lakehouse | Raw events as they arrive, no transformation |
| **Silver** | KQL materialized view or Lakehouse Delta table | Deduplicated, validated, enriched |
| **Gold** | KQL materialized view or Warehouse | Aggregated, business-ready metrics |

A common pattern is to land raw events in both the KQL Database (for real-time queries) and the Lakehouse (for batch history), then use the Warehouse for structured Gold layer data consumed by Power BI.

---

## Best Practices

- Use Eventstream for all streaming ingestion — handles fan-out, transformation, and error recovery without code
- Apply in-flight filters in Eventstream to drop irrelevant events before storage — reduces cost and improves query performance
- Use materialized views for any aggregation queried frequently — avoid re-scanning raw tables on every dashboard refresh
- Set hot cache based on actual query patterns — keeping 7 days hot covers most operational dashboards
- Enable OneLake availability for tables that need to be joined with batch data or consumed by Power BI Import mode
- Use `bin(timestamp, Xm)` for time bucketing — cleaner and more efficient than manual timestamp truncation
- Use `ago()` for relative time filters — dashboards stay relevant without maintenance
- Keep Real-Time Dashboards focused — one dashboard per operational domain, not one dashboard for everything
