# 8.4 Streaming vs Batch Ingestion

**TL;DR:** BigQuery supports two ingestion modes: batch (load once, low cost) and streaming (continuous, higher cost but real-time). The right choice depends on how fresh your data needs to be.

---

## What you'll learn

- The difference between batch and streaming ingestion
- How to use the Storage Write API (the modern streaming path)
- How to choose the right approach for your use case

---

## Batch ingestion

Data is collected and loaded in chunks — hourly, daily, or triggered by a pipeline.

Methods:
- `bq load` — load from GCS files (CSV, JSON, Parquet, Avro)
- BigQuery Data Transfer Service — scheduled loads from GCS, S3, or SaaS sources
- `LOAD DATA` SQL statement — load from GCS directly in SQL
- Dataflow/Spark batch jobs

```sql
-- Load from GCS using SQL
LOAD DATA INTO `project.analytics.events`
FROM FILES (
  format = 'PARQUET',
  uris = ['gs://my-data-bucket/events/2024/06/21/*.parquet']
);
```

**Cost:** Batch loads from GCS are **free** (no query charge). Storage cost only.

**Latency:** Minutes to hours between events happening and being queryable.

---

## Legacy streaming inserts

The original streaming API (`insertAll`) inserts rows in real-time:

```python
from google.cloud import bigquery

client = bigquery.Client()
errors = client.insert_rows_json(
    'project.analytics.events',
    [{'event_id': '123', 'event_name': 'page_view', 'event_date': '2024-06-21'}]
)
```

**Cost:** ~$0.01 per 200 MB inserted. Can get expensive at high volume.

**Freshness:** Seconds. Data is queryable almost immediately.

**Limitation:** Rows in the streaming buffer (first few minutes) are not available for DML (UPDATE/DELETE). The old API also has limited exactly-once guarantees.

---

## Storage Write API (recommended streaming path)

The modern successor to `insertAll`. Supports three modes:

| Mode | Use when |
|------|---------|
| **Committed** | Exactly-once semantics, data visible immediately after commit |
| **Buffered** | Batch within a stream, commit when ready |
| **Default (legacy)** | Simple, at-least-once delivery |

```python
from google.cloud.bigquery_storage_v1 import BigQueryWriteClient
from google.cloud.bigquery_storage_v1 import types

client = BigQueryWriteClient()

# Create a write stream (committed mode)
parent = f"projects/{project}/datasets/{dataset}/tables/{table}"
write_stream = types.WriteStream()
write_stream.type_ = types.WriteStream.Type.COMMITTED
write_stream = client.create_write_stream(parent=parent, write_stream=write_stream)

# Append rows (using proto format)
# ... (see GCP docs for full proto setup)
```

The Storage Write API is more complex to set up but provides:
- Exactly-once delivery (idempotent via offset tracking)
- Lower cost than legacy streaming inserts
- Better integration with Dataflow

---

## Choosing your ingestion strategy

| Factor | Batch | Streaming (Storage Write API) |
|--------|-------|-------------------------------|
| Data freshness needed | Hours acceptable | Minutes or seconds |
| Cost sensitivity | Very cheap | More expensive per GB |
| Volume | Any | Any |
| Exactly-once required | Yes (batch is idempotent) | Yes with committed mode |
| Use case | ETL, nightly loads, large files | Real-time events, user actions, IoT |

**Default to batch** unless you have a specific freshness requirement. Most analytics use cases don't need data fresher than 30–60 minutes, and batch is dramatically cheaper and simpler to operate.

---

## The hybrid pattern: micro-batch

For many use cases, micro-batch (every 5–15 minutes) is the right middle ground:
- Fresher than daily/hourly batch
- Much cheaper than row-by-row streaming
- Simpler to operate than streaming

```python
# Cloud Scheduler triggers every 15 minutes
# Pub/Sub → Dataflow → BigQuery batch load every 15 min
```

---

## Deduplication for streaming data

Streaming pipelines can produce duplicates (network retries, at-least-once delivery). Deduplicate in BigQuery:

```sql
CREATE OR REPLACE TABLE `project.analytics.events_deduplicated` AS
SELECT * EXCEPT (row_num)
FROM (
  SELECT
    *,
    ROW_NUMBER() OVER (PARTITION BY event_id ORDER BY _PARTITIONTIME ASC) AS row_num
  FROM `project.analytics.events_raw`
)
WHERE row_num = 1;
```

Or use a scheduled MERGE to deduplicate incrementally.

---

## QE Tip

Test your streaming pipeline's exactly-once behaviour explicitly. Send the same event twice (simulating a retry) and verify only one row lands in the final table. Most streaming issues in production are not "no data" failures — they're silent duplicates that inflate counts by 0.1–2% and are impossible to detect without deliberate testing.

---

**Key Takeaway:** Default to batch ingestion — it's free and simple. Use the Storage Write API for true streaming needs. For freshness between 5–60 minutes, micro-batch is often the right compromise. Always test deduplication behaviour explicitly.

---

**You've completed the course.** Return to the [full index](../README.md) to review any module, or explore the topics that are most relevant to your current work.
