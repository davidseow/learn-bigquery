# 3.3 Expensive Query Patterns to Avoid

**TL;DR:** A handful of common SQL patterns silently scan far more data than needed. Knowing them by name lets you catch them in code review before they hit production.

---

## What you'll learn

- The five most expensive anti-patterns in BigQuery SQL
- How to rewrite each one to scan less
- How to spot them in code review

---

## Anti-pattern 1: SELECT *

```sql
-- Bad: reads all columns, even ones you don't need
SELECT * FROM `project.analytics.events` WHERE event_date = CURRENT_DATE();

-- Good: read only what you need
SELECT event_id, user_id, event_name
FROM `project.analytics.events`
WHERE event_date = CURRENT_DATE();
```

On a wide table (50+ columns), `SELECT *` can scan 10–20× more data than selecting the 3 columns you actually use. This is BigQuery's columnar advantage — exploit it.

Exception: Use `SELECT *` freely in CTEs and subqueries within the same query. The cost is only assessed once at the outermost scan.

---

## Anti-pattern 2: Missing partition filter

```sql
-- Bad: scans the entire table (3 years of data)
SELECT COUNT(*) FROM `project.analytics.events`;

-- Good: scope to the period you need
SELECT COUNT(*) FROM `project.analytics.events`
WHERE event_date = CURRENT_DATE();
```

Set `require_partition_filter = TRUE` on large tables to catch this at runtime.

---

## Anti-pattern 3: LIMIT doesn't save you

```sql
-- Bad assumption: "I only need 100 rows, so this is cheap"
SELECT * FROM `project.analytics.events` LIMIT 100;
```

LIMIT cuts the result set, not the scan. BigQuery still reads every matching row from every relevant partition. It's only cheaper if the LIMIT triggers early termination — which it may or may not do depending on the execution plan.

Use WHERE to reduce the scan, then LIMIT to cap the output.

---

## Anti-pattern 4: Self-joins and cross joins

```sql
-- Bad: a cross join between two large tables = O(n²) data read
SELECT a.user_id, b.user_id
FROM `project.analytics.users` a
CROSS JOIN `project.analytics.users` b;

-- Bad: unnecessary self-join (use window functions instead)
SELECT a.user_id, a.event_date, b.prev_event_date
FROM events a
JOIN events b ON a.user_id = b.user_id AND b.event_date < a.event_date;

-- Good: window function — reads the table once
SELECT
  user_id,
  event_date,
  LAG(event_date) OVER (PARTITION BY user_id ORDER BY event_date) AS prev_event_date
FROM events;
```

---

## Anti-pattern 5: Functions on partition/cluster columns in WHERE

```sql
-- Bad: CAST breaks partition pruning
WHERE CAST(event_timestamp AS DATE) = '2024-06-01'

-- Good: use the partition column directly, or use DATE()
WHERE event_date = '2024-06-01'
-- or, if partitioned by timestamp:
WHERE DATE(event_timestamp) = '2024-06-01'
```

BigQuery can prune on `DATE(col)` when the partition is defined as `PARTITION BY DATE(col)`. But wrapping in other functions (CAST, FORMAT_DATE, etc.) usually disables pruning.

---

## Anti-pattern 6: Repeated subqueries

```sql
-- Bad: scans events table twice
SELECT
  user_id,
  (SELECT COUNT(*) FROM events WHERE user_id = u.user_id AND status = 'completed') AS completed,
  (SELECT COUNT(*) FROM events WHERE user_id = u.user_id AND status = 'pending')   AS pending
FROM users u;

-- Good: one scan with conditional aggregation
SELECT
  u.user_id,
  COUNTIF(e.status = 'completed') AS completed,
  COUNTIF(e.status = 'pending')   AS pending
FROM users u
LEFT JOIN events e USING (user_id)
GROUP BY u.user_id;
```

---

## QE Tip

In code review, search for `SELECT *` from production tables (not CTEs), missing WHERE clauses on tables you know are partitioned, and correlated subqueries in SELECT lists. These three patterns catch the majority of cost regressions before they ship.

---

**Key Takeaway:** The five most costly patterns: SELECT *, missing partition filters, trusting LIMIT, cross joins, and functions on filter columns. Each one is fixable with a straightforward rewrite.

**→ Next:** [3.4 Materialized Views](04-materialized-views.md)
