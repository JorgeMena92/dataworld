---
title: Power Query
description: Data transformation and ETL patterns using M language
tags: [powerbi, powerquery, etl]
---

# Power Query

Power Query is the data transformation layer in Power BI. It runs before the data reaches your model, allowing you to clean, reshape, and combine data from any source.

---

## The ETL Pattern

Power Query follows an Extract, Transform, Load pattern:

1. **Extract** — connect to a data source (database, file, API, etc.)
2. **Transform** — apply steps to clean and reshape the data
3. **Load** — send the result to the Power BI data model

Every transformation is recorded as a step in the **Applied Steps** panel. This makes the process transparent, repeatable, and easy to audit.

---

## Common Transformations

### Remove and rename columns

```
Remove unnecessary columns early — reduces model size and improves performance.
Rename columns to match your naming convention before loading.
```

### Filter rows

```
Filter out nulls, test data, cancelled records, or date ranges
that are out of scope for the report.
```

### Change data types

```
Always set the correct data type for each column:
- Dates → Date or DateTime
- Numeric → Decimal or Whole Number
- Text → Text
- True/False → Logical
```

### Split and merge columns

```
Split: "John Smith" → "John" + "Smith"
Merge: "John" + "Smith" → "John Smith"
Merge with separator: City + ", " + Country
```

### Group By

Aggregate data before loading — reduces row count and improves performance when summary data is sufficient.

### Pivot and Unpivot

- **Pivot** — turn row values into columns (wide format)
- **Unpivot** — turn columns into rows (long format, better for modeling)

---

## Combining Queries

### Merge (JOIN)
Combines two tables based on matching columns — equivalent to SQL JOIN.

```
Merge types:
- Left Outer  → all rows from left, matching from right
- Inner       → only matching rows from both
- Full Outer  → all rows from both
- Left Anti   → rows in left with no match in right
```

### Append (UNION)
Stacks two or more tables with the same structure on top of each other — equivalent to SQL UNION ALL.

Use append to combine monthly files, regional datasets, or historical and current data.

---

## Parameters

Parameters make queries dynamic and reusable. Common uses:

- Switch between development and production data sources
- Control date range filters
- Set file paths for folder-based data sources

---

## M Language Basics

Power Query steps are written in M behind the scenes. You rarely need to write M directly, but knowing the basics helps when the UI is not enough.

```m
-- Each step in a query is a variable
let
    Source = Sql.Database("server", "database"),
    dbo_sales = Source{[Schema="dbo", Item="sales"]}[Data],
    filtered = Table.SelectRows(dbo_sales, each [status] = "completed"),
    renamed = Table.RenameColumns(filtered, {{"amount", "total_amount"}})
in
    renamed
```

---

## Best Practices

- Filter and remove columns as early as possible in the query steps
- Disable **Enable Load** on staging/intermediate queries — only load final tables to the model
- Use parameters for source paths and environment switching
- Avoid using Power Query for complex business logic — do that in DAX or upstream in SQL
- Prefer database-side transformations (SQL views, stored procedures) over Power Query when working with large datasets
- Document each query with a description in the query properties
