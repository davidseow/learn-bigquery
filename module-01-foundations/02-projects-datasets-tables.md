# 1.2 Projects, Datasets, and Tables

**TL;DR:** BigQuery organises everything in three levels: Project > Dataset > Table. Understanding this hierarchy controls your costs, permissions, and data locality.

---

## What you'll learn

- How the three-level resource hierarchy works
- What each level controls (billing, IAM, region)
- How to reference tables in SQL

---

## The hierarchy

```
my-company-prod           ← Project (billing + IAM boundary)
  └── analytics           ← Dataset (region + permissions)
        ├── orders         ← Table (schema + data)
        ├── customers      ← Table
        └── daily_revenue  ← View (saved SQL)
```

---

## Projects

A project is the billing and IAM root. All costs roll up to the project. You control who has access at the project level using Google Cloud IAM roles.

Typical split:
- `company-data-dev` — sandbox for engineers
- `company-data-staging` — pre-production validation
- `company-data-prod` — live data, restricted access

---

## Datasets

A dataset is a container for tables. It has two critical properties:

**Region:** Where the data physically lives — `EU`, `US`, or a single region like `europe-west2`. Queries must run in the same region as the data. You cannot join a table in `EU` with one in `US` directly.

**Permissions:** You can grant access at the dataset level. A data analyst role might have `roles/bigquery.dataViewer` on `analytics` but not on `raw_events`.

---

## Tables, Views, and External Tables

| Type | What it is |
|------|-----------|
| Table | Physical storage — rows and typed columns |
| View | Saved SQL — no physical storage, always fresh |
| Materialized View | Pre-computed results — stored and auto-refreshed |
| External Table | Query data in GCS, Sheets, or Bigtable without loading it |

---

## Referencing tables in SQL

The fully qualified form is: `` `project.dataset.table` ``

```sql
-- Fully qualified — works from any project
SELECT * FROM `my-company-prod.analytics.orders` LIMIT 10;

-- Within the same project, you can omit the project name
SELECT * FROM analytics.orders LIMIT 10;
```

Backtick-quote any name with hyphens (`my-company-prod`) — hyphens are not valid in unquoted identifiers.

---

## QE Tip

Use separate datasets — not just naming conventions — to separate environments. `raw_prod` and `raw_dev` as distinct datasets lets you apply IAM policies that prevent a dev pipeline from ever touching production data. A naming convention like `orders_dev` in the same dataset gives you zero protection.

---

**Key Takeaway:** Project = billing + IAM, Dataset = region + permissions, Table = data. Control your environments by controlling your datasets, not just your table names.

**→ Next:** [1.3 Your First Queries](03-first-queries.md)
