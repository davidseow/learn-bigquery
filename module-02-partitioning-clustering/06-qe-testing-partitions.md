# 2.6 QE: Testing Partitioned Tables

**TL;DR:** Partitioning only helps if pruning actually fires. These tests let you verify your table design is working in code review and CI — before it hits production.

---

## What you'll learn

- How to confirm partition pruning is active
- How to detect partition health issues
- Automated checks you can run in a pipeline

---

## Test 1: Verify pruning fires

Use `INFORMATION_SCHEMA.JOBS` to inspect bytes processed after a query run:

```sql
-- Run your query, then check its job stats
SELECT
  job_id,
  total_bytes_processed,
  total_bytes_billed,
  query
FROM `region-eu.INFORMATION_SCHEMA.JOBS`
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 10 MINUTE)
  AND statement_type = 'SELECT'
ORDER BY creation_time DESC
LIMIT 10;
```

Compare `total_bytes_processed` against the full table size. If you filtered on 7 days of a 3-year table, you should see ~2% of the total bytes, not 100%.

---

## Test 2: Check for empty or skewed partitions

A healthy partitioned table has reasonably consistent partition sizes. Sudden drops or spikes indicate pipeline issues:

```sql
SELECT
  partition_id,
  total_rows,
  total_logical_bytes,
  last_modified_time
FROM `project.analytics.INFORMATION_SCHEMA.PARTITIONS`
WHERE table_name = 'events'
  AND partition_id != '__NULL__'
ORDER BY partition_id DESC
LIMIT 30;
```

Look for:
- Partitions with 0 rows (missing data — pipeline may have failed)
- Partitions with 10× more rows than neighbours (duplicate load — pipeline may have double-written)
- `last_modified_time` more than 24 hours ago for "today's" partition (stale data)

---

## Test 3: Assert the partition column is never NULL

If the partition column contains NULLs, those rows go into the `__NULL__` partition and are never pruned:

```sql
-- Run as a data quality assertion in your pipeline
SELECT
  COUNT(*) AS null_partition_rows
FROM `project.analytics.events`
WHERE event_date IS NULL;

-- Fail the pipeline if this returns > 0
```

In dbt, this is a built-in test:

```yaml
# schema.yml
models:
  - name: events
    columns:
      - name: event_date
        tests:
          - not_null
```

---

## Test 4: Validate partition expiry is working

```sql
-- Find partitions older than your expiry threshold (365 days)
-- If this returns rows, your expiry setting isn't working
SELECT
  partition_id,
  last_modified_time
FROM `project.analytics.INFORMATION_SCHEMA.PARTITIONS`
WHERE table_name = 'events'
  AND PARSE_DATE('%Y%m%d', partition_id) < DATE_SUB(CURRENT_DATE(), INTERVAL 365 DAY)
  AND partition_id NOT IN ('__NULL__', '__UNPARTITIONED__');
```

---

## Putting it in CI

Wrap these checks in SQL scripts or dbt tests and run them as part of your data pipeline:

```bash
# Example: fail if null partitions exist
bq query --nouse_legacy_sql '
  SELECT
    IF(COUNT(*) = 0, "PASS", ERROR("NULL partition rows detected")) AS result
  FROM project.analytics.events
  WHERE event_date IS NULL
'
```

---

## QE Tip

Add partition health checks to your pipeline's post-load validation step, not just as one-off audits. Catching a missing partition the day after a load failure is much cheaper than discovering it when a business report is wrong.

---

**Key Takeaway:** Partitioning is only as good as the checks around it. Test that pruning fires, partitions are balanced, the partition column is non-null, and old partitions expire as expected.

**→ Next:** [3.1 How Pricing Works](../module-03-cost-optimisation/01-pricing-model.md)
