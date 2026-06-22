# 4.5 Monitoring Freshness & Completeness

**TL;DR:** Know when your data is late or missing before your users do. BigQuery gives you the metadata to detect both — without scanning any actual row data.

---

## What you'll learn

- How to check when a partition was last updated
- How to detect missing partitions
- How to build a freshness alert using INFORMATION_SCHEMA

---

## Checking partition freshness

`INFORMATION_SCHEMA.PARTITIONS` gives you metadata about every partition without scanning the data:

```sql
SELECT
  table_name,
  partition_id,
  total_rows,
  last_modified_time
FROM `project.analytics.INFORMATION_SCHEMA.PARTITIONS`
WHERE table_name = 'events'
ORDER BY partition_id DESC
LIMIT 5;
```

`last_modified_time` tells you when data was last written to that partition. If today's partition hasn't been modified since yesterday, the pipeline has stopped loading.

---

## Freshness alert: detect stale today partition

```sql
-- Returns a row if today's events partition is more than 4 hours stale
SELECT
  partition_id,
  last_modified_time,
  TIMESTAMP_DIFF(CURRENT_TIMESTAMP(), last_modified_time, HOUR) AS hours_since_update
FROM `project.analytics.INFORMATION_SCHEMA.PARTITIONS`
WHERE table_name = 'events'
  AND partition_id = FORMAT_DATE('%Y%m%d', CURRENT_DATE())
  AND TIMESTAMP_DIFF(CURRENT_TIMESTAMP(), last_modified_time, HOUR) > 4;
```

No rows returned = everything is fresh. Rows returned = stale partition — fire an alert.

---

## Completeness: detecting missing partitions

For daily partitioned tables, there should be one partition per day. A missing partition means a pipeline didn't run:

```sql
-- Find dates in the last 30 days that have no partition
WITH expected_dates AS (
  SELECT DATE_SUB(CURRENT_DATE(), INTERVAL n DAY) AS expected_date
  FROM UNNEST(GENERATE_ARRAY(1, 30)) AS n
),
actual_partitions AS (
  SELECT PARSE_DATE('%Y%m%d', partition_id) AS partition_date
  FROM `project.analytics.INFORMATION_SCHEMA.PARTITIONS`
  WHERE table_name = 'events'
    AND partition_id NOT IN ('__NULL__', '__UNPARTITIONED__')
)
SELECT expected_date AS missing_date
FROM expected_dates
LEFT JOIN actual_partitions ON expected_date = partition_date
WHERE partition_date IS NULL
ORDER BY missing_date;
```

---

## Row count completeness

For tables with a known expected volume, compare actual vs expected:

```sql
-- Flag partitions where row count dropped more than 20% vs the 7-day average
SELECT
  partition_id,
  total_rows,
  avg_7d_rows,
  ROUND(total_rows / avg_7d_rows, 2) AS ratio
FROM (
  SELECT
    partition_id,
    total_rows,
    AVG(total_rows) OVER (
      ORDER BY partition_id
      ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING
    ) AS avg_7d_rows
  FROM `project.analytics.INFORMATION_SCHEMA.PARTITIONS`
  WHERE table_name = 'events'
    AND partition_id != '__NULL__'
)
WHERE ratio < 0.8
ORDER BY partition_id DESC;
```

---

## Building a monitoring dashboard

Schedule the above queries using BigQuery Scheduled Queries (covered in Module 8) or a Cloud Scheduler job to run hourly. Write results to a `data_quality_alerts` table, then connect it to Looker Studio or Grafana for visibility.

A minimal alert table:

```sql
CREATE TABLE `project.monitoring.data_quality_alerts`
(
  check_name    STRING,
  table_ref     STRING,
  alert_message STRING,
  severity      STRING,  -- 'WARNING', 'CRITICAL'
  checked_at    TIMESTAMP,
  resolved      BOOL
);
```

---

## dbt freshness checks

If you use dbt, it has built-in source freshness checking:

```yaml
# sources.yml
sources:
  - name: raw
    tables:
      - name: events_landing
        freshness:
          warn_after:  {count: 4,  period: hour}
          error_after: {count: 12, period: hour}
        loaded_at_field: event_timestamp
```

```bash
dbt source freshness
```

dbt compares `MAX(event_timestamp)` against the current time and raises warn/error if the gap exceeds the threshold.

---

## QE Tip

Freshness and completeness checks are most valuable when they run *before* downstream consumers query the data. Schedule them to run 30 minutes after each pipeline is expected to complete — not after business hours when someone has already noticed something is wrong.

---

**Key Takeaway:** Use `INFORMATION_SCHEMA.PARTITIONS` to check freshness and detect missing partitions without scanning data. Schedule these checks to run before downstream consumers, not after.

**→ Next:** [5.1 Schema Evolution Strategies](../module-05-derisking-changes/01-schema-evolution.md)
