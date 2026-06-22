# 2.3 Other Partition Types

**TL;DR:** When you don't have a date column, BigQuery offers integer range partitioning and ingestion-time partitioning as alternatives.

---

## What you'll learn

- Integer range partitioning for numeric IDs or buckets
- Ingestion-time partitioning when no natural time column exists
- Which type to choose for your table

---

## Integer range partitioning

Use this when your data has a numeric column you filter on frequently — customer IDs, product IDs, or score buckets.

```sql
CREATE TABLE `project.sales.orders`
(
  order_id      INT64  NOT NULL,
  customer_id   INT64,
  total_amount  NUMERIC,
  status        STRING
)
PARTITION BY RANGE_BUCKET(customer_id, GENERATE_ARRAY(0, 10000000, 100000));
-- Creates 100 buckets: 0-99999, 100000-199999, ..., 9900000-9999999
```

`GENERATE_ARRAY(start, end, step)` defines the bucket boundaries. Rows where `customer_id` falls outside the range go into an `__NULL__` or `__UNPARTITIONED__` partition.

Pruning works the same way:

```sql
-- Only scans the bucket containing customer IDs 200000-299999
SELECT * FROM `project.sales.orders`
WHERE customer_id BETWEEN 200000 AND 299999;
```

**When to use it:** You frequently filter on a high-cardinality integer column. Example: multi-tenant tables where each tenant queries only their own data.

---

## Ingestion-time partitioning

BigQuery automatically assigns each row to a partition based on when it was loaded — no date column required.

```sql
CREATE TABLE `project.logs.raw_events`
(
  event_id   STRING,
  event_name STRING,
  payload    JSON
)
PARTITION BY _PARTITIONDATE;
```

`_PARTITIONDATE` is a pseudo-column — BigQuery populates it automatically at load time. You query it like a real column:

```sql
SELECT *
FROM `project.logs.raw_events`
WHERE _PARTITIONDATE = CURRENT_DATE();
```

You can also use `_PARTITIONTIME` for TIMESTAMP granularity.

**Gotcha:** If you backfill historical data today, all of it lands in today's partition. Ingestion-time partitioning reflects *when data arrived*, not *when events happened*. For event-time semantics, use time-based partitioning on an actual event column.

---

## Choosing the right partition type

| Situation | Recommended type |
|-----------|-----------------|
| Table has an event/transaction date | Time-based on that column |
| Table has no date but has a numeric tenant/customer ID | Integer range |
| Table is a raw landing zone, no meaningful time column | Ingestion-time |
| Table is small (< 1 GB) | No partitioning needed |

---

## QE Tip

Ingestion-time partitioning is a red flag in pipelines that do backfills. If a pipeline fails and reruns 3 days of data, all of it lands in today's partition — making time-based filters silently wrong. Test backfill scenarios explicitly. If backfilling is a requirement, use event-time partitioning on an explicit timestamp column from the source.

---

**Key Takeaway:** Use integer range when filtering by numeric ID, use ingestion-time as a last resort for schemaless landing zones — but prefer event-time partitioning whenever the data has a meaningful timestamp.

**→ Next:** [2.4 Clustering](04-clustering.md)
