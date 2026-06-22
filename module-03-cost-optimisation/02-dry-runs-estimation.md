# 3.2 Dry Runs & Cost Estimation

**TL;DR:** BigQuery can tell you exactly how much a query will scan before it runs — for free. Use this before running anything on a large table.

---

## What you'll learn

- How to use dry runs in the Console, CLI, and API
- How to estimate cost from bytes scanned
- Building cost checks into your development workflow

---

## In the BigQuery Console

The Console automatically estimates bytes processed as you write your query. Look for the green checkmark or the bytes estimate in the top-right of the query editor:

```
This query will process 4.2 GB when run.
```

This estimate is free — BigQuery only reads schema and partition metadata, not actual data. It updates as you edit your query.

If the estimate is larger than expected, stop and check:
- Are you filtering on the partition column?
- Are you selecting only the columns you need?
- Is there a missing WHERE clause?

---

## In the CLI with --dry_run

```bash
bq query \
  --nouse_legacy_sql \
  --dry_run \
  'SELECT user_id, event_name FROM `project.analytics.events`
   WHERE event_date = CURRENT_DATE()'
```

Output:
```
Query successfully validated. Assuming the tables are not modified,
running this query will process 1234567890 bytes of data.
```

Convert bytes to GB/TB: divide by 1,073,741,824 (bytes in a GB).

---

## Converting bytes to cost

```
bytes_processed / 1,099,511,627,776 * 5 = cost in USD

Example: 4.2 GB = 4,200,000,000 bytes
4,200,000,000 / 1,099,511,627,776 * 5 = $0.019 (less than 2 cents)

Example: 5 TB = 5,497,558,138,880 bytes
5,497,558,138,880 / 1,099,511,627,776 * 5 = $25.00
```

A quick mental model: $5/TB = $0.005/GB. A 10 GB query costs ~$0.05.

---

## In code (Python client)

```python
from google.cloud import bigquery

client = bigquery.Client()

job_config = bigquery.QueryJobConfig(dry_run=True, use_query_cache=False)

query = """
    SELECT user_id, total_spend
    FROM `project.sales.customer_summary`
    WHERE cohort_month = '2024-06-01'
"""

job = client.query(query, job_config=job_config)

bytes_processed = job.total_bytes_processed
cost_usd = bytes_processed / 1_099_511_627_776 * 5

print(f"Estimated: {bytes_processed / 1e9:.2f} GB, ~${cost_usd:.4f}")
```

This is useful in CI to gate queries that suddenly scan much more than expected.

---

## Cost estimation in INFORMATION_SCHEMA

To see what already ran and what it cost:

```sql
SELECT
  job_id,
  user_email,
  ROUND(total_bytes_processed / POW(1024, 4), 4) AS tb_processed,
  ROUND(total_bytes_processed / POW(1024, 4) * 5, 4) AS estimated_cost_usd,
  query
FROM `region-eu.INFORMATION_SCHEMA.JOBS`
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND job_type = 'QUERY'
ORDER BY total_bytes_processed DESC
LIMIT 20;
```

This shows your top 20 most expensive queries from the last 7 days — a useful audit to run weekly.

---

## QE Tip

Add a dry run check to your code review checklist for any new SQL query that touches a table over 100 GB. A 5-second check in the Console can save you from a query that scans 50 TB when it's promoted to production and runs hourly.

---

**Key Takeaway:** Dry runs are free and instant. Make it a habit to check bytes processed before running any unfamiliar query on a large table — in the Console, CLI, or via the API in CI.

**→ Next:** [3.3 Expensive Query Patterns to Avoid](03-avoid-expensive-queries.md)
