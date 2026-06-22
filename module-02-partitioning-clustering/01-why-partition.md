# 2.1 Why Partition?

**TL;DR:** Partitioning splits a table into segments so BigQuery only scans the slices your query actually needs. It's the single biggest lever for reducing query cost and latency.

---

## What you'll learn

- What query pruning is and why it matters
- How partitioning lets BigQuery skip irrelevant data
- What happens without partitioning on a large table

---

## The problem with large tables

Imagine a 3-year `events` table with 500 billion rows. A query asking for "last 7 days of events" still has to scan all 500 billion rows unless BigQuery knows how to skip the rest.

Without partitioning: **500 GB scanned. Every time. For every 7-day query.**

---

## How partitioning works

Partitioning divides a table into physical segments — one per partition value. A table partitioned by `event_date` has one segment per day.

When you run:

```sql
SELECT user_id, event_name
FROM `project.analytics.events`
WHERE event_date BETWEEN '2024-06-01' AND '2024-06-07';
```

BigQuery reads the partition metadata first, then only opens the 7 day-partitions that match. Everything else is **pruned** — never read, never billed.

```
events (partitioned by event_date)
├── 2024-06-01  ← scanned
├── 2024-06-02  ← scanned
├── 2024-06-03  ← scanned
...
├── 2022-01-01  ← SKIPPED (pruned)
└── 2021-12-31  ← SKIPPED (pruned)
```

---

## The savings add up

A table growing at 5 GB/day accumulates 1.8 TB/year. A query for "today's data" on a partitioned table scans 5 GB. On an unpartitioned table: 1.8 TB. That's a 360× cost difference.

---

## Pruning only works with direct filters

BigQuery pruning is intelligent but not magic:

```sql
-- Pruning works: direct comparison on the partition column
WHERE event_date = '2024-06-01'

-- Pruning works: BETWEEN on the partition column
WHERE event_date BETWEEN '2024-06-01' AND '2024-06-07'

-- Pruning does NOT work: wrapping in a function breaks pruning
WHERE DATE_TRUNC(event_date, MONTH) = '2024-06-01'

-- Pruning does NOT work: dynamic values from subqueries (in most cases)
WHERE event_date = (SELECT MAX(event_date) FROM another_table)
```

Use `DATE_TRUNC` in SELECT, not in WHERE, when filtering on a partition column.

---

## QE Tip

After partitioning a table, always verify pruning actually works. In the BigQuery console, run a query with a partition filter and check "bytes processed" — it should be proportional to the partitions selected, not the full table size. If it's scanning the whole table, your WHERE clause isn't hitting the partition column correctly.

---

**Key Takeaway:** Partitioning enables query pruning — BigQuery skips irrelevant data segments entirely. A direct filter on the partition column is what triggers it.

**→ Next:** [2.2 Time-Based Partitioning](02-time-based-partitioning.md)
