# 4.1 Why Data Quality Matters

**TL;DR:** Bad data in BigQuery is silent. There are no foreign key violations, no constraint errors at write time. If you don't check your data, no one will — until a business decision is made on wrong numbers.

---

## What you'll learn

- Why BigQuery's permissiveness is a quality risk
- The real cost of bad data vs the cost of catching it early
- The three dimensions of data quality to monitor

---

## BigQuery is permissive by design

BigQuery is built for scale and flexibility, not enforcement:

- **No unique constraints.** Duplicate rows load silently.
- **No foreign keys.** A `customer_id` that doesn't exist in the customers table will never error.
- **No NOT NULL at load time** (for streaming inserts). NULLs can appear in columns declared NOT NULL if the schema was altered or data was loaded via certain methods.
- **No referential integrity.** Joins on mismatched keys simply produce fewer rows — no error, just missing data.

These are features, not bugs — they allow BigQuery to ingest data from heterogeneous sources at scale. But they shift the quality burden onto you.

---

## The cost of catching issues late

| When caught | Cost |
|-------------|------|
| At load time (pipeline check) | Fix the pipeline, reload the file |
| In staging (QE check before promotion) | Roll back the staging table |
| In prod, before business use | Incident, table repair, comms to stakeholders |
| In prod, after business decisions made | Audit, rework of decisions, trust damage |

Every shift right in the above table multiplies the cost by roughly 10×. Catching a duplicate row problem in the pipeline takes minutes. Catching it after the quarterly board report has been presented is a different conversation entirely.

---

## Three dimensions to monitor

**Freshness:** Is the data up to date? A pipeline that silently stops delivering data looks the same as a table with no issues — until someone queries it and gets yesterday's numbers without knowing.

**Completeness:** Are all expected rows present? A filter that accidentally dropped a country, a partition that loaded zero rows, a join that killed half the records — all of these show up as missing data.

**Correctness:** Are the values right? Nulls where there shouldn't be any, negative values in fields that should be positive, totals that don't match source systems.

---

## The silent failure problem

The most dangerous data quality issue is the one that doesn't error. An ETL job that finishes successfully but writes 40% fewer rows than usual is more dangerous than one that fails — because the failure is invisible.

```sql
-- A healthy pipeline should produce roughly consistent row counts per day.
-- This query flags days where the count was less than 80% of the 7-day average.
SELECT
  event_date,
  row_count,
  avg_7d,
  ROUND(row_count / avg_7d, 2) AS ratio
FROM (
  SELECT
    event_date,
    COUNT(*) AS row_count,
    AVG(COUNT(*)) OVER (
      ORDER BY event_date
      ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING
    ) AS avg_7d
  FROM `project.analytics.events`
  GROUP BY event_date
)
WHERE ratio < 0.8
ORDER BY event_date DESC;
```

---

## QE Tip

Data quality is not a QA team responsibility — it's a pipeline ownership responsibility. Every team that writes data to BigQuery owns the quality of that data. Build checks into the pipeline, not as an afterthought downstream. The consuming team should not be the first to discover your data is broken.

---

**Key Takeaway:** BigQuery won't tell you your data is wrong. Freshness, completeness, and correctness checks need to be built into every pipeline that writes to production tables.

**→ Next:** [4.2 Built-in Validation Techniques](02-built-in-validation.md)
