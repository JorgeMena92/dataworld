---
title: DAX
description: Measures, calculated columns, and common DAX patterns
tags: [powerbi, dax]
---

# DAX

Data Analysis Expressions (DAX) is the formula language used in Power BI to create measures, calculated columns, and calculated tables. Mastering DAX is the most important skill to develop after data modeling.

---

## Key Concepts

### Filter Context
The set of filters currently applied to the model when a measure is evaluated. Every visual, slicer, and row in a table creates a filter context.

### Row Context
The current row being evaluated in a calculated column or iterator function. Think of it as a loop over each row.

### Context Transition
When a measure is called inside a calculated column or iterator, row context is converted into an equivalent filter context. This is one of the most important and confusing behaviors in DAX.

---

## Measures vs Calculated Columns

Always prefer measures over calculated columns for aggregations. Measures are evaluated in the current filter context, making them flexible and reusable across any visual.

```dax
-- Measure (evaluated at query time, no storage cost)
Total Sales = SUM(fact_sales[amount])

-- Calculated column (evaluated at refresh, stored in model)
Full Name = dim_customer[first_name] & " " & dim_customer[last_name]
```

---

## Common Patterns

### Basic Aggregations

```dax
Total Sales     = SUM(fact_sales[amount])
Total Orders    = COUNTROWS(fact_sales)
Average Ticket  = DIVIDE([Total Sales], [Total Orders])
Unique Customers = DISTINCTCOUNT(fact_sales[customer_id])
```

### Time Intelligence

```dax
-- Year to date
Sales YTD = TOTALYTD([Total Sales], dim_date[date])

-- Previous year
Sales PY = CALCULATE([Total Sales], SAMEPERIODLASTYEAR(dim_date[date]))

-- Year over year growth
Sales YoY % =
DIVIDE(
    [Total Sales] - [Sales PY],
    [Sales PY]
)

-- Month to date
Sales MTD = TOTALMTD([Total Sales], dim_date[date])
```

### CALCULATE

`CALCULATE` is the most important function in DAX. It evaluates an expression in a modified filter context.

```dax
-- Sales for a specific category
Sales Electronics =
CALCULATE(
    [Total Sales],
    dim_product[category] = "Electronics"
)

-- Sales ignoring date filter
Sales All Time =
CALCULATE(
    [Total Sales],
    REMOVEFILTERS(dim_date)
)
```

### DIVIDE

Always use `DIVIDE` instead of `/` to handle division by zero gracefully.

```dax
-- Safe division
Margin % = DIVIDE([Gross Profit], [Total Sales], 0)
```

### Ranking

```dax
-- Rank products by sales
Product Rank =
RANKX(
    ALL(dim_product[product_name]),
    [Total Sales],
    ,
    DESC,
    DENSE
)
```

### Dynamic Titles

```dax
-- Dynamic title based on slicer selection
Report Title =
"Sales Report — " & SELECTEDVALUE(dim_date[year], "All Years")
```

---

## CALCULATE Modifiers

| Modifier | Description |
|---|---|
| `FILTER()` | Adds a filter to the context |
| `ALL()` | Removes filters from a table or column |
| `ALLEXCEPT()` | Removes all filters except specified columns |
| `KEEPFILTERS()` | Keeps existing filters instead of replacing them |
| `REMOVEFILTERS()` | Removes filters (cleaner syntax than ALL in some cases) |

---

## Best Practices

- Always use `DIVIDE` instead of `/`
- Name measures clearly — `Total Sales`, `Sales YTD`, `Margin %`
- Organize measures in a dedicated empty table (measure table)
- Avoid deeply nested DAX — break complex logic into intermediate measures
- Use variables (`VAR`) to improve readability and performance

```dax
-- Using variables for clarity
Sales YoY % =
VAR current_sales = [Total Sales]
VAR previous_sales = [Sales PY]
RETURN
    DIVIDE(current_sales - previous_sales, previous_sales)
```
