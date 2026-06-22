# 3.6 QE: Cost Regression Testing

**TL;DR:** A query that costs $0.10 today can cost $100 tomorrow when the underlying table grows. Cost regression testing catches this before the bill arrives.

---

## What you'll learn

- How to set query byte budgets in SQL and code
- How to track cost trends over time using INFORMATION_SCHEMA
- How to add cost gates to your CI pipeline

---

## The problem: data grows, queries don't change

A query is written when the events table has 50 GB. It works fine. Six months later the table is 5 TB and the same query now scans 10× more. No one notices until the monthly bill arrives.

The fix: test query cost the same way you test query correctness.

---

## Method 1: Maximum bytes billed

BigQuery lets you set a hard limit on how much a query can scan. If it exceeds the limit, the query fails instead of running:

```python
from google.cloud import bigquery

client = bigquery.Client()

job_config = bigquery.QueryJobConfig(
    maximum_bytes_billed=10 * 1024**3  # 10 GB limit
)

try:
    job = client.query(
        "SELECT ... FROM `project.analytics.events` WHERE event_date = CURRENT_DATE()",
        job_config=job_config
    )
    result = job.result()
except Exception as e:
    print(f"Query exceeded byte budget: {e}")
```

In SQL via the Console or CLI, you can set this per-session using query settings.

---

## Method 2: Dry run in CI

Add a pre-run cost check to your pipeline:

```python
def assert_query_within_budget(client, sql, max_gb=50):
    job_config = bigquery.QueryJobConfig(dry_run=True, use_query_cache=False)
    job = client.query(sql, job_config=job_config)
    
    gb_scanned = job.total_bytes_processed / 1024**3
    cost_usd = gb_scanned / 1024 * 5  # rough on-demand estimate
    
    assert gb_scanned <= max_gb, (
        f"Query scans {gb_scanned:.1f} GB (${cost_usd:.2f}), "
        f"exceeds budget of {max_gb} GB"
    )
```

Run this in your test suite on every PR that touches SQL. If a developer removes a partition filter by accident, the dry run catches it before merge.

---

## Method 3: Track cost trends in INFORMATION_SCHEMA

Build a weekly report on your most expensive queries:

```sql
-- Top 10 most expensive query patterns, last 7 days
SELECT
  REGEXP_REPLACE(query, r'\d{4}-\d{2}-\d{2}', 'DATE') AS query_template,
  COUNT(*) AS executions,
  ROUND(SUM(total_bytes_processed) / POW(1024, 4), 2) AS total_tb,
  ROUND(AVG(total_bytes_processed) / POW(1024, 3), 2) AS avg_gb_per_run,
  ROUND(SUM(total_bytes_processed) / POW(1024, 4) * 5, 2) AS est_cost_usd
FROM `region-eu.INFORMATION_SCHEMA.JOBS`
WHERE
  creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND job_type = 'QUERY'
  AND state = 'DONE'
  AND error_result IS NULL
GROUP BY 1
ORDER BY total_tb DESC
LIMIT 10;
```

The `REGEXP_REPLACE` normalises date literals so the same query with different dates is grouped together.

---

## Method 4: Alert on anomalous daily spend

```sql
-- Compare yesterday's total bytes to the 30-day average
-- Alert if yesterday was > 2× average
WITH daily_bytes AS (
  SELECT
    DATE(creation_time) AS run_date,
    SUM(total_bytes_processed) AS bytes_processed
  FROM `region-eu.INFORMATION_SCHEMA.JOBS`
  WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 31 DAY)
    AND job_type = 'QUERY'
  GROUP BY 1
)
SELECT
  run_date,
  bytes_processed / POW(1024, 4) AS tb_processed,
  AVG(bytes_processed) OVER (
    ORDER BY run_date
    ROWS BETWEEN 30 PRECEDING AND 1 PRECEDING
  ) / POW(1024, 4) AS avg_tb_30d,
  bytes_processed / AVG(bytes_processed) OVER (
    ORDER BY run_date
    ROWS BETWEEN 30 PRECEDING AND 1 PRECEDING
  ) AS ratio_vs_avg
FROM daily_bytes
ORDER BY run_date DESC;
```

---

## QE Tip

Treat a query's byte cost as a non-functional requirement, like response time. Add it to your PR template: "Estimated bytes scanned: ___. Is this proportional to the data needed?" Make the author fill it in — the act of checking creates the habit.

---

**Key Takeaway:** Use `maximum_bytes_billed` to hard-cap queries in production, dry-run checks in CI to catch regressions before merge, and weekly INFORMATION_SCHEMA reports to spot trends before they become incidents.

**→ Next:** [4.1 Why Data Quality Matters](../module-04-quality-engineering/01-why-data-quality.md)
