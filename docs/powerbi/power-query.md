---
title: Power Query
description: Data transformation and ETL patterns using M language
tags: [powerbi, powerquery, etl]
---

# Power Query

Power Query is the data transformation layer in Power BI. It runs before data reaches your model, allowing you to clean, reshape, and combine data from any source. Every transformation is recorded as a step, making the process transparent, repeatable, and easy to audit.

---

## The ETL Pattern

Power Query follows an Extract, Transform, Load pattern:

- **Extract** — connect to a data source (database, file, API, etc.)
- **Transform** — apply steps to clean and reshape the data
- **Load** — send the result to the Power BI data model

Each step in the **Applied Steps** panel corresponds to one M expression under the hood. Steps execute in sequence — each one receives the output of the previous step as its input.

Each table in Power Query is its own query. You build it independently, name it, and choose whether to load it to the model or keep it as a staging step. This means a complex pipeline is just a chain of queries referencing each other, not one monolithic script.

Transformations can be applied in two ways — and both are always in sync:

- **User interface** — click-through operations in the ribbon and column menus. Power Query writes the M code for you automatically.
- **Advanced Editor** — the full M script behind the query, editable directly. Useful when the UI cannot express what you need.

!!! warning
    There is no undo history beyond the current session. Deleting or reordering steps in the Applied Steps panel is permanent once the file is saved. Work carefully when modifying existing queries — especially in production files.

!!! note
    Not all transformations are equal from a performance standpoint. When Power Query can translate your steps into native source queries (SQL, for example), it does so — this is called **query folding**. Steps that break folding force Power Query to download raw data and process it locally. See the Performance page for full coverage.

---

## Common Transformations

### Remove and rename columns

Remove unnecessary columns as early as possible in the step sequence — this reduces the volume of data processed in all subsequent steps. Rename columns to match your naming convention before loading to the model.

### Filter rows

Filter out nulls, test records, cancelled statuses, or date ranges out of scope. Like column removal, filtering early reduces the data volume carried through the rest of the query.

### Change data types

Always set the correct data type explicitly for every column. Power Query's auto-detection is unreliable, especially with dates and mixed-type columns.

| Source type | Target type in Power Query |
|-------------|---------------------------|
| Date strings | Date or DateTime |
| Numeric | Decimal Number or Whole Number |
| True / False | Logical |
| Free text | Text |
| Identifiers (IDs) | Text (not Whole Number — avoids aggregation) |

### Split and merge columns

Split a full name into first and last, or merge city and country into a single label:

```
Split:  "John Smith"        → "John"  +  "Smith"
Merge:  "John" + "Smith"    → "John Smith"
Merge:  "Lima" + "Peru"     → "Lima, Peru"
```

### Group By

Aggregate rows before loading — reduces row count and improves model performance when detail-level data is not needed in the report.

### Pivot and Unpivot

- **Pivot** — turn distinct row values into columns (wide format)
- **Unpivot** — turn columns into rows (long format, better for modeling)

Unpivot is particularly useful when source data arrives in a spreadsheet-style layout with one column per month or category. Long format is almost always the right shape for a Power BI model.

→ See [Pivot and Unpivot](../sql/patterns-pivot.md) in SQL Patterns for the underlying concept.

### Reference vs Duplicate

Two ways to branch a query — easy to confuse:

- **Duplicate** — creates an independent copy of the query. Changes to the original do not affect the duplicate.
- **Reference** — creates a new query that starts from the output of another. Changes upstream flow through automatically.

Use Reference to build a staging → final table pipeline. Use Duplicate when you need a fully independent branch.

!!! warning
    Referencing a query that has **Enable Load** disabled is fine for staging. But if you reference a query that loads to the model, both queries will load — which can cause duplication. Be explicit about which queries are staging-only.

---

## Combining Queries

### Merge (JOIN)

Combines two tables based on matching columns — equivalent to a SQL JOIN.

| Merge type | Equivalent | Result |
|------------|-----------|--------|
| Left Outer | `LEFT JOIN` | All rows from left, matching from right |
| Right Outer | `RIGHT JOIN` | All rows from right, matching from left |
| Full Outer | `FULL OUTER JOIN` | All rows from both |
| Inner | `INNER JOIN` | Only matching rows from both |
| Left Anti | `WHERE right IS NULL` | Rows in left with no match in right |
| Right Anti | `WHERE left IS NULL` | Rows in right with no match in left |

### Append (UNION)

Stacks two or more tables with the same structure on top of each other — equivalent to SQL `UNION ALL`. Use it to combine monthly export files, regional datasets, or historical and current data loaded from different sources.

