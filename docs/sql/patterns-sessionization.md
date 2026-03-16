---
title: Sessionization
description: Grouping user events into sessions based on inactivity gaps
tags: [sql, patterns, sessionization, window-functions]
---

# Sessionization

Sessionization groups a stream of user events into discrete sessions — periods of continuous activity separated by gaps of inactivity. It is a core pattern in product analytics, web analytics, and user behavior analysis.

!!! note
    Sessionization is the applied form of the maximum gap tolerance pattern. For the foundational concept of grouping consecutive sequences with allowed gaps, see [Gaps and Islands](patterns-gaps-islands.md).

---

## The Problem

Given a log of user events with timestamps, group them into sessions where events are considered part of the same session if they occur within N minutes of each other.

```
user_id   event_time           event_type
1         2024-01-01 10:00     page_view
1         2024-01-01 10:03     click
1         2024-01-01 10:05     page_view
1         2024-01-01 10:45     page_view   ← 40 min gap → new session
1         2024-01-01 10:47     click
2         2024-01-01 11:00     page_view
2         2024-01-01 11:02     click
```

---

## Step 1 — Detect Session Boundaries

Use `LAG()` to find the time since the previous event per user. Flag events that start a new session (gap exceeds the threshold).

```sql
WITH events_with_gap AS (
    SELECT
        user_id,
        event_time,
        event_type,
        LAG(event_time) OVER (
            PARTITION BY user_id
            ORDER BY event_time
        ) AS prev_event_time
    FROM user_events
),
session_flags AS (
    SELECT
        *,
        CASE
            WHEN prev_event_time IS NULL THEN 1   -- first event for user
            -- Minutes since last event: use platform-appropriate function
            -- PostgreSQL: EXTRACT(EPOCH FROM (event_time - prev_event_time)) / 60
            -- SQL Server:  DATEDIFF(minute, prev_event_time, event_time)
            -- MySQL:       TIMESTAMPDIFF(MINUTE, prev_event_time, event_time)
            WHEN EXTRACT(EPOCH FROM (event_time - prev_event_time)) / 60 > 30 THEN 1
            ELSE 0
        END AS is_session_start
    FROM events_with_gap
)
SELECT * FROM session_flags;
```

!!! note
    `EXTRACT(EPOCH FROM interval)` is PostgreSQL syntax for converting an interval to seconds. SQL Server uses `DATEDIFF(minute, prev_event_time, event_time)` directly. MySQL uses `TIMESTAMPDIFF(MINUTE, prev_event_time, event_time)`. There is no ANSI SQL standard for interval-to-minutes conversion — adapt the minutes calculation to your platform.

---

## Step 2 — Assign Session IDs

Use a cumulative sum of the session start flags to generate a session number per user.

```sql
WITH events_with_gap AS (
    SELECT
        user_id,
        event_time,
        event_type,
        LAG(event_time) OVER (
            PARTITION BY user_id ORDER BY event_time
        ) AS prev_event_time
    FROM user_events
),
session_flags AS (
    SELECT
        *,
        CASE
            WHEN prev_event_time IS NULL THEN 1
            WHEN EXTRACT(EPOCH FROM (event_time - prev_event_time)) / 60 > 30 THEN 1
            ELSE 0
        END AS is_session_start
    FROM events_with_gap
),
sessions AS (
    SELECT
        user_id,
        event_time,
        event_type,
        SUM(is_session_start) OVER (
            PARTITION BY user_id
            ORDER BY event_time
        ) AS session_number
    FROM session_flags
)
SELECT
    user_id,
    session_number,
    event_time,
    event_type
FROM sessions
ORDER BY user_id, session_number, event_time;
```

---

## Step 3 — Aggregate Sessions

Summarize each session — start time, end time, duration, and event count.

```sql
WITH events_with_gap AS (
    SELECT
        user_id,
        event_time,
        event_type,
        LAG(event_time) OVER (
            PARTITION BY user_id ORDER BY event_time
        ) AS prev_event_time
    FROM user_events
),
session_flags AS (
    SELECT *,
        CASE
            WHEN prev_event_time IS NULL THEN 1
            WHEN EXTRACT(EPOCH FROM (event_time - prev_event_time)) / 60 > 30 THEN 1
            ELSE 0
        END AS is_session_start
    FROM events_with_gap
),
sessions AS (
    SELECT
        user_id,
        event_time,
        event_type,
        SUM(is_session_start) OVER (
            PARTITION BY user_id ORDER BY event_time
        ) AS session_number
    FROM session_flags
)
SELECT
    user_id,
    session_number,
    MIN(event_time)                                                        AS session_start,
    MAX(event_time)                                                        AS session_end,
    EXTRACT(EPOCH FROM (MAX(event_time) - MIN(event_time))) / 60           AS duration_minutes,
    COUNT(*)                                                               AS event_count
FROM sessions
GROUP BY user_id, session_number
ORDER BY user_id, session_number;
```

---

## Creating a Unique Session ID

Generate a globally unique session identifier by combining user and session number.

```sql
-- ANSI portable — combine user_id and session_number into a unique key
SELECT
    CONCAT(CAST(user_id AS VARCHAR), '-', CAST(session_number AS VARCHAR)) AS session_id,
    user_id,
    session_number,
    session_start,
    session_end,
    event_count
FROM session_summary;
```

!!! note
    Hash-based session IDs (e.g. `MD5(user_id || session_number)`) are common in practice but `MD5()` is not ANSI SQL — it is supported in PostgreSQL, MySQL, and Snowflake but not in SQL Server (which uses `HASHBYTES('MD5', ...)`). The `CONCAT + CAST` approach above is portable and sufficient for most use cases.

---

## Choosing the Session Timeout

The session timeout (inactivity threshold) depends on the product and use case.

| Context | Typical timeout |
|---|---|
| Web analytics | 30 minutes (Google Analytics default) |
| Mobile app | 15–30 minutes |
| E-commerce checkout | 60 minutes |
| B2B SaaS | 60–120 minutes |
| Real-time gaming | 5–10 minutes |

---

## Practical Use Cases

| Use Case | Description |
|---|---|
| Session duration | Average time users spend per session |
| Events per session | How many actions happen in a session |
| Bounce detection | Sessions with only one event |
| Conversion funnel | Track which sessions include a purchase event |
| User engagement | Sessions per user per day/week |

---

## Best Practices

- Choose a session timeout that reflects actual user behavior — validate against real usage patterns
- Index `(user_id, event_time)` — sessionization queries partition by user and order by time
- Store session IDs in the events table after computing them — avoid recomputing in every downstream query
- Use a CTE pipeline (gap → flag → session number → aggregate) — each step is testable independently
- Handle time zones carefully — convert all timestamps to UTC before sessionization
- Use platform-appropriate interval functions for gap calculation — `EXTRACT(EPOCH FROM ...)` in PostgreSQL, `DATEDIFF` in SQL Server, `TIMESTAMPDIFF` in MySQL