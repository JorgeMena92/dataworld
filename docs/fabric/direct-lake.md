---
title: Direct Lake
description: Direct Lake mode, semantic models on OneLake, and performance considerations in Microsoft Fabric
tags: [fabric, direct-lake, power-bi, semantic-model]
---

# Direct Lake

Direct Lake is a Power BI connection mode exclusive to Microsoft Fabric. It allows semantic models to read Delta files directly from OneLake — combining the freshness of Direct Query with the performance of Import mode, without scheduled data refreshes or data duplication.

---

## What is Direct Lake?

In traditional Power BI connection modes there is a trade-off between freshness and performance:

| Mode | Data freshness | Query performance | Refresh needed |
|------|---------------|------------------|---------------|
| **Import** | Snapshot at last refresh | ✅ Fastest — in-memory | ✅ Scheduled |
| **Direct Query** | Always live | ❌ Slowest — every visual fires a query | ❌ None |
| **Direct Lake** | Near real-time — reads latest Delta files | ✅ Fast — columnar reads | ❌ None |

Direct Lake eliminates the refresh cycle entirely. When a user opens a report, the semantic model reads the current Delta Parquet files from OneLake directly — no import, no query sent to a database. As soon as new data is written to the Lakehouse or Warehouse, it is available in the report.

---

## How Direct Lake Works

Direct Lake uses the same VertiPaq in-memory engine as Import mode — but instead of loading all data into memory upfront at refresh time, it loads column segments on demand as queries arrive.

```
User opens report
       │
       ▼
Power BI evaluates DAX measure
       │
       ▼
VertiPaq checks if column segment is in memory
       │
       ├── Cache hit  → returns result immediately (Import-like speed)
       └── Cache miss → reads column from Delta Parquet in OneLake
                        → loads into memory → returns result
```

This on-demand loading is called **transcoding** — converting Parquet column data into the VertiPaq columnar format. Once loaded, the segment stays in memory and subsequent queries against the same column are as fast as Import mode.

!!! note
    The first query against a cold semantic model will be slower as column segments load from OneLake. After warm-up, performance is comparable to Import mode. This is the main trade-off — no pre-loading at refresh time means the first user pays the loading cost.

---

## Prerequisites

Direct Lake has specific requirements:

- The workspace must be on a **Fabric capacity** (F SKU or Premium P SKU) — Direct Lake does not work on Pro or PPU workspaces
- The data source must be a **Lakehouse or Warehouse in the same workspace** — external sources are not supported
- Tables must be stored as **Delta format** — the default for all Fabric Lakehouses and Warehouses
- The semantic model must use **Direct Lake connection mode** — set when creating or editing the model

---

## Creating a Direct Lake Semantic Model

### From a Lakehouse

Every Lakehouse automatically generates a default semantic model containing all its Delta tables. To create a custom model:

1. Open the Lakehouse in Fabric
2. Select **New semantic model** from the ribbon
3. Choose the tables to include
4. The model is created in Direct Lake mode automatically

### From a Warehouse

1. Open the Warehouse in Fabric
2. Select **New semantic model**
3. Choose tables and views to include

### Editing the model

Open the semantic model in the Fabric web editor or connect Power BI Desktop via **Connect to a published dataset**. All DAX measures, relationships, hierarchies, and formatting are configured here — the same as any Power BI semantic model.

---

## Framing

**Framing** is the mechanism that controls which version of the Delta table the semantic model reads. When new data is written to a Lakehouse table, the Delta transaction log records a new version. The semantic model holds a reference to a specific version — the **frame** — and reads from it consistently.

```
Delta table versions:
v1 (baseline) → v2 (new batch loaded) → v3 (another batch)

Semantic model frame: v2
  └── All queries read from v2 until the frame is updated
```

### Automatic framing

By default, the semantic model updates its frame automatically within a few minutes of a write completing — this is what makes Direct Lake near-real-time without a manual refresh.

### Manual framing

Trigger a frame update explicitly via the Fabric REST API when you want to control exactly when new data becomes visible:

