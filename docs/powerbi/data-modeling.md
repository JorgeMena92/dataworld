---
title: Data Modeling
description: Star schema, relationships, and semantic model best practices
tags: [powerbi, modeling]
---

# Data Modeling

A well-designed data model is the foundation of every good Power BI report. It makes DAX simpler, reports faster, and maintenance easier.

---

## The Star Schema

The star schema is the recommended modeling pattern for Power BI. It consists of:

- **Fact tables** — contain measurable events (sales, orders, transactions). Large, with many rows.
- **Dimension tables** — contain descriptive attributes (customers, products, dates). Smaller, with fewer rows.

```
Date ──────────────┐
                   │
Product ───────── Fact Sales ───── Customer
                   │
Store ─────────────┘
```

!!! tip
    Always aim for a star schema. Avoid flat tables and snowflake schemas in Power BI — they complicate DAX and hurt performance.

---

## Relationships

Relationships connect tables so filters can flow between them.

### Cardinality
- **One-to-many (1:*)** — the standard. One row in the dimension matches many rows in the fact table.
- **Many-to-many (*:*)** — use with caution. Can cause ambiguity and unexpected results.
- **One-to-one (1:1)** — rare. Usually means the tables can be merged.

### Cross-filter direction
- **Single** — filters flow from the one side to the many side (dimension → fact). Default and recommended.
- **Both** — filters flow in both directions. Use sparingly — can cause circular dependencies and slow performance.

---

## The Date Table

Every Power BI model that uses time intelligence (YTD, MTD, previous year) requires a proper date table.

Requirements:
- Must contain a continuous range of dates with no gaps
- Must be marked as a date table in Power BI
- Must have a relationship to every fact table that contains dates

```dax
-- Minimum date table columns
Date
Year
Quarter
Month Number
Month Name
Week Number
Day of Week
Is Weekday
```

!!! tip
    Create your date table using DAX with `CALENDAR()` or `CALENDARAUTO()`, or load one from Power Query. Never use the auto date/time feature in production models.

---

## Calculated Columns vs Measures

| | Calculated Column | Measure |
|---|---|---|
| Evaluated | At refresh time | At query time |
| Stored in model | ✅ Yes | ❌ No |
| Uses row context | ✅ Yes | ❌ No |
| Uses filter context | ❌ No | ✅ Yes |
| Best for | Segmentation, filtering | Aggregations, KPIs |

!!! tip
    Prefer measures over calculated columns whenever possible. Measures are more flexible, don't increase model size, and are evaluated in the correct filter context.

---

## Best Practices

- Hide all foreign keys and technical columns from report view
- Use consistent naming conventions — `dim_`, `fact_`, snake_case or PascalCase
- Keep fact tables narrow — only IDs and measures
- Avoid calculated columns for aggregations — use measures instead
- Always create a date table manually
- Disable auto date/time in settings
