---
title: Materialized Views
description: Pre-computed, stored query results with ANSI SQL materialized views
tags: [sql, database-objects, materialized-views]
---

# Materialized Views

A materialized view is a view that physically stores its query results. Unlike a regular view — which re-executes its query every time it is referenced — a materialized view is pre-computed and persisted to disk. Queries against it read stored data, not the live underlying tables.

This makes materialized views significantly faster to query, at the cost of data freshness.

!!! note
    Materialized views are not part of the ANSI SQL standard. Support, syntax, and refresh behavior vary significantly across platforms — see the [Vendor Support](#vendor-support) section for details.

---

## View vs Materialized View

| | View | Materialized View |
|---|---|---|
| Stores data | ❌ | ✅ |
| Query performance | Depends on base tables | Fast — reads stored data |
| Data freshness | Always current | Stale until refreshed |
| Refresh required | ❌ | ✅ |
| Can be indexed | ❌ | ✅ |
| Best for | Real-time data access | Pre-aggregated, expensive queries |

---

## Creating a Materialized View

```sql
-- PostgreSQL
CREATE MATERIALIZED VIEW monthly_revenue AS
SELECT
    EXTRACT(YEAR  FROM order_date) AS order_year,
    EXTRACT(MONTH FROM order_date) AS order_month,
    country,
    SUM(amount) AS total_revenue,
    COUNT(*)    AS total_orders
FROM orders
JOIN customers USING (customer_id)
GROUP BY
    EXTRACT(YEAR  FROM order_date),
    EXTRACT(MONTH FROM order_date),
    country;
```

!!! note
    `DATE_TRUNC` is a common alternative for truncating dates to month boundaries but is not ANSI SQL. The `EXTRACT(YEAR ...)` / `EXTRACT(MONTH ...)` approach above is fully portable. See [Scalar Functions](scalar-functions.md).

---

## Refreshing a Materialized View

Data in a materialized view becomes stale as the underlying tables change. It must be explicitly refreshed.

```sql
-- Full refresh — recomputes the entire result
REFRESH MATERIALIZED VIEW monthly_revenue;

-- Concurrent refresh — allows reads during refresh (PostgreSQL)
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue;
```

!!! tip
    `CONCURRENTLY` requires a unique index on the materialized view. Without it, the refresh locks the view and blocks reads for its duration.

---

## Adding Indexes to Materialized Views

One of the key advantages over regular views — you can index a materialized view to speed up queries against it.

```sql
-- Create an index on the materialized view
CREATE INDEX idx_mv_monthly_revenue_month
ON monthly_revenue (month);

CREATE INDEX idx_mv_monthly_revenue_country
ON monthly_revenue (country);
```

---

## Dropping a Materialized View

```sql
-- ANSI SQL (where supported)
DROP MATERIALIZED VIEW monthly_revenue;

-- Vendor extension — supported in PostgreSQL, Oracle, Snowflake
DROP MATERIALIZED VIEW IF EXISTS monthly_revenue;
```

---

## Common Use Cases

### Pre-aggregated Reporting Tables

```sql
CREATE MATERIALIZED VIEW sales_dashboard AS
SELECT
    CAST(order_date AS DATE)  AS day,
    c.country,
    c.segment,
    COUNT(*)                  AS orders,
    SUM(o.amount)             AS revenue,
    AVG(o.amount)             AS avg_order_value
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'completed'
GROUP BY
    CAST(order_date AS DATE),
    c.country,
    c.segment;

-- Refresh daily
REFRESH MATERIALIZED VIEW sales_dashboard;
```

### Flattening Complex Joins

```sql
CREATE MATERIALIZED VIEW order_details AS
SELECT
    o.order_id,
    o.order_date,
    o.amount,
    c.first_name,
    c.country,
    p.product_name,
    oi.quantity
FROM orders o
JOIN customers c   ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p    ON oi.product_id = p.product_id;
```

---

## Refresh Strategies

| Strategy | How | When to use |
|---|---|---|
| Manual | `REFRESH MATERIALIZED VIEW` on demand | Ad hoc or triggered by pipeline |
| Scheduled | Cron job or scheduler calling REFRESH | Regular cadence (hourly, daily) |
| On commit | Automatic after each transaction | Small views, low write volume |
| Incremental | Only refresh changed data | Large views — platform dependent |

---

## Vendor Support

Materialized views are not part of the ANSI SQL standard — support and behavior vary significantly.

| Database | Support | Notes |
|---|---|---|
| PostgreSQL | ✅ Full | Manual refresh, `CONCURRENTLY` option |
| Oracle | ✅ Full | Auto-refresh on commit, incremental refresh |
| SQL Server | Partial | Called *Indexed Views* — limited aggregation support |
| MySQL | ❌ | No native support — simulate with tables + triggers |
| Snowflake | ✅ | Automatic incremental refresh |
| BigQuery | ✅ | Automatic incremental refresh |

```sql
-- SQL Server equivalent — Indexed View
CREATE VIEW order_totals WITH SCHEMABINDING AS
SELECT customer_id, COUNT_BIG(*) AS order_count, SUM(amount) AS total
FROM dbo.orders
GROUP BY customer_id;

CREATE UNIQUE CLUSTERED INDEX idx_order_totals
ON order_totals (customer_id);
```

---

## Best Practices

- Use materialized views for expensive aggregations that are queried frequently but change slowly
- Always add a unique index when using `REFRESH CONCURRENTLY` in PostgreSQL
- Schedule refreshes aligned with data pipeline completion — not on a fixed clock
- Name materialized views clearly to distinguish them from regular views — `mv_` prefix is common
- Monitor refresh duration — long refreshes on large views may need incremental refresh strategies
- Do not use materialized views for data that must be real-time — use regular views instead