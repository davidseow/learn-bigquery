# 3.4 Materialized Views

**TL;DR:** A materialized view pre-computes and stores the result of a query. Queries against it scan the stored result instead of the underlying tables — which can be orders of magnitude cheaper.

---

## What you'll learn

- How materialized views differ from regular views
- How to create and use them
- Smart refresh strategies and limitations

---

## Views vs materialized views

| | View | Materialized View |
|-|------|------------------|
| Storage | None — runs SQL every time | Stores pre-computed results |
| Cost per query | Full scan of underlying tables | Scan of the stored result (much smaller) |
| Freshness | Always fresh | Refreshes on a schedule or incrementally |
| SQL restrictions | None | Significant restrictions (see below) |

---

## Creating a materialized view

```sql
CREATE MATERIALIZED VIEW `project.analytics.daily_revenue_mv`
OPTIONS (
  enable_refresh = TRUE,
  refresh_interval_minutes = 60  -- refresh every hour
)
AS
SELECT
  DATE_TRUNC(order_date, DAY) AS order_day,
  product_category,
  country,
  SUM(revenue_usd) AS total_revenue,
  COUNT(DISTINCT order_id) AS order_count
FROM `project.sales.orders`
WHERE order_date >= '2023-01-01'
GROUP BY 1, 2, 3;
```

Now any query that matches this pattern (same aggregation, same filters or narrower) can be answered from the materialized view instead of the raw `orders` table.

---

## Intelligent query rewrite

BigQuery automatically rewrites queries to use the materialized view when it can serve the result. You don't need to change anything in your existing queries:

```sql
-- This query runs against orders (200 GB)
-- BigQuery detects that daily_revenue_mv can serve it
-- and transparently scans the MV instead (~50 MB)
SELECT
  order_day,
  country,
  SUM(total_revenue) AS total_revenue
FROM `project.analytics.daily_revenue_mv`
WHERE order_day >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY 1, 2;
```

You can verify a rewrite happened in job statistics: look for `referencedMaterializedViews` in the job detail.

---

## Incremental refresh

If the underlying table is partitioned, BigQuery can refresh the materialized view incrementally — only reprocessing changed partitions instead of the whole table:

```sql
CREATE MATERIALIZED VIEW `project.analytics.daily_active_users_mv`
AS
SELECT
  event_date,
  country,
  COUNT(DISTINCT user_id) AS dau
FROM `project.analytics.events`
GROUP BY 1, 2;
-- Automatically incremental because events is partitioned by event_date
```

This makes refreshes cheap even on multi-TB tables.

---

## Limitations to know

Materialized views in BigQuery have restrictions on what SQL they can contain:

- No subqueries in the SELECT list
- No UNION or EXCEPT
- Joins are allowed but with restrictions (inner joins on partitioned tables work; complex joins may not qualify for incremental refresh)
- Window functions are not supported
- Outer queries cannot reference the MV as a subquery in some cases

When in doubt, create it and test: BigQuery will error if your SQL isn't supported.

---

## When to use materialized views

Good use cases:
- Expensive aggregations queried frequently (dashboards, reports)
- COUNT DISTINCT over large event tables
- Pre-joining slow reference tables

Poor use cases:
- One-off queries — the refresh cost outweighs the benefit
- Data that changes every few minutes and needs real-time freshness
- Complex SQL that exceeds MV restrictions

---

## QE Tip

Treat materialized views like code: store their DDL in version control alongside the tables they depend on. When a source table schema changes (a column is renamed, a type changes), the MV silently breaks. Add a post-deploy check that queries the MV and validates it returns expected row counts and types.

---

**Key Takeaway:** Materialized views pre-compute expensive aggregations and BigQuery rewrites queries to use them automatically. Use them for any aggregation queried more than a few times a day.

**→ Next:** [3.5 Slots & Reservations](05-slots-reservations.md)
