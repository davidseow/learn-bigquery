# 1.1 What is BigQuery?

**TL;DR:** BigQuery is Google's serverless analytics warehouse. You write SQL, Google handles the infrastructure. You pay for data scanned, not for servers sitting idle.

---

## What you'll learn

- Why BigQuery is different from traditional databases
- How columnar storage makes analytics fast
- When BigQuery is the right tool (and when it isn't)

---

## The core idea

Traditional databases (Postgres, MySQL) store data row by row. They're great for transactional workloads — create an order, update a user profile, look up a single record by ID.

BigQuery stores data **column by column**. When you query `SELECT revenue, country FROM orders`, BigQuery only reads those two columns across millions of rows — skipping everything else. This makes analytics queries dramatically faster and cheaper.

There's no server to provision. No indexes to build. No vacuuming. You write SQL, BigQuery figures out how many workers to throw at it, and you get results.

---

## Compute is separate from storage

Your data lives in Google's distributed storage layer (Colossus). Compute — the workers that process your queries — is completely separate and spins up on demand.

This means:
- Storage is cheap and permanent
- Compute scales automatically to the size of your query
- You're not paying for idle capacity overnight

---

## Two pricing models

**On-demand:** Pay per byte scanned. First 1 TB/month is free, then ~$5/TB. Good for ad-hoc queries and teams getting started.

**Flat-rate (reservations):** Buy a fixed number of slots (units of compute). Predictable cost regardless of how much data you scan. Good for high-volume production workloads.

---

## When to use BigQuery

Use it for:
- Analytics and reporting on large datasets (millions to billions of rows)
- ML training data preparation
- Centralising logs, events, and business data for cross-system queries

Don't use it as a replacement for:
- **OLTP databases** — high-frequency single-row reads/writes (use Postgres, Spanner, Firestore)
- **Low-latency APIs** — BQ query latency starts at ~0.5s even for tiny queries
- **Session/cache storage** — use Redis or Memorystore

---

## QE Tip

BigQuery does not enforce uniqueness or foreign keys. There's nothing stopping duplicate rows from landing in your tables. From day one, build deduplication checks into your pipelines — don't assume the data is clean just because it arrived.

---

**Key Takeaway:** BigQuery is a columnar, serverless analytics warehouse. It excels at scanning large datasets cheaply and quickly — but it's an OLAP tool, not an OLTP replacement.

**→ Next:** [1.2 Projects, Datasets, Tables](02-projects-datasets-tables.md)
