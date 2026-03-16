---
title: Date Spine
description: Generating a complete date sequence to fill gaps in time series data
tags: [sql, patterns, date-spine, time-series]
---

# Date Spine

A date spine is a complete, uninterrupted sequence of dates used as a backbone for time series analysis. When you join event data to a date spine, every date appears in the result — even dates with no events — making gaps visible rather than hidden.

!!! note
    This page was introduced as a use case for recursive CTEs. See [CTEs](ctes.md) for the recursive CTE syntax that powers the portable date spine approach.

---

## The Problem

Sales data only contains dates where sales occurred. Dates with zero sales are simply missing.

```sql
SELECT order_date, SUM(amount) AS revenue
FROM orders
GROUP BY order_date
ORDER BY order_date;
```

```
order_date   revenue
2024-01-01   1200
2024-01-02   950
2024-01-04   1800   ← Jan 3 missing — no sales, not shown
2024-01-05   600
```

With a date spine, Jan 3 appears with revenue = 0.

---

## Generating a Date Spine

### Recursive CTE — ANSI SQL (Portable)

The portable approach works across all platforms that support recursive CTEs.

```sql
WITH RECURSIVE date_spine AS (
    -- Anchor: start date
    SELECT CAST('2024-01-01' AS DATE) AS dt

    UNION ALL

    -- Recursive: add one day until end date
    SELECT dt + INTERVAL '1' DAY
    FROM date_spine
    WHERE dt < CAST('2024-12-31' AS DATE)
)
SELECT dt FROM date_spine;
```

!!! note
    SQL Server uses plain `WITH` — the `RECURSIVE` keyword is not supported and will cause a syntax error in T-SQL. See [CTEs](ctes.md) for the full platform note.

### PostgreSQL — generate_series

PostgreSQL offers a more concise built-in function:

```sql
-- PostgreSQL only
SELECT
    CAST(generate_series(
        CAST('2024-01-01' AS DATE),
        CAST('2024-12-31' AS DATE),
        INTERVAL '1 day'
    ) AS DATE) AS dt;
```

!!! note
    `generate_series` is PostgreSQL-specific. Use the recursive CTE approach for cross-platform queries.

### From an Existing Date Dimension Table

In a data warehouse, a pre-built date dimension table is the most common approach.

```sql
-- Use the date dimension as your spine
SELECT date_key AS dt
FROM dim_date
WHERE date_key BETWEEN CAST('2024-01-01' AS DATE) AND CAST('2024-12-31' AS DATE);
```

---

## Joining to Fill Gaps

Using the ANSI recursive CTE approach:

```sql
WITH RECURSIVE date_spine AS (
    SELECT CAST('2024-01-01' AS DATE) AS dt
    UNION ALL
    SELECT dt + INTERVAL '1' DAY
    FROM date_spine
    WHERE dt < CAST('2024-12-31' AS DATE)
),
daily_sales AS (
    SELECT
        order_date,
        SUM(amount) AS revenue
    FROM orders
    WHERE order_date BETWEEN CAST('2024-01-01' AS DATE) AND CAST('2024-12-31' AS DATE)
    GROUP BY order_date
)
SELECT
    s.dt                       AS order_date,
    COALESCE(d.revenue, 0)     AS revenue
FROM date_spine s
LEFT JOIN daily_sales d ON s.dt = d.order_date
ORDER BY s.dt;
```

Now every date in the range appears, with 0 for dates with no sales.

---

## Date Spine per Entity

When you need a complete date range per user, product, or region.

```sql
WITH RECURSIVE date_spine AS (
    SELECT CAST('2024-01-01' AS DATE) AS dt
    UNION ALL
    SELECT dt + INTERVAL '1' DAY
    FROM date_spine
    WHERE dt < CAST('2024-03-31' AS DATE)
),
users AS (
    SELECT DISTINCT user_id FROM user_activity
),
user_dates AS (
    -- Cross join: every user × every date
    SELECT u.user_id, s.dt
    FROM users u
    CROSS JOIN date_spine s
),
daily_activity AS (
    SELECT user_id, activity_date, COUNT(*) AS events
    FROM user_activity
    GROUP BY user_id, activity_date
)
SELECT
    ud.user_id,
    ud.dt,
    COALESCE(da.events, 0) AS events
FROM user_dates ud
LEFT JOIN daily_activity da
    ON ud.user_id = da.user_id
    AND ud.dt = da.activity_date
ORDER BY ud.user_id, ud.dt;
```

