---
title: Date Spine
description: Generating a complete date sequence to fill gaps in time series data
tags: [sql, patterns, date-spine, time-series]
---

# Date Spine

A date spine is a complete, uninterrupted sequence of dates used as a backbone for time series analysis. When you join event data to a date spine, every date appears in the result — even dates with no events — making gaps visible rather than hidden.

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

### PostgreSQL — generate_series

```sql
-- Generate every date in a range
SELECT
    generate_series(
        '2024-01-01'::DATE,
        '2024-12-31'::DATE,
        '1 day'::INTERVAL
    )::DATE AS dt;
```

### Recursive CTE (ANSI SQL — portable)

```sql
WITH RECURSIVE date_spine AS (
    -- Anchor: start date
    SELECT DATE '2024-01-01' AS dt

    UNION ALL

    -- Recursive: add one day until end date
    SELECT dt + INTERVAL '1 day'
    FROM date_spine
    WHERE dt < DATE '2024-12-31'
)
SELECT dt FROM date_spine;
```

### From an Existing Date Dimension Table

In a data warehouse, a pre-built date dimension table is the most common approach.

```sql
-- Use the date dimension as your spine
SELECT d.date_key AS dt
FROM dim_date d
WHERE d.date_key BETWEEN '2024-01-01' AND '2024-12-31';
```

---

## Joining to Fill Gaps

```sql
WITH date_spine AS (
    SELECT generate_series(
        '2024-01-01'::DATE,
        '2024-12-31'::DATE,
        '1 day'::INTERVAL
    )::DATE AS dt
),
daily_sales AS (
    SELECT
        order_date,
        SUM(amount) AS revenue
    FROM orders
    WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31'
    GROUP BY order_date
)
SELECT
    s.dt                          AS order_date,
    COALESCE(d.revenue, 0)        AS revenue
FROM date_spine s
LEFT JOIN daily_sales d ON s.dt = d.order_date
ORDER BY s.dt;
```

Now every date in the range appears, with 0 for dates with no sales.

---

## Date Spine per Entity

When you need a complete date range per user, product, or region.

```sql
WITH date_spine AS (
    SELECT generate_series(
        '2024-01-01'::DATE,
        '2024-03-31'::DATE,
        '1 day'::INTERVAL
    )::DATE AS dt
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
WITH date_spine AS (
    SELECT generate_series(
        MIN(order_date), MAX(order_date), '1 day'::INTERVAL
    )::DATE AS dt
    FROM orders
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
CREATE TABLE dim_date AS
SELECT
    dt::DATE                                    AS date_key,
    EXTRACT(YEAR  FROM dt)::INT                 AS year,
    EXTRACT(MONTH FROM dt)::INT                 AS month,
    EXTRACT(DAY   FROM dt)::INT                 AS day,
    EXTRACT(DOW   FROM dt)::INT                 AS day_of_week,  -- 0=Sun, 6=Sat
    EXTRACT(WEEK  FROM dt)::INT                 AS week_of_year,
    EXTRACT(QUARTER FROM dt)::INT               AS quarter,
    TO_CHAR(dt, 'Month')                        AS month_name,
    TO_CHAR(dt, 'Day')                          AS day_name,
    CASE WHEN EXTRACT(DOW FROM dt) IN (0, 6)
         THEN FALSE ELSE TRUE END               AS is_weekday
FROM generate_series(
    '2020-01-01'::DATE,
    '2030-12-31'::DATE,
    '1 day'::INTERVAL
) AS dt;

-- Index for fast lookups
CREATE UNIQUE INDEX idx_dim_date_key ON dim_date (date_key);
```

---

## Vendor Notes

| Database | Date generation |
|---|---|
| PostgreSQL | `generate_series(start, end, interval)` |
| SQL Server | Recursive CTE or `sys.all_objects` cross join |
| MySQL | Recursive CTE (8.0+) |
| Snowflake | `GENERATOR(ROWCOUNT => n)` with `DATEADD` |
| BigQuery | `GENERATE_DATE_ARRAY(start, end, INTERVAL 1 DAY)` |

```sql
-- SQL Server — using a numbers table cross join
WITH numbers AS (
    SELECT TOP 365 ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1 AS n
    FROM sys.all_objects
)
SELECT DATEADD(DAY, n, '2024-01-01') AS dt
FROM numbers;

-- BigQuery
SELECT dt
FROM UNNEST(GENERATE_DATE_ARRAY('2024-01-01', '2024-12-31', INTERVAL 1 DAY)) AS dt;
```

---

## Best Practices

- Use a pre-built `dim_date` table in a data warehouse — generate it once, join to it everywhere
- Use a recursive CTE when `generate_series` is not available — it is ANSI SQL and portable
- Always `LEFT JOIN` your data to the date spine — not the other way around
- Use `COALESCE(value, 0)` to replace nulls from the left join with meaningful defaults
- Set the date spine range to match your reporting window — not the full history of the source table