```python
import requests

requests.post(
    f"https://api.fabric.microsoft.com/v1/workspaces/{workspace_id}/semanticModels/{dataset_id}/refresh",
    headers={"Authorization": f"Bearer {token}"}
)
```

!!! note
    Framing is not a full Import refresh. It is a lightweight metadata operation that updates the pointer to the latest Delta version — it does not reload data into memory. Column segments are loaded on demand when queries arrive.

---

## Fallback to Direct Query

Direct Lake can fall back to Direct Query mode automatically when certain DAX patterns or data characteristics exceed what it supports natively. The report continues to work but performance degrades to Direct Query speed — silently.

Common fallback triggers:

- Measures using unsupported DAX functions (some statistical and financial functions)
- Tables exceeding the column or row limit for the SKU
- RLS patterns that require full table scans
- Calculated tables referencing complex expressions

### Detecting fallback

In Power BI Desktop, connect to the published Direct Lake model and use **Performance Analyzer** — visuals triggering fallback show a `Direct Query` label instead of `Storage Engine`.

!!! warning
    Fallback is silent — users will not see an error, but report performance will be noticeably slower. Always test with Performance Analyzer after building the model to identify and resolve fallback triggers before publishing to production.

---

## SKU Guardrails

Direct Lake imposes limits based on the Fabric capacity SKU. Exceeding these limits triggers fallback to Direct Query:

| SKU | Max rows per table | Max columns per table | Max tables |
|-----|-------------------|----------------------|-----------|
| F2 – F32 | 300M | 1,000 | 100 |
| F64 | 1.5B | 3,000 | 500 |
| F128+ | 3B+ | 4,000+ | 1,000+ |

!!! note
    Most production fact tables with standard column counts stay well within F64 limits. If you are on a smaller SKU and approaching limits, reduce the column count by hiding unused columns in the semantic model rather than removing them from the Lakehouse.

---

## Row-Level Security in Direct Lake

RLS works in Direct Lake but with an important difference from Import mode. Because Direct Lake reads from OneLake files, RLS filters are applied at query time — similar to Direct Query. This means:

- RLS works correctly and securely
- Heavy RLS patterns (many roles, complex DAX filters) can trigger fallback to Direct Query
- Test RLS performance explicitly — it is more expensive in Direct Lake than in Import mode

Define RLS roles in the semantic model the same way as Import mode — using DAX filter expressions on dimension tables.

→ See [Deployment & Service](../powerbi/deployment.md) for the full RLS setup walkthrough.

---

## Direct Lake vs Import — When to Use Each

| Scenario | Direct Lake | Import |
|----------|------------|--------|
| Data updates frequently (hourly or more) | ✅ | ❌ Refresh lag |
| Data is large (billions of rows) | ✅ | ❌ Memory cost |
| Reports need guaranteed fast cold start | ⚠️ First query slower | ✅ |
| Complex DAX with statistical functions | ⚠️ May fall back | ✅ Full support |
| Workspace on Pro or PPU | ❌ Not supported | ✅ |
| Data lives outside Fabric | ❌ | ✅ Via gateway |

!!! tip
    Direct Lake is the default recommendation for any semantic model built on Fabric data. Only fall back to Import when you need guaranteed sub-second cold start, full DAX function coverage, or data from sources outside Fabric.

---

## Best Practices

- Always verify the workspace is on a Fabric capacity before building a Direct Lake model — it will not work on Pro workspaces
- Hide unused columns from the semantic model — reduces column count, lowers fallback risk, and improves usability
- Use Performance Analyzer in Power BI Desktop to detect fallback triggers before publishing
- Avoid complex calculated columns in Direct Lake models — push calculations upstream to the Lakehouse or Warehouse
- Build Gold layer tables with a clean star schema before creating the semantic model — Direct Lake works best on well-modeled data
- Test RLS performance explicitly — complex filters are more expensive in Direct Lake than Import
- Monitor the Fabric Capacity Metrics app — Direct Lake transcoding consumes CUs and can impact other workloads on the same capacity
