# 4.4 Data Contracts & Expectations

**TL;DR:** A data contract is a formal agreement between the team that produces a table and the teams that consume it. It specifies what columns exist, what types they are, what quality guarantees apply, and who owns them.

---

## What you'll learn

- What a data contract is and why it reduces incidents
- How to express contracts in BigQuery and dbt
- What to include in a practical contract

---

## Why contracts exist

Without contracts, consumers discover schema changes when their pipeline breaks. The producer changes a column type from `STRING` to `INT64`, and three downstream dbt models silently fail two hours later.

A contract is a coordination mechanism: it makes the producer responsible for communicating changes, and gives consumers a stable interface to build on.

---

## What a data contract covers

```
Table: project.analytics.orders
Owner: data-platform-team@company.com
SLA:
  - Available by 06:00 UTC daily
  - Row completeness: ≥ 99.9% of source orders reflected within 24h
  - Partition freshness: today's partition has rows by 06:00 UTC

Schema (guaranteed stable — any breaking change requires 30-day notice):
  order_id       STRING  NOT NULL  -- globally unique order identifier
  order_date     DATE    NOT NULL  -- date order was placed (partition key)
  customer_id    STRING  NOT NULL  -- FK to customers table
  total_usd      NUMERIC NOT NULL  -- order value, always > 0
  status         STRING  NOT NULL  -- one of: pending, completed, cancelled, refunded

Breaking changes: rename, delete, or type-change of any column
Non-breaking changes: adding new NULLABLE columns (30-day notice not required)
```

---

## Expressing contracts in dbt

dbt has native contract support (dbt v1.5+):

```yaml
# models/sales/schema.yml
models:
  - name: orders
    config:
      contract:
        enforced: true   # dbt will fail if the model output doesn't match the schema
    columns:
      - name: order_id
        data_type: string
        constraints:
          - type: not_null
          - type: unique
      - name: order_date
        data_type: date
        constraints:
          - type: not_null
      - name: total_usd
        data_type: numeric
        constraints:
          - type: not_null
```

With `contract.enforced: true`, dbt validates the output schema matches the declared contract before the run completes. If a transformation accidentally drops a column or changes a type, dbt fails the run.

---

## Communicating changes

A practical change protocol:

| Change type | Protocol |
|-------------|----------|
| Add new NULLABLE column | Announce in team channel, merge, deploy |
| Add new NOT NULL column | Coordinate with all consumers, add with a default, migrate |
| Rename a column | Deprecate old name (keep as alias), add new name, 30-day migration window, remove old |
| Change a column type | Treat as rename — old + new in parallel, then deprecate old |
| Remove a column | Same as rename — deprecate, wait, remove |

---

## Practical contract tracking

Store contracts in your dbt schema YAML files and treat them as code: pull requests, reviews, and approvals required for any breaking change. This gives you a full audit trail of what changed and who approved it.

For cross-team contracts, consider a shared `contracts/` directory in your data platform repo:

```
data-platform/
  contracts/
    orders.md          ← human-readable SLA and column descriptions
    orders_schema.yml  ← dbt-enforced schema contract
    events.md
    events_schema.yml
```

---

## QE Tip

Run a schema drift check at the start of every consumer pipeline:

```sql
-- Fail if the source table is missing an expected column
SELECT
  IF(
    'order_id' IN (SELECT column_name FROM project.analytics.INFORMATION_SCHEMA.COLUMNS WHERE table_name = 'orders'),
    'OK',
    ERROR('Contract violation: order_id column not found in orders table')
  ) AS contract_check;
```

This fails loudly the moment a producer breaks the contract — before your pipeline writes garbage data downstream.

---

**Key Takeaway:** A data contract is a written schema + SLA agreement stored in version control. Use dbt's enforced contracts to catch violations at build time, and a schema drift check at the start of consumer pipelines to catch them at runtime.

**→ Next:** [4.5 Monitoring Freshness & Completeness](05-monitoring-freshness.md)
