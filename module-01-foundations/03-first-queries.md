# 1.3 Your First Queries

**TL;DR:** BigQuery SQL is standard SQL. Most queries you already know work out of the box — but a few BigQuery behaviours will surprise you if you don't know them upfront.

---

## What you'll learn

- Run a SELECT query against a public BigQuery dataset
- Filter, group, and sort results
- Understand the LIMIT trap and bytes processed

---

## Your first query

Open the BigQuery Console and run this against a public dataset:

```sql
-- Top programming languages by repository count
SELECT
  language.name        AS language,
  COUNT(*) AS repo_count
FROM `bigquery-public-data.github_repos.languages`,
  UNNEST(language) AS language
GROUP BY language
ORDER BY repo_count DESC
LIMIT 20;
```

The `UNNEST` flattens an array column — a common BigQuery pattern you'll use often.

---

## The LIMIT trap

This is the most common BigQuery misconception:

```sql
-- This does NOT limit how much data is scanned.
-- BigQuery scans the full column, then returns 10 rows.
SELECT event_name FROM `project.analytics.events` LIMIT 10;
```

**LIMIT reduces rows returned, not bytes scanned.** If the `events` table is 500 GB, this query still scans 500 GB of `event_name`. You pay for the full scan.

To actually reduce cost, filter with `WHERE` on a partition column (covered in Module 2).

---

## Bytes processed

Before you run any query in the Console, look at the validator in the top-right corner. It shows estimated bytes processed.

```
This query will process 2.3 GB when run.
```

That estimate is free — BigQuery does a metadata-only check. Use it as a habit before running unfamiliar queries.

---

## Common patterns

```sql
-- Filter by date (efficient if the table is partitioned on order_date)
SELECT
  order_id,
  customer_id,
  total_amount
FROM `project.sales.orders`
WHERE order_date >= '2024-01-01'
  AND order_date < '2024-02-01';

-- Aggregate with conditional counts
SELECT
  product_category,
  COUNT(*) AS total_orders,
  COUNTIF(status = 'returned') AS returned_orders,
  ROUND(COUNTIF(status = 'returned') / COUNT(*) * 100, 2) AS return_rate_pct
FROM `project.sales.orders`
GROUP BY product_category
ORDER BY total_orders DESC;
```

---

## BigQuery-specific SQL features to know early

| Feature | Example |
|---------|---------|
| `COUNTIF` | `COUNTIF(status = 'active')` |
| `DATE_TRUNC` | `DATE_TRUNC(order_date, MONTH)` |
| `SAFE_DIVIDE` | `SAFE_DIVIDE(revenue, sessions)` — returns NULL instead of dividing by zero |
| `ARRAY_AGG` | Collect values into an array |
| `STRUCT` | Nest related columns |

---

## QE Tip

Always check bytes processed before approving a query pattern for production. A query that scans 5 TB on every dashboard refresh costs ~$25 per run at on-demand pricing. Spot this in code review, not on the invoice.

---

**Key Takeaway:** LIMIT saves you nothing in BigQuery — only WHERE filters on partition columns reduce scan cost. Check bytes processed before running unfamiliar queries.

**→ Next:** [2.1 Why Partition?](../module-02-partitioning-clustering/01-why-partition.md)
