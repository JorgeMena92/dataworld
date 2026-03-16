---
title: Gaps and Islands
description: Identifying consecutive sequences and breaks in sequences with SQL
tags: [sql, patterns, gaps-islands, window-functions]
---

# Gaps and Islands

The gaps and islands problem involves identifying consecutive sequences (islands) and breaks in those sequences (gaps) within ordered data. It appears in time series analysis, session detection, scheduling, and any dataset where continuity matters.

---

## The Problem

Given a sequence of dates, IDs, or events — find where the sequence is unbroken (islands) and where there are missing values (gaps).

```
Dates present:  Jan 1, Jan 2, Jan 3, Jan 5, Jan 6, Jan 9
                └── island 1 ──┘  └─ island 2 ┘  └ island 3 ┘
Gaps:                        Jan 4          Jan 7, Jan 8
```

---

## Finding Gaps in a Sequence

### Gap in Dates — ANSI Portable with LAG

The simplest portable approach uses `LAG()` to compare each date to the previous one — a gap exists when the difference is greater than 1 day.

```sql
-- Find dates where a gap exists before them
WITH ordered AS (
    SELECT
        activity_date,
        LAG(activity_date) OVER (ORDER BY activity_date) AS prev_date
    FROM (SELECT DISTINCT activity_date FROM user_activity) t
)
SELECT
    prev_date + INTERVAL '1' DAY  AS gap_start,
    activity_date - INTERVAL '1' DAY AS gap_end
FROM ordered
WHERE activity_date - prev_date > INTERVAL '1' DAY
ORDER BY gap_start;
```

!!! note
    For a complete date spine approach — generating every expected date and left-joining to find missing ones — see [Date Spine](patterns-date-spine.md). The `LAG()` approach above is more portable; the date spine approach is more explicit about which dates are missing.

### Gap in Integer Sequences

```sql
-- Find gaps in order IDs
WITH numbered AS (
    SELECT
        order_id,
        LAG(order_id) OVER (ORDER BY order_id) AS prev_id
    FROM orders
)
SELECT
    prev_id + 1  AS gap_start,
    order_id - 1 AS gap_end,
    order_id - prev_id - 1 AS missing_count
FROM numbered
WHERE order_id - prev_id > 1
ORDER BY gap_start;
```

---

## Finding Islands — Consecutive Sequences

The classic technique: subtract the row number from the value. Consecutive values produce the same result — forming a group key.

### Consecutive Active Days

```sql
WITH ranked AS (
    SELECT
        user_id,
        activity_date,
        activity_date - CAST(
            ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY activity_date)
            AS INT
        ) AS island_group
    FROM (
        SELECT DISTINCT user_id, activity_date FROM user_activity
    ) t
)
SELECT
    user_id,
    MIN(activity_date) AS island_start,
    MAX(activity_date) AS island_end,
    COUNT(*)           AS consecutive_days
FROM ranked
GROUP BY user_id, island_group
ORDER BY user_id, island_start;
```

The key insight: if dates are consecutive, subtracting an incrementing row number always gives the same date. A gap breaks the pattern and creates a new group.

```
Date        ROW_NUMBER   Date - RowNum
Jan 1           1        Dec 31  ← island group A
Jan 2           2        Dec 31  ← island group A
Jan 3           3        Dec 31  ← island group A
Jan 5           4        Jan 1   ← island group B (gap at Jan 4)
Jan 6           5        Jan 1   ← island group B
```

!!! note
    Subtracting an integer from a date (`activity_date - CAST(ROW_NUMBER() AS INT)`) is supported in PostgreSQL and some other databases. SQL Server requires `DATEADD(day, -ROW_NUMBER() OVER (...), activity_date)`. MySQL supports direct date arithmetic with integers. Always verify date subtraction behavior on your platform.

---

## Consecutive Dates with a Maximum Gap Allowed

Sometimes a gap of 1 or 2 days should still be considered the same island (weekends, for example).

```sql
WITH ordered AS (
    SELECT
        user_id,
        activity_date,
        LAG(activity_date) OVER (PARTITION BY user_id ORDER BY activity_date) AS prev_date
    FROM (SELECT DISTINCT user_id, activity_date FROM user_activity) t
),
flagged AS (
    SELECT
        user_id,
        activity_date,
        -- Start a new island if gap is more than 2 days
        CASE WHEN activity_date - prev_date > 2 THEN 1 ELSE 0 END AS new_island
    FROM ordered
),
grouped AS (
    SELECT
        user_id,
        activity_date,
        SUM(new_island) OVER (PARTITION BY user_id ORDER BY activity_date) AS island_id
    FROM flagged
)
SELECT
    user_id,
    MIN(activity_date) AS island_start,
    MAX(activity_date) AS island_end,
    COUNT(*)           AS days_in_island
FROM grouped
GROUP BY user_id, island_id
ORDER BY user_id, island_start;
```

!!! note
    The maximum gap tolerance pattern is closely related to sessionization — grouping events into sessions based on inactivity thresholds. See [Sessionization](patterns-sessionization.md) for the full treatment applied to event streams.

---

## Practical Use Cases

| Use Case | Description |
|---|---|
| Active streaks | Find the longest consecutive login streak per user |
| Missing transactions | Identify gaps in invoice or order sequences |
| Availability windows | Find continuous blocks of available time slots |
| Sensor data | Detect outages in continuous measurement streams |
| SLA monitoring | Find periods where a metric was continuously above a threshold |

---

## Longest Consecutive Streak

```sql
WITH ranked AS (
    SELECT
        user_id,
        activity_date,
        activity_date - CAST(
            ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY activity_date) AS INT
        ) AS island_group
    FROM (SELECT DISTINCT user_id, activity_date FROM user_activity) t
),
islands AS (
    SELECT
        user_id,
        COUNT(*) AS streak_length
    FROM ranked
    GROUP BY user_id, island_group
)
SELECT
    user_id,
    MAX(streak_length) AS longest_streak
FROM islands
GROUP BY user_id
ORDER BY longest_streak DESC;
```

---

## Best Practices

- Always deduplicate (`SELECT DISTINCT`) before applying the island grouping technique — duplicate dates break the row number subtraction logic
- Use `LAG()` for gap detection when you only need to find breaks, not group them
- Use the row number subtraction technique when you need to group consecutive rows into islands
- For the date subtraction technique, verify behavior on your platform — use `DATEADD` in SQL Server
- For large datasets, ensure the partition key and order column are indexed