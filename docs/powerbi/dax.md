---
title: DAX Fundamentals
description: Core concepts, measures, calculated columns, variables, and CALCULATE
tags: [powerbi, dax]
---

# DAX Fundamentals

Data Analysis Expressions (DAX) is the formula language used in Power BI to create measures, calculated columns, and calculated tables. Mastering DAX is the most important skill to develop after data modeling — most report errors are not visual or structural, they are DAX context errors that produce numbers that look right but are not.

---

## Key Concepts

### Filter Context

The set of filters active on the model when a measure is evaluated. Every visual, slicer, row in a matrix, and page filter contributes to the filter context. The same measure returns different results in different cells of a table — not because the formula changes, but because the filter context changes.

```dax
-- This measure always sums the same column
-- but returns a different number in every row of a report
-- because each row filters the model to a different product, date, or region
Total Sales = SUM(fact_sales[amount])
```

### Row Context

The current row being evaluated in a calculated column or inside an iterator function. Think of it as a loop — DAX processes one row at a time, and row context tells it which row it is on.

```dax
-- Row context: DAX evaluates this expression for each row individually
-- [first_name] and [last_name] refer to the values in the current row
Full Name = dim_customer[first_name] & " " & dim_customer[last_name]
```

Row context exists in calculated columns and iterator functions. It does not exist in regular measures — measures only have filter context.

### Context Transition

When a measure is called inside a calculated column or an iterator function, DAX automatically converts the current row context into an equivalent filter context. This is called context transition.

```dax
-- Inside SUMX, each row has a row context for fact_sales
-- Calling [Unit Margin] (a measure) inside SUMX triggers context transition:
-- the row context is converted to a filter that isolates that specific row
Total Margin =
SUMX(
    fact_sales,
    [Unit Margin]   -- measure called inside iterator = context transition
)
```

!!! warning
    Context transition can produce unexpected results if you are not aware it is happening. A measure called inside a calculated column will filter the entire table down to the current row before evaluating — this is often much slower and less intuitive than writing the expression directly. Always be deliberate about when you call measures inside iterators.

---

## Measures vs Calculated Columns

Both are DAX expressions but they live in different places and serve different purposes.

| | Calculated Column | Measure |
|--|-------------------|---------|
| Evaluated | At refresh time | At query time |
| Stored in model | ✅ Yes | ❌ No |
| Increases model size | ✅ Yes | ❌ No |
| Context available | Row context | Filter context |
| Best for | Fixed attributes, segmentation, filtering | Aggregations, KPIs, dynamic calculations |

```dax
-- Measure — evaluated at query time in the current filter context
Total Sales = SUM(fact_sales[amount])

-- Calculated column — evaluated row by row at refresh, stored in the table
Full Name = dim_customer[first_name] & " " & dim_customer[last_name]
```

!!! tip
    Use measures for anything that aggregates or responds to report filters. Use calculated columns only for fixed attributes that need to appear as a column — such as a customer segment label or a concatenated key — and that cannot be computed upstream in Power Query.

---

## Calculated Tables

A calculated table is a table created entirely from a DAX expression. It is evaluated at refresh time and stored in the model like any other table.

Common uses:

- **Measure tables** — empty tables used purely to organize measures in the field list
- **Date tables** — generated with `CALENDAR()` or `CALENDARAUTO()`
- **Disconnected tables** — lookup tables for slicers not directly related to fact data (e.g. a "Select metric" slicer)
- **Filtered copies** — a subset of another table for a specific analytical purpose

```dax
-- Disconnected table for a dynamic metric slicer
Metric Selector =
DATATABLE(
    "Metric Name", STRING,
    "Metric Key",  STRING,
    {
        { "Total Sales",   "sales"   },
        { "Total Orders",  "orders"  },
        { "Avg Ticket",    "ticket"  }
    }
)
```

```dax
-- Filtered copy of a table
High Value Customers =
FILTER(
    dim_customer,
    dim_customer[lifetime_value] >= 10000
)
```

!!! note
    Calculated tables increase model size and are recalculated at every refresh. Use them deliberately — if the table can be produced in Power Query, that is usually the better option.

---

## Variables

Variables are one of the most important features in DAX. They store the result of an expression and allow you to reference it multiple times without recalculating it. This improves both readability and performance.

### Syntax

```dax
Measure Name =
VAR variable_one = <expression>
VAR variable_two = <expression>
RETURN
    <final expression using variables>
```

Variables are evaluated once when the measure runs. The `RETURN` clause defines what the measure actually outputs.

### Avoiding double evaluation

Without variables, referencing the same expression twice forces DAX to calculate it twice:

```dax
-- Without variables — the DIVIDE denominator is calculated twice
-- because [Sales PY] appears in both the numerator and the denominator
Sales YoY % =
DIVIDE(
    SUM(fact_sales[amount]) - CALCULATE(SUM(fact_sales[amount]), SAMEPERIODLASTYEAR(dim_date[date])),
    CALCULATE(SUM(fact_sales[amount]), SAMEPERIODLASTYEAR(dim_date[date]))
)
```

```dax
-- With variables — previous year is calculated once and reused twice
Sales YoY % =
VAR current_sales  = SUM(fact_sales[amount])
VAR previous_sales = CALCULATE(SUM(fact_sales[amount]), SAMEPERIODLASTYEAR(dim_date[date]))
RETURN
    DIVIDE(current_sales - previous_sales, previous_sales)
```

