# 5.3 Snapshots & Time Travel

**TL;DR:** BigQuery keeps a 7-day history of every table's state. You can query any point in that window — and restore data that was accidentally deleted or overwritten.

---

## What you'll learn

- How BigQuery's time travel works
- How to query historical table state with FOR SYSTEM_TIME AS OF
- How to restore accidentally modified or deleted data

---

## How time travel works

BigQuery automatically retains every version of a table's data for 7 days (the default; configurable up to 7 days). No action required on your part — it's always on.

This means you can query a table as it existed at any point in the last 7 days:

```sql
-- Query the events table as it was 24 hours ago
SELECT COUNT(*) FROM `project.analytics.events`
FOR SYSTEM_TIME AS OF TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR);

-- Query as of a specific timestamp
SELECT * FROM `project.analytics.events`
FOR SYSTEM_TIME AS OF '2024-06-15 09:00:00 UTC'
WHERE event_date = '2024-06-14';
```

---

## Recovering from an accidental DELETE or overwrite

If a pipeline accidentally wiped a partition:

```sql
-- Step 1: Find the right timestamp (just before the accident)
-- Check INFORMATION_SCHEMA.JOBS to find when the bad write happened
SELECT job_id, creation_time, query
FROM `region-eu.INFORMATION_SCHEMA.JOBS`
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 2 HOUR)
  AND (LOWER(query) LIKE '%delete%' OR LOWER(query) LIKE '%truncate%' OR LOWER(query) LIKE '%insert overwrite%')
ORDER BY creation_time DESC;

-- Step 2: Restore the affected partition from before the accident
INSERT INTO `project.analytics.events`
SELECT *
FROM `project.analytics.events`
FOR SYSTEM_TIME AS OF '2024-06-15 08:55:00 UTC'  -- just before the bad write
WHERE event_date = '2024-06-14';                  -- the affected partition
```

---

## Table snapshots

For snapshots older than 7 days, or to create a named checkpoint before a risky operation, use explicit table snapshots:

```sql
-- Create a snapshot before a risky migration
CREATE SNAPSHOT TABLE `project.analytics.events_snapshot_20240615`
CLONE `project.analytics.events`
OPTIONS (expiration_timestamp = TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 30 DAY));
```

A snapshot is a zero-cost clone at the time of creation — it shares storage with the original table (copy-on-write). It only consumes additional storage as the original table diverges over time.

Query a snapshot just like a regular table:

```sql
SELECT COUNT(*) FROM `project.analytics.events_snapshot_20240615`;
```

---

## Table clones

A clone is an editable copy of a table at a point in time:

```sql
-- Create an editable clone for testing a migration
CREATE TABLE `project.analytics.events_migration_test`
CLONE `project.analytics.events`
  FOR SYSTEM_TIME AS OF '2024-06-15 00:00:00 UTC';
```

Clones use copy-on-write storage — changes to the clone don't affect the original. Use them to test destructive operations safely.

---

## Configuring time travel window

The default is 7 days. You can reduce it to save storage costs on high-churn tables:

```sql
ALTER TABLE `project.analytics.events`
SET OPTIONS (max_time_travel_hours = 48);  -- 2-day window instead of 7
```

Reducing time travel also reduces storage used by historical versions. For compliance-sensitive tables, keep the full 7 days.

---

## QE Tip

Before any destructive operation (DELETE, TRUNCATE, INSERT OVERWRITE), record the current timestamp in your pipeline logs. If something goes wrong, you'll know exactly which timestamp to use in `FOR SYSTEM_TIME AS OF` to get the pre-operation state. An unrecorded timestamp means you're guessing.

---

**Key Takeaway:** BigQuery keeps 7 days of history automatically. Use `FOR SYSTEM_TIME AS OF` to query past state, create snapshots before risky changes, and clones to test migrations without touching production data.

**→ Next:** [5.4 Safe Migration Patterns](04-safe-migrations.md)
