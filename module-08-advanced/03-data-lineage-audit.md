# 8.3 Data Lineage & Audit Logs

**TL;DR:** BigQuery records every query, every table read, and every data modification. This audit trail lets you answer: "Who ran this query?", "What tables feed this table?", and "When was this data last changed?"

---

## What you'll learn

- How to query BigQuery audit logs via INFORMATION_SCHEMA
- Tracing data lineage automatically
- Building an access audit for compliance

---

## INFORMATION_SCHEMA.JOBS: your audit backbone

Every BigQuery job (query, load, export) is recorded. Query it:

```sql
-- Who has been querying the customers table in the last 7 days?
SELECT
  user_email,
  COUNT(*) AS query_count,
  SUM(total_bytes_processed) / POW(1024, 3) AS gb_scanned
FROM `region-eu.INFORMATION_SCHEMA.JOBS`
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  AND LOWER(query) LIKE '%project.crm.customers%'
  AND job_type = 'QUERY'
GROUP BY user_email
ORDER BY query_count DESC;
```

---

## Finding the blast radius of a column change

Before renaming a column, find all queries that referenced it in the last 30 days:

```sql
SELECT
  user_email,
  creation_time,
  SUBSTR(query, 0, 300) AS query_snippet
FROM `region-eu.INFORMATION_SCHEMA.JOBS`
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  AND LOWER(query) LIKE '%old_column_name%'
  AND referenced_tables IS NOT NULL
ORDER BY creation_time DESC
LIMIT 50;
```

This tells you which users and pipelines would break if you rename the column.

---

## Data lineage: what feeds this table?

`referenced_tables` in `INFORMATION_SCHEMA.JOBS` tells you what tables a query read from:

```sql
-- Find all source tables that feed into project.analytics.orders_daily
SELECT
  ARRAY_TO_STRING(ARRAY_AGG(DISTINCT CONCAT(ref.projectId, '.', ref.datasetId, '.', ref.tableId)), '\n') AS source_tables,
  COUNT(*) AS times_run
FROM `region-eu.INFORMATION_SCHEMA.JOBS`,
  UNNEST(referenced_tables) AS ref
WHERE destination_table.datasetId = 'analytics'
  AND destination_table.tableId = 'orders_daily'
  AND creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
GROUP BY 1
ORDER BY times_run DESC
LIMIT 10;
```

---

## Cloud Audit Logs for deeper tracing

For full data access logs (including table reads by external tools), enable Cloud Audit Logs in Google Cloud Console:

**Project → IAM & Admin → Audit Logs → BigQuery → enable Data Read and Data Write**

These logs capture every table access — including reads that don't use BigQuery jobs (e.g., Storage API reads from Looker, Spark, etc.). They land in Cloud Logging and can be exported to BigQuery:

```sql
-- Query audit logs exported to BigQuery
SELECT
  timestamp,
  proto_payload.audit_log.authentication_info.principal_email AS user,
  proto_payload.audit_log.resource_name AS table_accessed,
  proto_payload.audit_log.method_name AS operation
FROM `project.audit_logs.cloudaudit_googleapis_com_data_access`
WHERE DATE(timestamp) = CURRENT_DATE()
  AND proto_payload.audit_log.resource_name LIKE '%crm.customers%'
ORDER BY timestamp DESC;
```

---

## Automated data lineage with Dataplex

Google Dataplex provides automated data lineage across BigQuery, Dataflow, and other Google services:

```bash
# Enable Data Lineage API
gcloud services enable datalineage.googleapis.com

# Lineage is automatically captured for BigQuery, Dataflow, Dataproc
# View it in the Cloud Console → Dataplex → Data Lineage
```

Dataplex draws the lineage graph automatically — no manual tagging required. Useful for compliance audits and impact analysis.

---

## Building a compliance report

For GDPR/data governance: who accessed PII tables in the last 30 days?

```sql
SELECT
  DATE(creation_time) AS access_date,
  user_email,
  COUNT(*) AS access_count,
  ARRAY_AGG(DISTINCT ref.tableId) AS tables_accessed
FROM `region-eu.INFORMATION_SCHEMA.JOBS`,
  UNNEST(referenced_tables) AS ref
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  AND ref.tableId IN ('customers', 'user_profiles', 'payment_methods')  -- PII tables
  AND ref.datasetId = 'crm'
GROUP BY 1, 2
ORDER BY access_date DESC, access_count DESC;
```

---

## QE Tip

Run the blast-radius query (who references a column/table) as part of your change review process, not as an afterthought. Automate it: whenever a PR touches a BigQuery schema, a bot comments with the INFORMATION_SCHEMA query output showing all consumers from the last 30 days. Reviewers can then make an informed decision about whether the change is safe.

---

**Key Takeaway:** INFORMATION_SCHEMA.JOBS is a rich audit log of all BigQuery activity. Use it for impact analysis before changes, access auditing for compliance, and lineage tracing for debugging data issues.

**→ Next:** [8.4 Streaming vs Batch Ingestion](04-streaming-vs-batch.md)
