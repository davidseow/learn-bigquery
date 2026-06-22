# 2.2 Time-Based Partitioning

**TL;DR:** Partition by a DATE or TIMESTAMP column and BigQuery creates one segment per day (or month/year/hour). This is the most common and impactful partition type.

---

## What you'll learn

- How to create a table with time-based partitioning
- The difference between DAY, MONTH, YEAR, and HOUR granularity
- How partition expiry automatically manages storage costs

---

## Creating a partitioned table

```sql
CREATE TABLE `project.analytics.events`
(
  event_id      STRING    NOT NULL,
  user_id       STRING,
  event_name    STRING,
  event_date    DATE,
  properties    JSON
)
PARTITION BY event_date
OPTIONS (
  partition_expiration_days = 365  -- partitions older than 1 year auto-deleted
);
```

BigQuery will create a new partition for each unique value of `event_date` as data is inserted.

---

## Granularity options

| Granularity | Syntax | Use when |
|-------------|--------|----------|
| Day (default) | `PARTITION BY DATE_TRUNC(event_ts, DAY)` | Event/log tables, most common |
| Month | `PARTITION BY DATE_TRUNC(event_ts, MONTH)` | Lower cardinality data, financial tables |
| Year | `PARTITION BY DATE_TRUNC(event_ts, YEAR)` | Archival tables, very low write volume |
| Hour | `PARTITION BY TIMESTAMP_TRUNC(event_ts, HOUR)` | High-frequency streaming, real-time dashboards |

```sql
-- Partitioning a TIMESTAMP column by day
CREATE TABLE `project.analytics.page_views`
(
  session_id  STRING,
  page_url    STRING,
  viewed_at   TIMESTAMP
)
PARTITION BY DATE(viewed_at);
```

Note: `PARTITION BY DATE(viewed_at)` extracts the date from the timestamp — BigQuery creates one partition per day.

---

## Partition expiry

`partition_expiration_days` is your automated housekeeping. Partitions older than the threshold are deleted without any manual intervention.

```sql
-- Change expiry on an existing table
ALTER TABLE `project.analytics.events`
SET OPTIONS (partition_expiration_days = 180);
```

Without expiry, old partitions accumulate forever. Storage is cheap but not free, and keeping stale data creates compliance risk.

---

## Requiring partition filters

Enforce that queries always filter on the partition column:

```sql
CREATE TABLE `project.analytics.events`
(
  event_id   STRING,
  event_date DATE
)
PARTITION BY event_date
OPTIONS (require_partition_filter = TRUE);
```

With this set, any query that doesn't include a `WHERE` on `event_date` returns an error. This is a guardrail — it prevents accidental full-table scans.

```sql
-- This fails with require_partition_filter = TRUE:
SELECT COUNT(*) FROM `project.analytics.events`;

-- This works:
SELECT COUNT(*) FROM `project.analytics.events`
WHERE event_date = CURRENT_DATE();
```

---

## QE Tip

Enable `require_partition_filter = TRUE` on your largest tables in production. When it trips in a code review or a failing pipeline test, it's catching a query that would otherwise scan the entire table — a free cost guardrail. Add it to your table creation standard for any table expected to exceed 10 GB.

---

**Key Takeaway:** Partition by your most common time filter column, set an expiry, and consider requiring partition filters on high-volume tables.

**→ Next:** [2.3 Other Partition Types](03-other-partition-types.md)
