---
title: Fundamentals
description: Core concepts to start building reports in Power BI
tags: [powerbi, fundamentals]
---

# Fundamentals

Core concepts every Power BI developer should understand before building reports and data models.

---

## What is Power BI?

Power BI is a business intelligence platform developed by Microsoft. It allows you to connect to data sources, transform and model data, and create interactive reports and dashboards.

It has three main components:

- **Power BI Desktop** — the development environment where you build reports
- **Power BI Service** — the cloud platform where you publish and share reports
- **Power BI Mobile** — the app for consuming reports on mobile devices

---

## The Development Workflow

A typical Power BI project follows this sequence:

1. **Connect** to a data source (SQL Server, Excel, API, etc.)
2. **Transform** the data using Power Query
3. **Model** the data — define tables, relationships, and hierarchies
4. **Write measures** using DAX
5. **Build the report** — visuals, filters, bookmarks, drill throughs
6. **Publish** to Power BI Service
7. **Share** via workspaces, apps, or embedding

---

## Key Concepts

### Data Sources
Power BI can connect to hundreds of sources — databases, files, cloud services, APIs, and more. The two main connection modes are:

- **Import** — data is loaded into Power BI's in-memory engine (VertiPaq). Faster performance, requires scheduled refresh.
- **Direct Query** — queries are sent live to the source. Always up to date, but slower and more limited in DAX.

### Power Query
The transformation layer. Before data reaches your model, Power Query lets you clean, reshape, filter, and combine it. Uses M language under the hood.

### Data Model
The set of tables and relationships that form the foundation of your report. A well-designed model makes DAX simpler and reports faster.

### DAX
Data Analysis Expressions — the formula language used to create measures and calculated columns. The most important skill to develop in Power BI after modeling.

### Visuals
Power BI includes a large library of built-in visuals. You can also import custom visuals from AppSource or build your own.

---

## Import vs Direct Query vs Live Connection

| Mode | Data stored in PBI | Refresh needed | DAX support | Best for |
|---|---|---|---|---|
| Import | ✅ Yes | ✅ Scheduled | Full | Most reports |
| Direct Query | ❌ No | ❌ Always live | Limited | Real-time data |
| Live Connection | ❌ No | ❌ Always live | Limited | SSAS / Fabric models |

---

!!! tip
    Start with Import mode. It gives you the best performance and full DAX support. Switch to Direct Query only when real-time data is a hard requirement.
