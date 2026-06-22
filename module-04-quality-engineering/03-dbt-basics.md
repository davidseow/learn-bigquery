# 4.3 dbt for BigQuery

**TL;DR:** dbt (data build tool) turns your SQL SELECT statements into a tested, documented, version-controlled data pipeline. It's the closest thing to software engineering practices applied to analytics.

---

## What you'll learn

- What dbt does and why it's worth learning alongside BigQuery
- Models, tests, and documentation in a 5-minute overview
- dbt-specific BigQuery optimisations

---

## What dbt is

dbt sits between your raw data and your analytics tables. You write `SELECT` statements — dbt handles the `CREATE TABLE`, `INSERT`, and orchestration.

```
raw sources  →  dbt models (your SQL)  →  BigQuery tables/views
                     ↓
              dbt tests (assertions)
                     ↓
              dbt docs (auto-generated)
```

dbt does not replace BigQuery — it's a transformation layer that runs on top of it.

---

## Models: SELECT statements that become tables

Every dbt model is a `.sql` file containing a `SELECT`:

```sql
-- models/sales/orders_daily.sql
SELECT
  DATE(order_timestamp) AS order_date,
  product_category,
  country,
  SUM(total_usd)        AS revenue_usd,
  COUNT(order_id)       AS order_count
FROM {{ ref('orders_raw') }}       -- reference another model
WHERE order_status != 'cancelled'
GROUP BY 1, 2, 3
```

`{{ ref('orders_raw') }}` tells dbt this model depends on `orders_raw`. dbt builds a dependency graph and executes models in the right order.

Run with:
```bash
dbt run --select orders_daily
```

dbt executes `CREATE OR REPLACE TABLE project.analytics.orders_daily AS SELECT ...` in BigQuery.

---

## Config: partitioning and clustering in dbt

Add partitioning/clustering to your dbt model config:

```sql
-- models/analytics/events.sql
{{
  config(
    materialized   = 'incremental',
    partition_by   = {'field': 'event_date', 'data_type': 'date'},
    cluster_by     = ['country', 'platform', 'event_name'],
    incremental_strategy = 'insert_overwrite'
  )
}}

SELECT ...
FROM {{ source('raw', 'events_landing') }}
{% if is_incremental() %}
  WHERE event_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 3 DAY)
{% endif %}
```

`incremental` means dbt only processes new/changed rows on subsequent runs — not the full table every time.

---

## Tests: data quality built in

dbt ships with built-in tests that you define in YAML:

```yaml
# models/sales/schema.yml
models:
  - name: orders_daily
    columns:
      - name: order_date
        tests:
          - not_null
          - dbt_utils.not_constant  # value should vary
      - name: revenue_usd
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              max_value: 1000000
```

Run tests with:
```bash
dbt test --select orders_daily
```

Each test generates a SQL query that returns failing rows — empty result = pass.

---

## Custom tests

For checks not covered by built-in tests:

```sql
-- tests/assert_no_duplicate_orders.sql
SELECT order_id, COUNT(*) AS cnt
FROM {{ ref('orders_daily') }}
GROUP BY order_id
HAVING cnt > 1
```

dbt runs this and fails the test if any rows are returned.

---

## Documentation

Add descriptions to your YAML and dbt generates a browsable data catalogue:

```yaml
models:
  - name: orders_daily
    description: "Daily revenue aggregated by product category and country. Excludes cancelled orders."
    columns:
      - name: revenue_usd
        description: "Total order value in USD, net of cancellations."
```

```bash
dbt docs generate && dbt docs serve
```

---

## QE Tip

The three dbt tests every model should have as a baseline: `not_null` on every primary key and partition column, `unique` on primary keys, and a row count test (`dbt_utils.expression_is_true`) that checks today's count is within 20% of yesterday's. These three tests catch the majority of pipeline issues automatically.

---

**Key Takeaway:** dbt gives you version-controlled, tested, documented SQL pipelines. Models are just SELECT statements; tests are SQL assertions; docs are generated automatically. Use the BigQuery `incremental` materialization to process only new partitions.

**→ Next:** [4.4 Data Contracts & Expectations](04-data-contracts.md)
