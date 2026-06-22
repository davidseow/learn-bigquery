# 7.1 Feature Engineering in SQL

**TL;DR:** The quality of your ML model depends on the quality of its features. BigQuery SQL — especially window functions and aggregations — is a powerful feature engineering tool that scales to billions of rows.

---

## What you'll learn

- Building time-based and behavioural features with SQL
- Window functions for rolling and lag features
- Entity-level aggregations as features

---

## What feature engineering means

Raw data rarely contains the right signal for ML. Feature engineering transforms raw events into the meaningful numeric or categorical inputs a model can learn from.

Example: raw data is a `page_views` event log. Useful features:
- Sessions in the last 7 days (recency + frequency)
- Average time between sessions (engagement pattern)
- Ratio of product pages to checkout pages (intent signal)
- Days since first visit (tenure)

---

## Pattern 1: Rolling window aggregations

```sql
-- Features per user: activity in the last 30 days as of a given date
SELECT
  user_id,
  snapshot_date,

  -- Recency
  COUNT(DISTINCT session_id)           AS sessions_30d,
  COUNT(*)                             AS total_events_30d,
  DATE_DIFF(snapshot_date, MAX(DATE(event_timestamp)), DAY) AS days_since_last_event,

  -- Frequency per channel
  COUNTIF(source = 'organic')          AS organic_sessions_30d,
  COUNTIF(source = 'paid')             AS paid_sessions_30d,

  -- Engagement
  SAFE_DIVIDE(
    COUNTIF(event_name = 'checkout_started'),
    COUNTIF(event_name = 'product_viewed')
  )                                    AS checkout_intent_rate

FROM `project.analytics.events`
WHERE DATE(event_timestamp) BETWEEN
    DATE_SUB(snapshot_date, INTERVAL 30 DAY) AND snapshot_date
GROUP BY user_id, snapshot_date;
```

---

## Pattern 2: Lag features with window functions

Lag features capture how a metric changed over time — often more predictive than point-in-time values:

```sql
SELECT
  user_id,
  snapshot_date,
  sessions_this_week,

  -- How much did activity change vs last week?
  LAG(sessions_this_week, 1) OVER (
    PARTITION BY user_id ORDER BY snapshot_date
  ) AS sessions_prev_week,

  -- Percentage change (handle division by zero)
  SAFE_DIVIDE(
    sessions_this_week - LAG(sessions_this_week, 1) OVER (
      PARTITION BY user_id ORDER BY snapshot_date
    ),
    NULLIF(LAG(sessions_this_week, 1) OVER (
      PARTITION BY user_id ORDER BY snapshot_date
    ), 0)
  ) AS sessions_wow_change

FROM `project.ml.user_weekly_activity`
ORDER BY user_id, snapshot_date;
```

---

## Pattern 3: Cross-entity features (join-based)

Sometimes features come from related entities:

```sql
SELECT
  o.customer_id,
  o.order_id,

  -- Customer-level features
  c.account_tier,
  c.days_as_customer,

  -- Product-level features
  p.category,
  p.avg_rating,

  -- Historical order features for this customer
  ch.total_past_orders,
  ch.avg_order_value,
  ch.return_rate

FROM `project.sales.orders` o
JOIN `project.crm.customers` c USING (customer_id)
JOIN `project.catalogue.products` p USING (product_id)
JOIN `project.ml.customer_order_history` ch USING (customer_id);
```

---

## Pattern 4: Date and time features

Temporal patterns are powerful predictors in many domains:

```sql
SELECT
  event_timestamp,
  EXTRACT(HOUR FROM event_timestamp)        AS hour_of_day,
  EXTRACT(DAYOFWEEK FROM event_timestamp)   AS day_of_week,  -- 1=Sunday
  EXTRACT(MONTH FROM event_timestamp)       AS month,
  IF(EXTRACT(DAYOFWEEK FROM event_timestamp) IN (1, 7), 1, 0) AS is_weekend,
  DATE_DIFF(CURRENT_DATE(), DATE(event_timestamp), DAY) AS days_ago
FROM `project.analytics.events`;
```

---

## Storing a feature table for reuse

Feature computation is expensive — don't recompute in every training run. Store features in a feature table:

```sql
CREATE OR REPLACE TABLE `project.ml.user_features_weekly`
PARTITION BY snapshot_date
CLUSTER BY user_id
AS
SELECT ... -- your feature computation query
```

Training queries then join against this table rather than recomputing from raw events.

---

## QE Tip

Feature engineering bugs are silent: a misaligned window, an accidental `LEFT JOIN` that inflates row counts, or a `SAFE_DIVIDE` that returns NULL instead of 0 can all degrade model performance without producing an error. Add row count checks and distribution assertions (min/max/NULL rate per feature) to your feature table build as standard post-build validation.

---

**Key Takeaway:** Most ML feature engineering is SQL: rolling windows, lags, cross-entity joins, and temporal decompositions. Store computed features in a partitioned table rather than recomputing on every training run.

**→ Next:** [7.2 Embeddings with BQML](02-embeddings-bqml.md)
