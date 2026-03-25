---
title: Fundamentals
description: Core concepts to start building reports in Power BI
tags: [powerbi, fundamentals]
---

# Fundamentals

What Power BI is, how a project is structured, and what each layer of the platform does — before you write a single measure or build a single visual.

---

## What is Power BI?

Power BI is a business intelligence platform developed by Microsoft. It allows you to connect to data sources, transform and model data, and create interactive reports and dashboards.

It has three main components:

- **Power BI Desktop** — the development environment where you build reports
- **Power BI Service** — the cloud platform where you publish and share reports
- **Power BI Mobile** — the app for consuming reports on mobile devices

!!! note
    Power BI Mobile is rarely used in practice. Most business users consume reports directly through Power BI Service in a browser. Mobile is worth knowing exists, but it is not a focus for most developers or organizations.

---

## What is Microsoft Fabric?

Microsoft Fabric is Microsoft's unified analytics platform, introduced in 2023. It brings together Power BI, Data Engineering, Data Science, Data Warehousing, Real-Time Intelligence, and Data Factory under a single product and a single capacity model.

Power BI is a core component of Fabric — everything you build in Power BI works within Fabric. The difference is that Fabric extends the platform beyond reporting:

- **Without Fabric** — Power BI connects to external data sources, transforms data in Power Query, and publishes reports to the Service
- **With Fabric** — the data itself lives inside the platform (OneLake), pipelines and lakehouses are built natively, and Power BI reports on top of that data without moving it

For most Power BI developers, Fabric is primarily relevant in two ways: the **licensing model** (F SKUs replace the old Premium P SKUs) and **Git integration** (native version control for semantic models and reports). The core report development workflow remains the same.

!!! note
    You do not need Fabric to use Power BI. Pro and PPU licenses work independently of Fabric capacity. Fabric becomes relevant when the organization needs the broader data platform — lakehouses, pipelines, notebooks — or wants to take advantage of Premium-level features at scale. See [Licensing](licensing.md) for the full picture.

---

## The Development Workflow

A typical Power BI project follows this sequence. Each step maps to a specific layer of the platform — understanding that mapping makes it much easier to know where a problem lives and where to look when something goes wrong.

### 🔌 Power Query — Connect & Transform

- **Connect** to a data source (SQL Server, Excel, API, SharePoint, etc.)
- **Transform** the data — clean columns, filter rows, fix data types, combine tables

Power Query runs before data reaches your model. Every transformation is recorded as a step, making the process transparent and repeatable. It uses the M language under the hood, though most work is done through the UI.

→ See [Power Query](power-query.md) for transformation patterns and M language basics.

---

### 🗄️ Data Model — Structure & Relate

- **Model** the data — define tables, relationships, hierarchies, and calculated columns

The data model is the foundation everything else builds on. A well-designed model makes DAX simpler, reports faster, and maintenance easier. The recommended pattern is the star schema: fact tables at the center, dimension tables around them.

→ See [Data Modeling](data-modeling.md) for star schema, relationships, and date table setup.

---

### 🧮 DAX — Define Business Logic

- **Write measures** using DAX — aggregations, KPIs, time intelligence, dynamic titles

DAX (Data Analysis Expressions) is the formula language used to create measures and calculated columns. Measures are evaluated at query time in the current filter context, making them flexible and reusable across any visual.

→ See [DAX](dax.md) for measures, calculated columns, and common patterns.

---

### 📊 Power BI Desktop — Build the Report

- **Build the report** — visuals, filters, slicers, bookmarks, drill throughs, tooltips

This is where data becomes a report. Power BI Desktop includes a large library of built-in visuals. You can also import custom visuals from AppSource. Good report design communicates clearly and requires minimal explanation from the developer.

→ See [Report Design](report-design.md) for layout principles, bookmarks, and themes.

---

### ☁️ Power BI Service — Publish & Share

- **Publish** to Power BI Service — upload the report to a shared workspace
- **Share** via workspaces, apps, or embedding — make it accessible to the right people

Power BI Service is the cloud platform where reports are hosted, refreshed, and distributed. Workspaces are used for collaboration; Apps are used for end-user distribution.

→ See [Deployment & Service](deployment.md) for workspaces, gateways, pipelines, and CI/CD.

---

## Connection Modes

How Power BI connects to your data source determines performance, refresh behavior, and which DAX features are available. Choose the right mode before you start building — switching later is costly.

| Mode | Data in PBI | Refresh | DAX support | Best for |
|------|-------------|---------|-------------|----------|
| **Import** | ✅ In-memory (VertiPaq) | Scheduled | Full | Most reports — best performance |
| **Direct Query** | ❌ Queried live | Always live | Limited | Real-time requirements |
| **Live Connection** | ❌ Queried live | Always live | Limited | Connecting to SSAS or Fabric semantic models |
| **Composite Model** | ✅ / ❌ Mixed | Mixed | Partial | Combining Import tables with Direct Query sources |

!!! tip
    Start with Import mode. It gives you the best performance and full DAX support. Switch to Direct Query only when real-time data is a hard requirement.

!!! warning
    Direct Query shifts query load to your source database. Every visual interaction triggers a query. On large datasets or slow sources, this makes reports sluggish and can impact other workloads running on the same database.

!!! note
    Composite Model allows mixing Import and Direct Query tables in the same model — useful when most data can be imported but one source must stay live. It adds complexity and requires careful testing.

---

## Power BI Desktop Views

Power BI Desktop is organized into distinct views, accessible from the left-side panel. Each view surfaces a different layer of your project — the same report file, seen from a different angle.

| View | Purpose |
|------|---------|
| **Report View** | Build and arrange visuals, slicers, pages, and bookmarks. |
| **Table View** | Inspect the data loaded into each table and validate calculated columns. |
| **Model View** | Manage relationships, cardinality, and cross-filter direction. |
| **DAX View** | Write and run DAX queries directly against the model. |
| **TMDL View** | Edit the semantic model as code — bulk changes and advanced properties. |

!!! note
    DAX View (formally DAX Query View) became generally available in 2024. It lets you write `EVALUATE` queries to test expressions directly — similar to DAX Studio but without leaving Desktop.

!!! note
    TMDL View was introduced in preview in January 2025 and reached general availability in September 2025. It is the most developer-oriented view in Desktop — think of it as a code editor for your semantic model. It is not needed for day-to-day report building, but becomes valuable for advanced modeling, bulk edits, and sharing reusable model objects like date tables or calculation groups.

---

## Best Practices

- Choose Import mode by default — only deviate when there is a specific, documented reason
- Design the data model before building visuals — a poor model cannot be fixed with DAX
- Keep transformation logic in Power Query or upstream SQL — not in DAX calculated columns
- Publish to shared workspaces, never to My Workspace, for anything used by others
- Learn DAX context (filter context and row context) early — it explains most unexpected behavior