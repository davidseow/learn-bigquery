# 5.4 Safe Migration Patterns

**TL;DR:** The safest migrations run in parallel with production, validate before cutting over, and are fully reversible. This lesson is the step-by-step playbook.

---

## What you'll learn

- The shadow write pattern for validating new pipelines
- Parallel validation before cutover
- A gradual rollout approach using traffic splitting

---

## Pattern 1: Shadow writes

Run a new pipeline in parallel with the old one, writing to a shadow table. Compare outputs before cutting over.

```
Old pipeline → production table      (consumers use this)
New pipeline → shadow table          (no consumers — just for comparison)
```

Once the shadow table passes validation, the cutover is just updating the consumer pointer (the view from Lesson 5.2).

**Implementation:**

```python
# In your pipeline code, write to both destinations
def run_pipeline(event_date: str, shadow_mode: bool = False):
    result = transform_events(event_date)
    
    # Always write to production
    write_to_bigquery(result, table='project.analytics.events')
    
    if shadow_mode:
        # Also write to shadow for comparison
        write_to_bigquery(result, table='project.analytics.events_shadow')
```

Run with `shadow_mode=True` for a week, then run the validation queries below.

---

## Pattern 2: Parallel validation

After running shadow writes, compare the two tables systematically:

```sql
-- Compare row counts per partition
SELECT
  'production' AS source, event_date, COUNT(*) AS row_count
FROM `project.analytics.events`
WHERE event_date BETWEEN '2024-06-01' AND '2024-06-07'
GROUP BY 1, 2

UNION ALL

SELECT
  'shadow' AS source, event_date, COUNT(*) AS row_count
FROM `project.analytics.events_shadow`
WHERE event_date BETWEEN '2024-06-01' AND '2024-06-07'
GROUP BY 1, 2

ORDER BY event_date, source;

-- Find rows in production missing from shadow
SELECT p.*
FROM `project.analytics.events` p
LEFT JOIN `project.analytics.events_shadow` s USING (event_id, event_date)
WHERE s.event_id IS NULL
  AND p.event_date = '2024-06-07';

-- Find rows in shadow missing from production
SELECT s.*
FROM `project.analytics.events_shadow` s
LEFT JOIN `project.analytics.events` p USING (event_id, event_date)
WHERE p.event_id IS NULL
  AND s.event_date = '2024-06-07';
```

Zero discrepancies = ready to cut over.

---

## Pattern 3: The staged cutover

For high-risk migrations, don't cut over all consumers at once. Route a small percentage first:

```sql
-- View that mixes old and new table by hashing a stable ID
CREATE OR REPLACE VIEW `project.analytics.events` AS
SELECT *
FROM `project.analytics.events_next`
WHERE MOD(ABS(FARM_FINGERPRINT(event_id)), 10) < 2  -- 20% from new table

UNION ALL

SELECT *
FROM `project.analytics.events_current`
WHERE MOD(ABS(FARM_FINGERPRINT(event_id)), 10) >= 2; -- 80% from old table
```

Monitor downstream metrics. If they look healthy, increase the percentage. Full cutover when you're confident.

---

## The migration runbook template

Write this before starting any migration:

```
Migration: Add device_type column and change partitioning
Date: 2024-06-20
Owner: @your-name
Rollback time: < 5 minutes (view swap)

Pre-migration checklist:
  [ ] Create snapshot: events_snapshot_20240620
  [ ] Record current row count: SELECT COUNT(*) ...
  [ ] Notify consuming teams via #data-engineering

Steps:
  1. CREATE TABLE events_next AS ... (estimated 2h)
  2. Run shadow validation queries
  3. Review: zero discrepancies required to proceed
  4. Create OR REPLACE VIEW events → events_next
  5. Monitor dashboards for 30 min

Rollback:
  CREATE OR REPLACE VIEW events → events_current

Post-migration:
  [ ] Validate dashboards look correct
  [ ] Update documentation
  [ ] Drop events_current (after 48h monitoring)
```

---

## QE Tip

The most common reason migrations fail is skipping the runbook. Engineers who have done it before "know what to do" — and then discover mid-migration that they forgot to take a snapshot, or that a consumer they didn't know about breaks. Write the runbook. Make a teammate review it. The 30-minute investment saves hours of incident response.

---

**Key Takeaway:** Shadow writes give you a parallel validation safety net. Validate completely before cutting over, and always write the rollback step before you start — not after something breaks.

**→ Next:** [5.5 Change Testing Checklist](05-change-testing-checklist.md)
