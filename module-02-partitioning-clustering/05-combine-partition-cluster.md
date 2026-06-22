# 2.5 Partition + Cluster Together

**TL;DR:** Combining partitioning and clustering gives you two layers of data pruning. Partition handles the time/range dimension; clustering handles everything within it. Use both on production tables.

---

## What you'll learn

- How the two strategies stack on top of each other
- A design pattern for production-grade tables
- Common mistakes when combining them

---

## How they layer

Think of it as two-level filtering:

1. **Partition pruning** eliminates entire partitions (e.g., all days except June 1–7)
2. **Clustering pruning** then eliminates blocks within the remaining partitions (e.g., only UK rows in those 7 day-partitions)

Together they can reduce a multi-TB scan to gigabytes or megabytes.

```
events (500 GB total, partitioned by day, clustered by country/platform)

Query: WHERE event_date = '2024-06-01' AND country = 'GB' AND platform = 'ios'

Step 1 — Partition pruning:
  Only opens the 2024-06-01 partition (say, 2 GB out of 500 GB)

Step 2 — Clustering pruning:
  Within that partition, skips blocks where country ≠ 'GB' or platform ≠ 'ios'
  Reads maybe 50 MB instead of 2 GB
```

---

## The production table template

This pattern covers most analytics and event tables:

```sql
CREATE TABLE `project.analytics.events`
(
  event_id      STRING    NOT NULL,
  event_date    DATE      NOT NULL,
  user_id       STRING,
  session_id    STRING,
  event_name    STRING,
  country       STRING,
  platform      STRING,   -- 'ios', 'android', 'web'
  properties    JSON
)
PARTITION BY event_date
CLUSTER BY country, platform, event_name
OPTIONS (
  partition_expiration_days = 400,
  require_partition_filter  = TRUE
);
```

---

## Inserting into a partitioned+clustered table

No special syntax needed — just insert normally:

```sql
INSERT INTO `project.analytics.events`
SELECT
  event_id,
  DATE(event_timestamp) AS event_date,
  user_id,
  session_id,
  event_name,
  country,
  platform,
  properties
FROM `project.raw.events_landing`
WHERE DATE(event_timestamp) = CURRENT_DATE();
```

BigQuery routes each row to the correct partition and sorts it into the right cluster blocks automatically.

---

## Common mistakes

**Clustering on a column you then wrap in a function:**
```sql
-- This does NOT use clustering:
WHERE LOWER(country) = 'gb'

-- This does:
WHERE country = 'GB'
```
Store values in canonical form (consistent casing) so filters can prune without transformations.

**Choosing partition granularity too fine for the data volume:**
Hourly partitioning on a table that only receives 100 rows/hour creates thousands of tiny partitions, which can slow metadata operations. Use daily unless you have a strong reason for hourly.

---

## QE Tip

Validate the combined strategy with a query that matches your most common production access pattern. Check job statistics for `totalBytesProcessed` vs `totalBytesBilled`. If they diverge significantly, caching is involved — run with `CREATE OR REPLACE TABLE ... AS SELECT` to force a fresh scan and get a clean benchmark.

---

**Key Takeaway:** Partition on time, cluster on your most common filter columns. Together they give you compounding pruning — use both on any production table above 10 GB.

**→ Next:** [2.6 QE: Testing Partitioned Tables](06-qe-testing-partitions.md)
