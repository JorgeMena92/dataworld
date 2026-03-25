---
title: DAX Iterators
description: Iterator functions, row-by-row evaluation, and RANKX patterns
tags: [powerbi, dax, iterators]
---

# DAX Iterators

Iterator functions evaluate an expression row by row over a table and then aggregate the results. They are one of the most powerful tools in DAX — they let you perform calculations that simple aggregations cannot express, such as multiplying two columns before summing, or ranking dynamically across a filtered set.

---

## How Iterators Work

A standard aggregation like `SUM` operates on a single column:

```dax
-- Sums the amount column directly
Total Sales = SUM(fact_sales[amount])
```

An iterator like `SUMX` takes a table and an expression, evaluates the expression for each row, and then aggregates the results:

```dax
-- For each row: multiply quantity by unit_price, then sum all results
Total Sales =
SUMX(
    fact_sales,
    fact_sales[quantity] * fact_sales[unit_price]
)
```

The key difference is that iterators give you **row context** — access to the values of each row as the expression is evaluated. This makes them essential when the calculation requires combining values from multiple columns in the same row.

!!! note
    Every iterator function follows the same pattern: `FUNCTIONX(table, expression)`. The table defines what rows to iterate over, and the expression is evaluated once per row with full row context.

---

## Common Iterator Functions

### SUMX

Evaluates an expression for each row and returns the sum. The most commonly used iterator.

```dax
-- Revenue calculated at row level (quantity × price), then summed
Total Revenue =
SUMX(
    fact_sales,
    fact_sales[quantity] * fact_sales[unit_price]
)

-- Gross profit: revenue minus cost, calculated per row
Total Gross Profit =
SUMX(
    fact_sales,
    (fact_sales[quantity] * fact_sales[unit_price]) -
    (fact_sales[quantity] * fact_sales[unit_cost])
)
```

### AVERAGEX

Evaluates an expression for each row and returns the average.

```dax
-- Average order value (not average of the amount column, but per-order total)
Avg Order Value =
AVERAGEX(
    VALUES(fact_sales[order_id]),
    CALCULATE(SUM(fact_sales[amount]))
)
```

### MAXX / MINX

Returns the maximum or minimum result of an expression evaluated across rows.

```dax
-- Most expensive single transaction
Max Transaction =
MAXX(
    fact_sales,
    fact_sales[quantity] * fact_sales[unit_price]
)

-- Earliest order date per customer
First Order Date =
MINX(
    VALUES(fact_sales[customer_id]),
    CALCULATE(MIN(fact_sales[order_date]))
)
```

### COUNTX

Counts the number of rows where the expression returns a non-blank value.

```dax
-- Count rows where margin is positive
Profitable Transactions =
COUNTX(
    fact_sales,
    IF(
        fact_sales[unit_price] > fact_sales[unit_cost],
        1,
        BLANK()
    )
)
```

### FILTER

`FILTER` iterates a table and returns only the rows where the condition is true. It is used to build filtered tables for other functions — not as a standalone measure.

```dax
-- High value transactions only (used as input to another function)
FILTER(
    fact_sales,
    fact_sales[amount] >= 1000
)

-- Common usage: iterate only over a filtered subset
High Value Revenue =
SUMX(
    FILTER(fact_sales, fact_sales[amount] >= 1000),
    fact_sales[amount]
)
```

!!! warning
    `FILTER` iterates the full table row by row. On large tables this is expensive — avoid wrapping large fact tables in `FILTER` inside `CALCULATE` when a simple column filter will do. See [DAX Fundamentals](dax.md) for the correct use of `CALCULATE` filters.

---

## RANKX

`RANKX` is a specialized iterator that ranks a value relative to all values in a table. It is one of the more complex functions in DAX because it evaluates in two passes.

```dax
RANKX(table, expression, [value], [order], [ties])
```

| Parameter | Description |
|-----------|-------------|
| `table` | The set of rows to rank against |
| `expression` | The value to rank |
| `value` | The value to rank (defaults to the expression result in current context) |
| `order` | `ASC` or `DESC` (default: `DESC`) |
| `ties` | `SKIP` (default) or `DENSE` |

### Basic ranking

```dax
-- Rank each product by total sales, highest first
Product Sales Rank =
RANKX(
    ALL(dim_product[product_name]),
    [Total Sales],
    ,
    DESC,
    DENSE
)
```

`ALL(dim_product[product_name])` removes any product filter so the rank is computed across all products, not just the one in the current filter context.

### Ranking within a group

```dax
-- Rank products within their category
Product Rank in Category =
RANKX(
    ALLSELECTED(dim_product[product_name]),
    [Total Sales],
    ,
    DESC,
    DENSE
)
```

`ALLSELECTED` ranks within whatever is currently visible in the report — respecting slicers but ignoring the row-level product filter.

### Using rank to filter Top N

```dax
-- Return sales only for the top 5 products, blank otherwise
Top 5 Sales =
IF(
    [Product Sales Rank] <= 5,
    [Total Sales]
)
```

!!! tip
    Use `DENSE` ranking when you want sequential ranks with no gaps after ties (1, 2, 2, 3). Use `SKIP` (default) when you want standard competition ranking (1, 2, 2, 4). For most business reports, `DENSE` is the more intuitive choice.

---

## Iterators vs Aggregations — When to Use Each

| Situation | Use |
|-----------|-----|
| Sum a single column | `SUM()` |
| Multiply two columns then sum | `SUMX()` |
| Average of a calculated expression | `AVERAGEX()` |
| Max/min of a calculated expression | `MAXX()` / `MINX()` |
| Rank against a dynamic set | `RANKX()` |
| Filter rows before aggregating | `FILTER()` inside `SUMX` or similar |

!!! warning
    Iterators are more expensive than simple aggregations — they process every row in the table for every cell in the visual. Avoid iterating over large fact tables when a simpler aggregation is sufficient. If performance is a concern, consider whether the row-level calculation can be pushed upstream into Power Query or the source SQL.

---

## Best Practices

- Use `SUMX` when a calculation needs to happen at row level before aggregation — never fake it with a calculated column just to then `SUM` it
- Always be explicit about which table you are iterating — avoid using the full fact table when `VALUES()` or `FILTER()` can narrow it down
- Prefer `DENSE` over `SKIP` for `RANKX` in business reports — gaps in rankings confuse end users
- Combine `RANKX` with `IF` to build Top N visuals without needing report-level filters
- Be aware that every iterator creates row context — calling measures inside an iterator triggers context transition