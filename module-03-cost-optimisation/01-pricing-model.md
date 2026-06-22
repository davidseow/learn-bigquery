# 3.1 How BigQuery Pricing Works

**TL;DR:** BigQuery charges for data scanned (on-demand) or compute capacity reserved (flat-rate). Understanding which model you're on changes every cost decision you make.

---

## What you'll learn

- The two pricing models and when each applies
- What "bytes billed" actually means
- Storage pricing basics

---

## On-demand pricing

You pay for the bytes scanned by each query.

- **Rate:** ~$5 per TB scanned (varies by region; check current Google Cloud pricing)
- **Free tier:** First 1 TB of queries per month per project is free
- **Minimum:** Each query is billed a minimum of 10 MB, even if it scans less

This model is pay-as-you-go. Great for experimentation and teams with unpredictable query patterns. The downside: a poorly written query on a large table can create an unexpected bill in minutes.

---

## Flat-rate (capacity) pricing

You buy slots — units of compute — and all queries share that capacity.

- A slot is a virtual CPU that BigQuery uses to execute queries
- You pay for slots per second, not for bytes scanned
- Queries still run, they just queue if you run out of slots

Two sub-options:
- **Flex slots:** Rent slots by the minute. Good for short-duration bursts.
- **Committed reservations:** 1-year or 3-year commit for maximum discount (~50–70% off flex).

Once you're on flat-rate, the cost of scanning 1 GB vs 1 TB is the same — you've already paid for the compute. This changes the incentive: now you care about query latency (slot efficiency), not bytes scanned.

---

## What counts as "bytes billed"

BigQuery charges for the columns your query reads, not the full table.

```sql
-- This query on a 1 TB table with 20 columns...
SELECT user_id, email FROM `project.crm.customers`;

-- ...only bills for the bytes in user_id + email columns (maybe 50 GB total)
-- Not 1 TB
```

This is why columnar storage directly reduces costs on on-demand pricing.

**Minimum billing:** Even `SELECT 1` is billed as 10 MB. Metadata-only queries (like `SELECT COUNT(*)` on some system tables) are free, but most queries are not.

---

## Storage pricing

| Storage type | Cost |
|-------------|------|
| Active storage | ~$0.02/GB/month |
| Long-term storage (no edits for 90+ days) | ~$0.01/GB/month |
| Streaming inserts buffer | Charged at insert rate |

Storage automatically transitions to long-term pricing after a table partition hasn't been modified for 90 days. Partition expiry (covered in Module 2) keeps old data from accumulating.

---

## Free operations

These don't cost query bytes:
- Loading data (batch loads via `bq load` or the Console)
- Exporting data to GCS
- Metadata queries on `INFORMATION_SCHEMA`
- Cached query results (same query within 24 hours, data unchanged)

---

## QE Tip

Set up billing budget alerts in Google Cloud Billing — not in BigQuery itself. Alert at 50%, 90%, and 100% of your monthly budget. Without alerts, the first signal you get from an out-of-control query pattern is the invoice.

---

**Key Takeaway:** On-demand = pay per byte scanned; flat-rate = pay per slot. On on-demand, reducing bytes scanned is the primary cost lever. On flat-rate, maximising slot efficiency (query latency) matters most.

**→ Next:** [3.2 Dry Runs & Estimation](02-dry-runs-estimation.md)