---

## Date Spine for Running Totals with Gaps

Without a date spine, running totals skip days with no activity. With a spine, the running total continues correctly.

```sql
WITH RECURSIVE date_spine AS (
    SELECT MIN(order_date) AS dt FROM orders
    UNION ALL
    SELECT dt + INTERVAL '1' DAY
    FROM date_spine
    WHERE dt < (SELECT MAX(order_date) FROM orders)
),
daily AS (
    SELECT order_date, SUM(amount) AS revenue
    FROM orders
    GROUP BY order_date
),
filled AS (
    SELECT
        s.dt,
        COALESCE(d.revenue, 0) AS daily_revenue
    FROM date_spine s
    LEFT JOIN daily d ON s.dt = d.order_date
)
SELECT
    dt,
    daily_revenue,
    SUM(daily_revenue) OVER (ORDER BY dt) AS running_total
FROM filled
ORDER BY dt;
```

---

## Building a Date Dimension Table

For a data warehouse, generate a permanent date dimension once and reuse it everywhere.

```sql
-- PostgreSQL example — using generate_series and EXTRACT
CREATE TABLE dim_date AS
SELECT
    CAST(dt AS DATE)                AS date_key,
    EXTRACT(YEAR    FROM dt)        AS year,
    EXTRACT(MONTH   FROM dt)        AS month,
    EXTRACT(DAY     FROM dt)        AS day,
    EXTRACT(QUARTER FROM dt)        AS quarter
FROM generate_series(
    CAST('2020-01-01' AS DATE),
    CAST('2030-12-31' AS DATE),
    INTERVAL '1 day'
) AS dt;

-- Index for fast lookups
CREATE UNIQUE INDEX idx_dim_date_key ON dim_date (date_key);
```

!!! note
    `EXTRACT(DOW ...)` (day of week) and `EXTRACT(WEEK ...)` (week of year) are not ANSI SQL — they are PostgreSQL extensions. ANSI `EXTRACT` supports `YEAR`, `MONTH`, `DAY`, `HOUR`, `MINUTE`, and `SECOND` only. For day of week and week number, use platform-specific functions: `DATEPART(weekday, dt)` in SQL Server, `DAYOFWEEK(dt)` in MySQL.

!!! note
    `TO_CHAR(dt, 'Month')` for month and day names is PostgreSQL-specific. SQL Server uses `DATENAME(month, dt)`, MySQL uses `MONTHNAME(dt)`. There is no ANSI SQL standard for formatting date parts as strings.

---

## Vendor Notes

| Database | Date generation |
|---|---|
| PostgreSQL | `generate_series(start, end, interval)` |
| SQL Server | Recursive CTE (no `RECURSIVE` keyword) |
| MySQL | Recursive CTE (8.0+) |
| Snowflake | `GENERATOR(ROWCOUNT => n)` with `DATEADD` |
| BigQuery | `GENERATE_DATE_ARRAY(start, end, INTERVAL 1 DAY)` |

```sql
-- SQL Server — recursive CTE (no RECURSIVE keyword)
WITH date_spine AS (
    SELECT CAST('2024-01-01' AS DATE) AS dt
    UNION ALL
    SELECT DATEADD(DAY, 1, dt)
    FROM date_spine
    WHERE dt < CAST('2024-12-31' AS DATE)
)
SELECT dt FROM date_spine
OPTION (MAXRECURSION 366);

-- BigQuery
SELECT dt
FROM UNNEST(GENERATE_DATE_ARRAY('2024-01-01', '2024-12-31', INTERVAL 1 DAY)) AS dt;
```

!!! note
    SQL Server requires `OPTION (MAXRECURSION n)` for recursive CTEs that exceed 100 iterations — the default limit. Set it to the number of days in your range, or use `OPTION (MAXRECURSION 0)` to remove the limit entirely.

---

## Best Practices

- Use a pre-built `dim_date` table in a data warehouse — generate it once, join to it everywhere
- Use a recursive CTE when `generate_series` is not available — it is ANSI SQL and portable
- Always `LEFT JOIN` your data to the date spine — not the other way around
- Use `COALESCE(value, 0)` to replace nulls from the left join with meaningful defaults
- Set the date spine range to match your reporting window — not the full history of the source table
- Use `CAST(... AS DATE)` instead of `::DATE` for portable date literal casting