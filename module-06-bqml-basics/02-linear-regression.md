# 6.2 Linear Regression in BQML

**TL;DR:** Linear regression predicts a numeric value (price, revenue, duration). This lesson walks through a full end-to-end example: prepare data, train, predict.

---

## What you'll learn

- How to structure training data for BQML
- CREATE MODEL with linear regression options
- Run predictions with ML.PREDICT

---

## The use case

Predict the total order value for new customers based on their acquisition channel, device type, country, and number of sessions in their first week.

---

## Step 1: Prepare training data

BQML expects a flat table with:
- One row per training example
- Feature columns (inputs)
- One label column (what you're predicting)

```sql
CREATE OR REPLACE TABLE `project.ml.customer_ltv_training` AS
SELECT
  -- Features
  acquisition_channel,
  device_type,
  country,
  sessions_week_1,
  pages_viewed_week_1,
  
  -- Label (what we want to predict)
  total_revenue_90d AS label
FROM `project.analytics.customer_cohorts`
WHERE cohort_date BETWEEN '2023-01-01' AND '2024-01-01'  -- 1 year of training data
  AND total_revenue_90d IS NOT NULL;                       -- exclude missing labels
```

---

## Step 2: Train the model

```sql
CREATE OR REPLACE MODEL `project.ml.customer_ltv_model`
OPTIONS (
  model_type             = 'linear_reg',
  input_label_cols       = ['label'],
  data_split_method      = 'AUTO_SPLIT',  -- BQML holds out 20% for validation
  max_iterations         = 50,
  learn_rate_strategy    = 'line_search',
  l1_reg                 = 0.01,
  l2_reg                 = 0.01
)
AS SELECT * FROM `project.ml.customer_ltv_training`;
```

BQML automatically:
- One-hot encodes string columns (acquisition_channel, device_type, country)
- Normalises numeric columns
- Splits data for train/validation

Training for 100k rows takes a few minutes. Check the BigQuery Console — model training shows progress in the job detail.

---

## Step 3: Check training metrics

```sql
SELECT
  iteration,
  loss,
  eval_loss,
  duration_ms,
  learning_rate
FROM ML.TRAINING_INFO(MODEL `project.ml.customer_ltv_model`)
ORDER BY iteration;
```

Look for:
- `loss` (training) and `eval_loss` (validation) both decreasing
- The two converging — a widening gap means overfitting

---

## Step 4: Evaluate on held-out test data

```sql
SELECT
  mean_absolute_error,
  mean_squared_error,
  mean_squared_log_error,
  median_absolute_error,
  r2_score,
  explained_variance
FROM ML.EVALUATE(
  MODEL `project.ml.customer_ltv_model`,
  (
    SELECT *
    FROM `project.analytics.customer_cohorts`
    WHERE cohort_date >= '2024-01-01'   -- unseen test data
      AND total_revenue_90d IS NOT NULL
  )
);
```

Key metric: `r2_score`. Above 0.7 is generally useful; below 0.5 means the features don't explain much of the variance.

---

## Step 5: Generate predictions

```sql
SELECT
  customer_id,
  predicted_label AS predicted_revenue_90d,
  predicted_label_intervals[OFFSET(0)].lower AS lower_bound_95,
  predicted_label_intervals[OFFSET(0)].upper AS upper_bound_95
FROM ML.PREDICT(
  MODEL `project.ml.customer_ltv_model`,
  (
    SELECT
      customer_id,
      acquisition_channel,
      device_type,
      country,
      sessions_week_1,
      pages_viewed_week_1
    FROM `project.analytics.new_customers_this_week`
  )
);
```

Store predictions back in BigQuery for downstream use in dashboards or targeting logic.

---

## QE Tip

Always keep a separate held-out test set — never evaluate on data the model saw during training. BQML's `AUTO_SPLIT` handles train/validation splitting, but you should still hold back the most recent month of data as a true out-of-time test. This is the most realistic measure of how the model will perform in production.

---

**Key Takeaway:** BQML linear regression needs a flat table with feature columns and a `label` column. CREATE MODEL trains it, ML.EVALUATE tests it, ML.PREDICT applies it. The whole workflow is SQL.

**→ Next:** [6.3 Classification Models](03-classification.md)