### Simplifying conditional logic

Variables make nested `IF` expressions far easier to read and debug:

```dax
Sales Performance =
VAR target        = [Sales Target]
VAR actual        = [Total Sales]
VAR variance      = actual - target
VAR variance_pct  = DIVIDE(variance, target)
RETURN
    IF(variance_pct >= 0.05, "Above Target",
    IF(variance_pct >= 0,    "On Target",
                             "Below Target"))
```

### Debugging with variables

You can temporarily return a variable instead of the final expression to inspect intermediate values while building a measure:

```dax
Sales YoY % =
VAR current_sales  = [Total Sales]
VAR previous_sales = [Sales PY]
RETURN
    previous_sales  -- temporarily return this to verify the value
```

!!! tip
    Always use variables in any measure longer than one line. They make complex logic readable, prevent double calculation, and make debugging significantly easier.

---

## CALCULATE

`CALCULATE` is the most important function in DAX. It evaluates an expression in a modified filter context — it can add filters, remove filters, replace filters, or override relationships.

```dax
CALCULATE(<expression>, <filter1>, <filter2>, ...)
```

### Adding a filter

```dax
-- Total sales for Electronics only, regardless of any category filter in the report
Sales Electronics =
CALCULATE(
    [Total Sales],
    dim_product[category] = "Electronics"
)
```

### Removing filters

```dax
-- Total sales ignoring any date filter applied in the report
Sales All Time =
CALCULATE(
    [Total Sales],
    REMOVEFILTERS(dim_date)
)

-- Total sales as a % of all products (removes product filter only)
Sales % of Total =
DIVIDE(
    [Total Sales],
    CALCULATE([Total Sales], REMOVEFILTERS(dim_product))
)
```

### Combining filters

```dax
-- Sales for Electronics in 2024 only
Sales Electronics 2024 =
CALCULATE(
    [Total Sales],
    dim_product[category] = "Electronics",
    dim_date[year] = 2024
)
```

### CALCULATE Modifiers

| Modifier | Description |
|----------|-------------|
| `ALL(table)` | Removes all filters from a table |
| `ALL(table[column])` | Removes filters from a specific column |
| `ALLEXCEPT(table, col1, col2)` | Removes all filters except the specified columns |
| `REMOVEFILTERS(table/column)` | Cleaner alternative to `ALL()` for removing filters |
| `KEEPFILTERS(filter)` | Adds a filter without replacing the existing one on that column |
| `USERELATIONSHIP(col1, col2)` | Activates an inactive relationship for this evaluation |
| `CROSSFILTER(col1, col2, direction)` | Changes cross-filter direction for this evaluation |

!!! warning
    Avoid using `FILTER()` as a `CALCULATE` argument when a simple boolean condition will do. `FILTER()` iterates the entire table row by row — `CALCULATE([Total Sales], dim_product[category] = "Electronics")` is far more efficient than `CALCULATE([Total Sales], FILTER(dim_product, dim_product[category] = "Electronics"))`. Reserve `FILTER()` for cases where you genuinely need to iterate, such as when filtering on a measure result.

---

## Error Handling

DAX provides several functions for handling blanks, errors, and missing values gracefully.

### BLANK and ISBLANK

`BLANK()` is DAX's equivalent of null. Many aggregation functions return `BLANK()` when there is no data — for example, `SUM()` on an empty filter context returns `BLANK()`, not zero.

```dax
-- Check if a measure has no data
Has Sales = NOT ISBLANK([Total Sales])

-- Return zero instead of blank
Total Sales Zero = IF(ISBLANK([Total Sales]), 0, [Total Sales])
```

!!! note
    In most visuals, `BLANK()` causes the row or data point to be hidden entirely. This is usually desirable — but if you need zeros to appear explicitly (for charts, tables, or conditional formatting), convert blanks to zero with `IF(ISBLANK(...), 0, ...)` or by using `+0` at the end of the measure.

### IFERROR

Returns an alternate value if the expression produces an error:

```dax
-- Return 0 if the calculation fails for any reason
Safe Ratio = IFERROR(DIVIDE([Numerator], [Denominator]), 0)
```

!!! warning
    `IFERROR` swallows all errors silently — including ones caused by bugs in your formula. Use it sparingly and only when you are certain about what kind of error can occur. Prefer `DIVIDE()` over `IFERROR(x / y, 0)` for division — it is more explicit and performant.

### COALESCE

Returns the first non-blank value from a list of expressions — useful for fallback logic:

```dax
-- Use regional price if available, otherwise fall back to standard price
Effective Price =
COALESCE(
    [Regional Price],
    [Standard Price],
    0
)
```

---

## Best Practices

- Prefer measures over calculated columns for any aggregation or dynamic logic
- Always use `DIVIDE()` instead of `/` — never divide without a safe denominator
- Use variables in any measure longer than one line — readability and performance both improve
- Name measures clearly and consistently — `Total Sales`, `Sales YTD`, `Margin %`
- Organize measures in a dedicated empty table — one measure table per subject area if the model is large
- Break complex measures into intermediate measures rather than nesting deeply
- Avoid `FILTER()` inside `CALCULATE` when a simple column filter will do
- Do not use `IFERROR` to mask unknown errors — fix the formula instead
- Understand filter context and row context before writing anything beyond basic aggregations — most DAX bugs are context bugs