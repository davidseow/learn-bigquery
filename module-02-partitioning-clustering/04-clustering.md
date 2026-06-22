# 2.4 Clustering

**TL;DR:** Clustering sorts data within each partition by up to 4 columns. BigQuery uses this to skip blocks of data that don't match your filter — without you doing anything extra.

---

## What you'll learn

- How clustering differs from partitioning
- How to choose good clustering keys
- What block-level pruning means in practice

---

## Clustering vs partitioning

| | Partitioning | Clustering |
|-|-------------|-----------|
| Splits data into | Separate physical segments | Sorted blocks within each segment |
| Max columns | 1 | Up to 4 |
| Cost savings shown upfront? | Yes (in query validator) | No — savings only shown after query runs |
| Good for | High-cardinality date/time, integer buckets | String filters, enum-like columns, join keys |

They complement each other — use both together (covered in the next lesson).

---

## How block-level pruning works

BigQuery stores data in blocks of ~1 MB. When you cluster by `country`, all rows where `country = 'GB'` are physically adjacent within a partition. BigQuery tracks the min/max value in each block.

When your query filters on `country = 'GB'`, BigQuery checks each block's range metadata and skips blocks that can't contain `'GB'`. This is **block-level pruning** — it doesn't require reading any actual row data to skip whole blocks.

---

## Creating a clustered table

```sql
CREATE TABLE `project.analytics.events`
(
  event_id    STRING    NOT NULL,
  user_id     STRING,
  event_name  STRING,
  country     STRING,
  platform    STRING,
  event_date  DATE
)
PARTITION BY event_date
CLUSTER BY country, platform, event_name;
```

Clustering key order matters. Pruning works best when your WHERE clause filters on the **first** clustering column. It degrades as you skip columns.

```sql
-- Best pruning: filters on first clustering key (country)
WHERE event_date = '2024-06-01' AND country = 'GB'

-- Good pruning: filters on first two keys
WHERE event_date = '2024-06-01' AND country = 'GB' AND platform = 'ios'

-- Weaker pruning: skips first key
WHERE event_date = '2024-06-01' AND platform = 'ios'
```

---

## Choosing good clustering keys

Good clustering keys are columns you frequently:
- Filter on in WHERE clauses
- Use in JOIN conditions
- GROUP BY in aggregations

Prefer columns with moderate cardinality — not too few values (boolean is useless), not too many (UUID columns give no benefit because every block has different values).

Good candidates: `country`, `status`, `product_category`, `event_name`, `tenant_id`.
Poor candidates: `user_id` (too many unique values), `is_active` (only 2 values).

---

## Automatic re-clustering

BigQuery automatically re-clusters tables in the background as new data is inserted — at no charge and without any action required. Over time, clustering performance is maintained even as the table grows.

---

## QE Tip

Clustering savings are not shown in the query estimator — they only appear in actual job stats after the query runs. When benchmarking a clustering strategy, run the same query before and after and compare `totalBytesProcessed` in the job details. That's the only reliable measurement.

---

**Key Takeaway:** Cluster on the columns your WHERE clauses and JOINs filter on most. Order the clustering keys by filter frequency — first key gets the most pruning benefit.

**→ Next:** [2.5 Partition + Cluster Together](05-combine-partition-cluster.md)