!!! warning
    Column names and data types must match across all appended tables. If a column exists in one table but not another, Power Query will not raise an error — it will silently add the column to the result and fill the missing rows with `null`. Always verify column alignment before appending.

---

## Parameters

Parameters make queries dynamic and reusable without editing the M code directly. They appear as typed values you can change from the Power Query UI or from a parameter table driven by the data source.

Common uses:

- Switch between development and production connection strings
- Control a date cutoff for incremental-style filtering
- Set a file path for folder-based data sources

```m
// Parameter: ServerName (type = Text, current value = "prod-server")
// Parameter: DatabaseName (type = Text, current value = "sales_db")

let
    Source = Sql.Database(ServerName, DatabaseName),
    dbo_orders = Source{[Schema="dbo", Item="orders"]}[Data]
in
    dbo_orders
```

Changing `ServerName` from `"prod-server"` to `"dev-server"` in the parameter definition updates every query that references it — no need to edit each query individually.

!!! tip
    Store environment parameters (server, database, file path) in a dedicated parameters table loaded from a config file or SharePoint list. This makes environment switching a data change, not a code change.

---

## M Language

Power Query steps are written in M behind the scenes. Most work is done through the UI, but knowing M directly lets you handle cases the UI cannot express — custom columns with conditional logic, dynamic filtering, reusable functions, and more.

### The let…in structure

Every M query follows the same pattern:

```m
let
    Step1 = <expression>,
    Step2 = <expression using Step1>,
    Step3 = <expression using Step2>
in
    Step3
```

Each line is a named variable. The `in` clause declares which step is the final output. This maps directly to what you see in the Applied Steps panel.

### Custom columns

Add a calculated column using `Table.AddColumn`. The third argument is a function applied to each row — `each` is shorthand for `(_) =>`.

```m
// Add a full name column
Table.AddColumn(Source, "full_name", each [first_name] & " " & [last_name])

// Add a revenue column
Table.AddColumn(Source, "revenue", each [quantity] * [unit_price], type number)
```

### Conditional logic

Use `if / then / else` for row-level conditions:

```m
// Classify order size
Table.AddColumn(
    Source,
    "order_size",
    each
        if [amount] >= 10000 then "Large"
        else if [amount] >= 1000 then "Medium"
        else "Small",
    type text
)
```

### Common M functions

| Function | Description |
|----------|-------------|
| `Table.SelectRows(table, condition)` | Filter rows — equivalent to `WHERE` |
| `Table.SelectColumns(table, columns)` | Keep only specified columns |
| `Table.RenameColumns(table, renames)` | Rename columns by pair list |
| `Table.AddColumn(table, name, fn)` | Add a calculated column |
| `Table.Group(table, keys, aggs)` | Group and aggregate rows |
| `Table.NestedJoin(...)` | Merge two tables |
| `Text.Trim(text)` | Remove leading and trailing spaces |
| `Text.Upper(text)` / `Text.Lower(text)` | Change case |
| `Date.Year(date)` / `Date.Month(date)` | Extract date parts |
| `Number.Round(number, decimals)` | Round a number |
| `List.Distinct(list)` | Return unique values from a list |
| `Value.ReplaceType(value, type)` | Assign a type explicitly |

### Reusable functions

You can define a query as a function and call it from other queries — useful for applying the same transformation logic to multiple tables.

```m
// Function query: CleanTextColumn
// Takes a table and a column name, returns the table with that column trimmed and uppercased
(inputTable as table, columnName as text) as table =>
let
    trimmed  = Table.TransformColumns(inputTable, {{columnName, Text.Trim}}),
    uppercased = Table.TransformColumns(trimmed,  {{columnName, Text.Upper}})
in
    uppercased
```

Call it from another query:

```m
let
    Source   = Excel.Workbook(...),
    Cleaned  = CleanTextColumn(Source, "country")
in
    Cleaned
```

---

## Best Practices

- Filter rows and remove columns as early as possible — every step after them processes less data
- Disable **Enable Load** on staging and intermediate queries — only final tables should reach the model
- Use parameters for connection strings and file paths — never hardcode environment-specific values
- Prefer source-side transformations (SQL views, stored procedures) for large datasets — keep Power Query for light reshaping and type enforcement
- Avoid complex business logic in Power Query — calculated measures belong in DAX, aggregations belong upstream
- Name every query clearly — `stg_orders`, `dim_customer`, `fact_sales` — not `Query1` or `Sheet1`
- Document each query with a description in the query properties panel