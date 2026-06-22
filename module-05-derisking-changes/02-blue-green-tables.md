# 5.2 Blue-Green Table Swaps

**TL;DR:** Build the new version of a table while the old one stays live, validate it thoroughly, then swap with a view — atomically, with zero downtime and an instant rollback option.

---

## What you'll learn

- How to use a view as a stable pointer to the current table
- The blue-green pattern for safe table cutover
- How to roll back in under a minute

---

## The core idea

Consumers point to a **view**, not a table. When you need to change the table, you swap what the view points to. The consumer never knows — they always query the same view name.

```
consumers → view: analytics.events → table: analytics.events_v2  (active)
                                    table: analytics.events_v1  (previous, kept for rollback)
```

---

## Step 1: Give consumers a view, not the raw table

First, create a view that consumers query:

```sql
-- Create the view as a stable pointer
CREATE OR REPLACE VIEW `project.analytics.events` AS
SELECT * FROM `project.analytics.events_current`;
```

Now rename your existing table to `events_current`:
```sql
-- Rename table (BigQuery uses copy+delete for table renames)
CREATE TABLE `project.analytics.events_current`
AS SELECT * FROM `project.analytics.events_old`;
```

Going forward, consumers always query `project.analytics.events` (the view).

---

## Step 2: Build the new table in parallel (green)

While production continues running on `events_current` (blue), build the new version:

```sql
-- Build the new table with the new schema or partitioning
CREATE TABLE `project.analytics.events_next`
PARTITION BY event_date
CLUSTER BY country, platform, event_name
AS
SELECT
  event_id,
  user_id,
  event_name,
  DATE(event_timestamp) AS event_date,  -- new: explicit date column
  country,
  platform,
  event_timestamp
FROM `project.raw.events_all_time`;
```

This can run for hours. It doesn't affect production at all.

---

## Step 3: Validate the new table

Before cutting over, verify the new table is correct:

```sql
-- Row count comparison (should be equal or the difference explained)
SELECT
  (SELECT COUNT(*) FROM `project.analytics.events_current`) AS old_count,
  (SELECT COUNT(*) FROM `project.analytics.events_next`)    AS new_count;

-- Revenue total should match
SELECT
  (SELECT SUM(total_usd) FROM `project.analytics.events_current` WHERE event_date = '2024-06-01') AS old_total,
  (SELECT SUM(total_usd) FROM `project.analytics.events_next`    WHERE event_date = '2024-06-01') AS new_total;
```

Run your full suite of data quality checks against `events_next` before proceeding.

---

## Step 4: Atomic cutover

Swap the view to point to the new table. This is instantaneous — consumers see the new table on their next query:

```sql
-- Atomic swap: update the view
CREATE OR REPLACE VIEW `project.analytics.events` AS
SELECT * FROM `project.analytics.events_next`;
```

The old table (`events_current`) still exists. Nothing has been deleted yet.

---

## Step 5: Monitor, then clean up

Monitor dashboards and pipeline outputs for 24–48 hours. If something is wrong:

```sql
-- Rollback: just swap the view back
CREATE OR REPLACE VIEW `project.analytics.events` AS
SELECT * FROM `project.analytics.events_current`;
```

This rollback takes one second. Once you're confident the new table is healthy:

```sql
-- Rename to keep naming consistent
-- (optional: archive the old table first)
DROP TABLE `project.analytics.events_current`;
RENAME TABLE `project.analytics.events_next` TO `project.analytics.events_current`;
```

---

## QE Tip

The validation step (Step 3) is where most blue-green swaps fail — not technically, but because teams skip it under time pressure. Write your validation queries before you start the cutover process. They should take less than 5 minutes to run. If you're making a change important enough for a blue-green swap, it's important enough to spend 5 minutes validating.

---

**Key Takeaway:** Use a view as a stable consumer-facing pointer. Build the new table in parallel, validate it, then update the view atomically. Rollback takes one SQL statement.

**→ Next:** [5.3 Snapshots & Time Travel](03-snapshots-time-travel.md)
