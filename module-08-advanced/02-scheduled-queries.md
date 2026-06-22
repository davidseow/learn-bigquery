# 8.2 Scheduled Queries & Workflows

**TL;DR:** BigQuery Scheduled Queries run SQL on a cron schedule — useful for refreshing summary tables, running daily data quality checks, and creating snapshots. For multi-step orchestration, pair them with Cloud Workflows or Composer.

---

## What you'll learn

- How to create a scheduled query in BigQuery
- Error handling and monitoring for scheduled queries
- When to use scheduled queries vs a proper orchestration tool

---

## What a scheduled query does

A scheduled query runs a SQL statement on a recurring schedule (every hour, daily at 06:00 UTC, etc.) under a specified service account. The output can be written to a table.

Common uses:
- Refreshing daily summary/reporting tables
- Running data quality checks and writing results to a monitoring table
- Creating snapshots before pipeline runs
- Incrementally loading data from one dataset to another

---

## Creating a scheduled query (Console)

In the BigQuery Console:
1. Write your query
2. Click "Schedule" → "Create new schedule"
3. Set: name, schedule (cron expression or natural language), destination table, service account

---

## Creating with the bq CLI

```bash
bq mk \
  --transfer_config \
  --target_dataset=analytics \
  --display_name='daily_active_users_refresh' \
  --schedule='every 24 hours' \
  --params='{
    "query": "CREATE OR REPLACE TABLE project.analytics.dau AS SELECT event_date, COUNT(DISTINCT user_id) AS dau FROM project.analytics.events WHERE event_date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY) GROUP BY 1",
    "destination_table_name_template": "dau",
    "write_disposition": "WRITE_TRUNCATE"
  }' \
  --data_source=scheduled_query \
  --service_account_name=bq-scheduled@project.iam.gserviceaccount.com
```

---

## Parameterised scheduled queries

Use built-in parameters to make queries dynamic:

```sql
-- @run_date and @run_time are automatically provided by the scheduler
INSERT INTO `project.analytics.daily_summary`
SELECT
  @run_date AS report_date,
  product_category,
  SUM(revenue_usd) AS revenue,
  COUNT(order_id) AS orders
FROM `project.sales.orders`
WHERE order_date = DATE_SUB(@run_date, INTERVAL 1 DAY)
GROUP BY 1, 2;
```

`@run_date` is the date the job was scheduled for (not necessarily today — useful for backfills). `@run_time` is the scheduled execution timestamp.

---

## Monitoring scheduled queries

Check execution history:

```sql
SELECT
  display_name,
  run_time,
  state,
  error_status.message AS error_message
FROM `region-eu.INFORMATION_SCHEMA.SCHEDULED_QUERY_TIMELINE`
WHERE run_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
ORDER BY run_time DESC;
```

Set up email notifications for failures in the scheduled query settings.

---

## When scheduled queries are not enough

Scheduled queries are single-SQL-statement jobs. They don't support:
- Dependencies between queries (run B only if A succeeded)
- Dynamic parameters based on previous step output
- Retry logic more sophisticated than "retry N times"
- Cross-project or cross-service orchestration

For these, use:
- **Cloud Composer (managed Airflow):** Full DAG-based orchestration, Python, rich ecosystem
- **Cloud Workflows:** Serverless, JSON/YAML steps, good for simple multi-step flows
- **Dataform:** SQL-only orchestration, like dbt but native to Google Cloud

---

## Dataform: SQL workflow orchestration

Dataform is Google's native alternative to dbt — manages SQL pipelines with dependency resolution:

```js
// definitions/orders_daily.sqlx
config {
  type: "table",
  schema: "analytics",
  partitionBy: "order_date",
  clusterBy: ["country", "product_category"]
}

SELECT
  DATE(order_timestamp) AS order_date,
  country,
  product_category,
  SUM(total_usd) AS revenue
FROM ${ref("orders_raw")}        -- Dataform resolves the dependency
GROUP BY 1, 2, 3
```

Dataform runs in the Google Cloud Console with Git integration, making it easier to version-control SQL pipelines without a separate tool.

---

## QE Tip

Treat scheduled query failures the same as production incidents: set up Pub/Sub notifications for failures, route them to your alerting system (PagerDuty, Slack), and have an on-call response plan. A scheduled query silently failing at 06:00 means your 09:00 dashboard is serving stale data — that's an incident, not a background task.

---

**Key Takeaway:** Scheduled queries handle simple recurring SQL jobs. For multi-step pipelines with dependencies, use Cloud Composer, Dataform, or Cloud Workflows. Always monitor scheduled query failures as production signals, not background noise.

**→ Next:** [8.3 Data Lineage & Audit Logs](03-data-lineage-audit.md)
