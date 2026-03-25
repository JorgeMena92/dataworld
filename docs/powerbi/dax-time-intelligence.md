---
title: DAX Time Intelligence
description: YTD, MTD, previous periods, growth rates, and rolling calculations
tags: [powerbi, dax, time-intelligence]
---

# DAX Time Intelligence

Time intelligence functions in DAX allow you to compare and aggregate data across time periods — year to date, previous year, month over month growth, and rolling windows. They are among the most frequently used DAX patterns in business reports.

---

## Prerequisites

Time intelligence functions require a properly configured date table in the model:

- A continuous range of dates with no gaps
- Marked as a date table in Power BI Desktop (**Table tools → Mark as date table**)
- An active relationship to every fact table date column you want to analyze

Without these conditions, time intelligence functions will produce incorrect or blank results. See [Data Modeling](data-modeling.md) for date table setup.

---

## Year to Date

Accumulates a measure from the start of the year to the current date in context.

```dax
-- Standard YTD using the quick function
Sales YTD = TOTALYTD([Total Sales], dim_date[date])

-- Equivalent using CALCULATE + DATESYTD (more flexible)
Sales YTD =
CALCULATE(
    [Total Sales],
    DATESYTD(dim_date[date])
)
```

Use the `CALCULATE` + `DATESYTD` form when you need to add additional filters alongside the time intelligence filter, or when working with a fiscal year end date:

```dax
-- YTD for a fiscal year ending June 30
Sales Fiscal YTD =
CALCULATE(
    [Total Sales],
    DATESYTD(dim_date[date], "06-30")
)
```

---

## Quarter to Date and Month to Date

```dax
Sales QTD = TOTALQTD([Total Sales], dim_date[date])

Sales MTD = TOTALMTD([Total Sales], dim_date[date])
```

The same `CALCULATE` + `DATESQTD` / `DATESMTD` pattern applies when additional filters are needed.

---

## Previous Period Comparisons

### Previous year

```dax
Sales PY =
CALCULATE(
    [Total Sales],
    SAMEPERIODLASTYEAR(dim_date[date])
)
```

### Previous month

```dax
Sales PM =
CALCULATE(
    [Total Sales],
    DATEADD(dim_date[date], -1, MONTH)
)
```

### Previous quarter

```dax
Sales PQ =
CALCULATE(
    [Total Sales],
    DATEADD(dim_date[date], -1, QUARTER)
)
```

`DATEADD` is the most flexible function for period shifting — it works with `DAY`, `MONTH`, `QUARTER`, and `YEAR` and accepts negative values for going back in time.

---

## Growth Rates

Always use variables when building growth measures — it avoids evaluating the same measure twice and makes the formula easier to read and debug.

### Year over year

```dax
Sales YoY % =
VAR current_sales  = [Total Sales]
VAR previous_sales = [Sales PY]
RETURN
    DIVIDE(current_sales - previous_sales, previous_sales)
```

### Month over month

```dax
Sales MoM % =
VAR current_sales  = [Total Sales]
VAR previous_sales = [Sales PM]
RETURN
    DIVIDE(current_sales - previous_sales, previous_sales)
```

### YTD vs previous year YTD

```dax
Sales YTD PY =
CALCULATE(
    [Sales YTD],
    SAMEPERIODLASTYEAR(dim_date[date])
)

Sales YTD YoY % =
VAR current  = [Sales YTD]
VAR previous = [Sales YTD PY]
RETURN
    DIVIDE(current - previous, previous)
```

---

## Rolling Windows

Rolling calculations aggregate over a fixed number of days regardless of where the calendar boundaries fall — useful for smoothing trends.

### Rolling 30 days

```dax
Sales Rolling 30D =
CALCULATE(
    [Total Sales],
    DATESINPERIOD(
        dim_date[date],
        LASTDATE(dim_date[date]),
        -30,
        DAY
    )
)
```

### Rolling 3 months

```dax
Sales Rolling 3M =
CALCULATE(
    [Total Sales],
    DATESINPERIOD(
        dim_date[date],
        LASTDATE(dim_date[date]),
        -3,
        MONTH
    )
)
```

`DATESINPERIOD` starts from the last date in the current context and goes back by the specified interval — it always produces a window of fixed size regardless of month length or year boundaries.

---

## Comparing to a Fixed Date

Sometimes you need to compare against a specific date rather than the previous period dynamically.

```dax
-- Sales from a specific date onward
Sales From Launch =
CALCULATE(
    [Total Sales],
    dim_date[date] >= DATE(2023, 9, 1)
)

-- Sales up to and including a specific cutoff
Sales To Cutoff =
CALCULATE(
    [Total Sales],
    dim_date[date] <= DATE(2024, 12, 31)
)
```

---

## Semi-Additive Measures

Some measures should not be summed across time — balances, headcounts, and inventory levels are the classic examples. These are called semi-additive measures.

```dax
-- Inventory balance: last value in the period, not sum
Inventory Balance =
CALCULATE(
    LASTNONBLANK(fact_inventory[balance], 1),
    ALLSELECTED(dim_date[date])
)

-- Headcount: use LASTDATE to get the end-of-period snapshot
Headcount =
CALCULATE(
    SUM(fact_headcount[employees]),
    LASTDATE(dim_date[date])
)
```

!!! warning
    Never use `SUM` for semi-additive measures. Summing a daily balance across a month gives you a meaningless number. Use `LASTNONBLANKVALUE`, `LASTDATE`, or `AVERAGEX` depending on what the business requires — end-of-period snapshot, or average over the period.

---

## Common Mistakes

!!! warning
    Time intelligence functions require the date table to be **marked as a date table** in Power BI. If it is not marked, functions like `TOTALYTD` and `SAMEPERIODLASTYEAR` will not work correctly — they will either return blank or aggregate the wrong rows. Always verify this setting when time intelligence measures return unexpected results.

!!! warning
    Do not use time intelligence functions on a date column in the fact table directly. Always route through the date dimension. `TOTALYTD([Total Sales], fact_sales[order_date])` bypasses the date table and will not behave correctly with the rest of your model.

---

## Best Practices

- Always build time intelligence on top of a marked date table — never on a raw date column in the fact table
- Use variables in all period comparison measures — previous period values are always referenced at least twice
- Use `DATEADD` for flexible period shifting instead of hardcoding date ranges
- Use `DATESINPERIOD` for rolling windows — it handles month and year boundaries correctly
- For fiscal year time intelligence, pass the fiscal year end date as the second argument to `DATESYTD`
- Build previous period measures first (`Sales PY`, `Sales PM`), then reference them in growth measures — easier to debug and reuse
- Semi-additive measures need special handling — never sum a balance or headcount across time
