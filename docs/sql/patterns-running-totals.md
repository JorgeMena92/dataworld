---
title: Running Totals
description: Cumulative sums, moving averages, and rolling calculations with SQL window functions
tags: [sql, patterns, running-totals, window-functions]
---

# Running Totals

A running total (cumulative sum) adds up values progressively as you move through ordered rows. Variants include moving averages, rolling sums, and period-over-period comparisons. All are implemented with window functions.

---

## Basic Running Total

```sql
SELECT
    order_date,
    amount,
    SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM daily_sales
ORDER BY order_date;
```

```
order_date   amount   running_total
2024-01-01    100         100
2024-01-02    250         350
2024-01-03    180         530
2024-01-04    320         850
```

---

## Running Total per Partition

Reset the running total for each group — customer, region, category.

```sql
SELECT
    customer_id,
    order_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY customer_id
        ORDER BY order_date
    ) AS customer_running_total
FROM orders
ORDER BY customer_id, order_date;
```

---

## Running Count and Running Average

```sql
SELECT
    order_date,
    amount,
    COUNT(*)    OVER (ORDER BY order_date) AS running_count,
    SUM(amount) OVER (ORDER BY order_date) AS running_total,
    AVG(amount) OVER (ORDER BY order_date) AS running_avg
FROM daily_sales
ORDER BY order_date;
```

---

## Rolling Window — Fixed Number of Rows

Calculate over a fixed number of preceding rows — useful for smoothing noisy time series.

```sql
-- 7-day rolling sum and average
SELECT
    order_date,
    amount,
    SUM(amount) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_7day_sum,
    AVG(amount) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_7day_avg
FROM daily_sales
ORDER BY order_date;
```

---

## Rolling Window — Time-Based Range

Use `RANGE` instead of `ROWS` to include all rows within a time interval, regardless of how many rows that is.

```sql
-- Sum of all sales in the last 7 days (by date value, not row count)
SELECT
    order_date,
    amount,
    SUM(amount) OVER (
        ORDER BY order_date
        RANGE BETWEEN INTERVAL '6' DAY PRECEDING AND CURRENT ROW
    ) AS rolling_7day_sum
FROM daily_sales
ORDER BY order_date;
```

!!! tip
    Use `ROWS` when you want a fixed number of rows (e.g. last 7 rows). Use `RANGE` when you want a fixed time window (e.g. last 7 days) — especially when there are missing dates.

!!! note
    `RANGE BETWEEN INTERVAL '6' DAY PRECEDING` is the ANSI form. PostgreSQL also accepts `INTERVAL '6 days'`. SQL Server supports time-based `RANGE` frames only with `UNBOUNDED` — for date-based rolling windows in SQL Server, use `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` with a pre-aggregated daily table to approximate the behavior.

---

## Centered Moving Average

Include rows both before and after the current row.

```sql
-- 3-period centered moving average
SELECT
    order_date,
    amount,
    AVG(amount) OVER (
        ORDER BY order_date
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS centered_3day_avg
FROM daily_sales
ORDER BY order_date;
```

---

## Period-over-Period Comparison

Compare each period to the same period in the previous cycle using `LAG`.

```sql
WITH monthly AS (
    SELECT
        EXTRACT(YEAR  FROM order_date) AS order_year,
        EXTRACT(MONTH FROM order_date) AS order_month,
        SUM(amount) AS total
    FROM orders
    GROUP BY
        EXTRACT(YEAR  FROM order_date),
        EXTRACT(MONTH FROM order_date)
)
SELECT
    order_year,
    order_month,
    total,
    LAG(total) OVER (ORDER BY order_year, order_month)         AS prev_month,
    total - LAG(total) OVER (ORDER BY order_year, order_month) AS change,
    ROUND(
        100.0 * (total - LAG(total) OVER (ORDER BY order_year, order_month))
        / NULLIF(LAG(total) OVER (ORDER BY order_year, order_month), 0),
        2
    ) AS pct_change
FROM monthly
ORDER BY order_year, order_month;
```

---

## Year-over-Year Comparison

```sql
WITH monthly AS (
    SELECT
        EXTRACT(YEAR  FROM order_date) AS order_year,
        EXTRACT(MONTH FROM order_date) AS order_month,
        SUM(amount) AS total
    FROM orders
    GROUP BY
        EXTRACT(YEAR  FROM order_date),
        EXTRACT(MONTH FROM order_date)
)
SELECT
    order_year,
    order_month,
    total,
    LAG(total, 12) OVER (ORDER BY order_year, order_month) AS same_month_last_year,
    ROUND(
        100.0 * (total - LAG(total, 12) OVER (ORDER BY order_year, order_month))
        / NULLIF(LAG(total, 12) OVER (ORDER BY order_year, order_month), 0),
        2
    ) AS yoy_pct_change
FROM monthly
ORDER BY order_year, order_month;
```

---

## Running Total with Reset on Condition

Reset the running total when a condition is met — useful for streak tracking or quota resets.

```sql
WITH flagged AS (
    SELECT
        sale_date,
        amount,
        SUM(CASE WHEN amount < 0 THEN 1 ELSE 0 END)
            OVER (ORDER BY sale_date) AS reset_group
    FROM daily_sales
)
SELECT
    sale_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY reset_group
        ORDER BY sale_date
    ) AS running_total_since_reset
FROM flagged
ORDER BY sale_date;
```

---

## Best Practices

- Use `ROWS BETWEEN` for fixed row counts, `RANGE BETWEEN` for time-based windows
- Always include `ORDER BY` inside the window — without it, the frame is undefined
- Use `NULLIF` when calculating percentages to avoid division by zero
- For period-over-period, use `LAG(value, n)` where `n` is the number of periods to look back
- Filter and aggregate in a CTE first, then apply window functions — cleaner and more efficient
- Use `EXTRACT(YEAR ...)` / `EXTRACT(MONTH ...)` for date grouping instead of `DATE_TRUNC` — more portable across platforms