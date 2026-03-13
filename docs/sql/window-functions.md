---
title: Window Functions
description: Calculations across related rows without collapsing the result set
tags: [sql, dql, window-functions]
---

# Window Functions

A window function performs a calculation across a set of rows related to the current row — without collapsing the result like `GROUP BY` does. Each row keeps its identity while gaining access to aggregate or positional information from its surrounding rows.

---

## Syntax

```sql
function() OVER (
    PARTITION BY column   -- divide rows into groups (optional)
    ORDER BY column       -- define order within each partition (optional)
    ROWS/RANGE BETWEEN ... AND ...  -- define the frame (optional)
)
```

All three clauses inside `OVER()` are optional — but most window functions need at least `ORDER BY` to be meaningful.

---

## Aggregate Window Functions

The same aggregate functions used with `GROUP BY`, but applied over a window without collapsing rows.

```sql
SELECT
    employee_id,
    department,
    salary,
    SUM(salary)   OVER (PARTITION BY department) AS dept_total,
    AVG(salary)   OVER (PARTITION BY department) AS dept_avg,
    COUNT(*)      OVER (PARTITION BY department) AS dept_headcount,
    MIN(salary)   OVER (PARTITION BY department) AS dept_min,
    MAX(salary)   OVER (PARTITION BY department) AS dept_max,
    ROUND(salary * 100.0 / SUM(salary) OVER (PARTITION BY department), 2) AS pct_of_dept
FROM employees;
```

Every row is returned. No rows are collapsed. Each employee still appears with their own salary — plus the department-level aggregates alongside.

---

## Ranking Functions

### ROW_NUMBER, RANK, DENSE_RANK

```sql
SELECT
    employee_id,
    department,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num,
    RANK()       OVER (PARTITION BY department ORDER BY salary DESC) AS rank,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rank
FROM employees;
```

| Function | Ties | Gaps after ties |
|---|---|---|
| `ROW_NUMBER()` | Unique number always | — |
| `RANK()` | Same rank for ties | ✅ Yes |
| `DENSE_RANK()` | Same rank for ties | ❌ No |

```
Salary   ROW_NUMBER   RANK   DENSE_RANK
9000          1         1        1
8000          2         2        2
8000          3         2        2
7000          4         4        3   ← RANK skips 3, DENSE_RANK does not
```

### NTILE — Divide into Buckets

```sql
-- Divide employees into 4 salary quartiles
SELECT
    employee_id,
    salary,
    NTILE(4) OVER (ORDER BY salary DESC) AS quartile
FROM employees;
```

### PERCENT_RANK and CUME_DIST

```sql
SELECT
    employee_id,
    salary,
    ROUND(PERCENT_RANK() OVER (ORDER BY salary), 4) AS percent_rank,
    ROUND(CUME_DIST()    OVER (ORDER BY salary), 4) AS cume_dist
FROM employees;
```

---

## LAG and LEAD

Access values from previous or following rows without a self-join.

```sql
SELECT
    order_date,
    amount,
    LAG(amount)  OVER (ORDER BY order_date) AS prev_amount,
    LEAD(amount) OVER (ORDER BY order_date) AS next_amount,
    amount - LAG(amount) OVER (ORDER BY order_date) AS change
FROM daily_sales;
```

With offset and default value:

```sql
-- Look back 3 rows, default to 0 if no prior row exists
SELECT
    order_date,
    amount,
    LAG(amount, 3, 0) OVER (ORDER BY order_date) AS amount_3_days_ago
FROM daily_sales;
```

---

## Running Totals and Moving Averages

### Running Total

```sql
SELECT
    order_date,
    amount,
    SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM daily_sales;
```

### Running Total per Partition

```sql
SELECT
    customer_id,
    order_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY customer_id
        ORDER BY order_date
    ) AS customer_running_total
FROM orders;
```

### 7-Day Rolling Average

```sql
SELECT
    order_date,
    amount,
    AVG(amount) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_7day_avg
FROM daily_sales;
```

---

## Window Frames

The frame clause controls exactly which rows are included in the calculation relative to the current row.

```sql
-- All rows in the partition (default when no ORDER BY)
SUM(amount) OVER (PARTITION BY customer_id)

-- From the start of the partition to the current row
SUM(amount) OVER (
    ORDER BY order_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
)

-- 3-row centered moving average
AVG(amount) OVER (
    ORDER BY order_date
    ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
)

-- From current row to end of partition
SUM(amount) OVER (
    ORDER BY order_date
    ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
)
```

### ROWS vs RANGE

```sql
-- ROWS: physical rows (exact count)
SUM(amount) OVER (ORDER BY order_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)

-- RANGE: logical range (includes all rows with the same ORDER BY value)
SUM(amount) OVER (ORDER BY order_date RANGE BETWEEN 6 PRECEDING AND CURRENT ROW)
```

!!! tip
    Use `ROWS` for most cases — it is predictable and performs better. Use `RANGE` only when you need to include ties in the frame boundary.

---

## FIRST_VALUE and LAST_VALUE

```sql
SELECT
    employee_id,
    department,
    salary,
    FIRST_VALUE(salary) OVER (
        PARTITION BY department ORDER BY salary DESC
    ) AS highest_in_dept,
    LAST_VALUE(salary) OVER (
        PARTITION BY department
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS lowest_in_dept
FROM employees;
```

!!! warning
    `LAST_VALUE` requires an explicit frame of `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` — otherwise it defaults to only looking up to the current row and returns the current row's value.

---

## Deduplication with ROW_NUMBER

The most common window function pattern in data engineering — keep one row per business key.

```sql
SELECT *
FROM (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY updated_at DESC
        ) AS rn
    FROM customers
) t
WHERE rn = 1;
```

---

## Window Functions vs GROUP BY

```sql
-- GROUP BY — collapses rows, one result per group
SELECT department, AVG(salary) AS dept_avg
FROM employees
GROUP BY department;

-- Window function — keeps all rows, adds the aggregate alongside
SELECT
    employee_id,
    department,
    salary,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees;
```

Use `GROUP BY` when you want a summary. Use window functions when you want the detail rows with summary context attached.

---

## Common Use Cases

| Use Case | Function |
|---|---|
| Rank items within a group | `RANK()`, `DENSE_RANK()` |
| Keep latest record per key | `ROW_NUMBER()` |
| Running total | `SUM() OVER (ORDER BY ...)` |
| Period-over-period change | `LAG()` |
| Rolling average | `AVG() OVER (ROWS BETWEEN ...)` |
| Percentile / distribution | `PERCENT_RANK()`, `NTILE()` |
| First / last value in group | `FIRST_VALUE()`, `LAST_VALUE()` |
| % of total within group | `SUM() OVER (PARTITION BY ...)` |

---

## Best Practices

- Name window expressions clearly with aliases — `dept_avg`, `running_total`, not just `avg`
- Reuse the same `OVER()` definition with a named window when using the same window multiple times
- Use `ROWS` instead of `RANGE` for rolling calculations unless you specifically need to handle ties
- Always add an explicit frame when using `LAST_VALUE` — the default frame will give unexpected results
- Avoid window functions on very large unfiltered datasets — filter first in a CTE, then apply the window

```sql
-- Named window — avoids repeating the same OVER() clause
SELECT
    employee_id,
    salary,
    AVG(salary)  OVER w AS dept_avg,
    MAX(salary)  OVER w AS dept_max,
    MIN(salary)  OVER w AS dept_min
FROM employees
WINDOW w AS (PARTITION BY department ORDER BY salary DESC);
```