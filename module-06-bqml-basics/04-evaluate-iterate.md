# 6.4 Evaluate & Iterate

**TL;DR:** A trained model is not done — it's version 1 of an ongoing process. This lesson covers how to inspect model internals, compare versions, and set up a baseline to detect when the model degrades.

---

## What you'll learn

- How to inspect feature importance
- How to compare two model versions objectively
- Model monitoring: detecting drift before it causes problems

---

## Feature importance

Understanding which features drive the model's predictions helps you validate it makes sense, and identifies where to focus feature engineering:

```sql
-- For boosted tree models
SELECT
  feature,
  importance_weight,
  importance_gain,
  importance_cover
FROM ML.FEATURE_IMPORTANCE(MODEL `project.ml.churn_boosted_tree`)
ORDER BY importance_gain DESC;
```

`importance_gain`: how much the feature reduces loss when used for splitting (most useful metric).

If `days_since_last_login` has 10× the importance of all other features combined, consider whether that's genuinely signal or whether it's leaking information from after the prediction horizon.

---

## Comparing two model versions

When iterating, use a held-out test set to compare objectively:

```sql
WITH model_v1 AS (
  SELECT 'v1_logistic' AS model_name, roc_auc, f1_score
  FROM ML.EVALUATE(
    MODEL `project.ml.churn_logistic`,
    TABLE `project.ml.churn_test_set`
  )
),
model_v2 AS (
  SELECT 'v2_boosted_tree' AS model_name, roc_auc, f1_score
  FROM ML.EVALUATE(
    MODEL `project.ml.churn_boosted_tree`,
    TABLE `project.ml.churn_test_set`
  )
)
SELECT * FROM model_v1
UNION ALL
SELECT * FROM model_v2
ORDER BY roc_auc DESC;
```

A small improvement (e.g., ROC AUC from 0.78 to 0.80) may not justify switching — consider training cost, complexity, and explainability.

---

## Hyperparameter tuning

BQML supports basic hyperparameter search:

```sql
CREATE OR REPLACE MODEL `project.ml.churn_tuned`
OPTIONS (
  model_type              = 'boosted_tree_classifier',
  input_label_cols        = ['label'],
  num_trials              = 10,     -- try 10 combinations
  max_parallel_trials     = 2,
  hparam_tuning_algorithm = 'VIZIER_DEFAULT',
  -- Define search space:
  num_trees               = hparam_range(50, 300),
  max_tree_depth          = hparam_range(3, 8),
  learn_rate              = hparam_candidates([0.01, 0.05, 0.1, 0.3])
)
AS SELECT * FROM `project.ml.churn_training`;
```

Results:

```sql
SELECT * FROM ML.TRIAL_INFO(MODEL `project.ml.churn_tuned`)
ORDER BY eval_loss ASC;
```

---

## Detecting model degradation (model monitoring)

Models degrade over time as user behaviour changes. Detect it by scoring your model on current data and comparing the prediction distribution to historical baselines:

```sql
-- Compare average churn probability this week vs 4 weeks ago
SELECT
  week_start,
  AVG(churn_probability) AS avg_predicted_churn_rate
FROM (
  SELECT
    DATE_TRUNC(prediction_date, WEEK) AS week_start,
    predicted_label_probs[OFFSET(0)].prob AS churn_probability
  FROM `project.ml.churn_predictions_history`
)
GROUP BY 1
ORDER BY 1 DESC
LIMIT 8;
```

A sudden jump in predicted churn rate may indicate real user behaviour change (act on it) or feature distribution shift (retrain the model).

---

## When to retrain

Retrain when:
- Actual vs predicted accuracy drops by more than ~5% on recent data
- Feature distributions change significantly (new product, seasonal shift)
- The label definition changes (business logic update)
- A new relevant feature becomes available

For BQML, retraining is as simple as re-running `CREATE OR REPLACE MODEL` — it overwrites the existing model object.

---

## QE Tip

Log predictions to a BigQuery table with a `prediction_date` column. This gives you the historical prediction distribution needed to detect drift. Without a prediction log, you have no way to know if the model is performing well on current data vs when it was trained.

```sql
INSERT INTO `project.ml.churn_predictions_history`
SELECT
  user_id,
  CURRENT_DATE() AS prediction_date,
  predicted_label,
  predicted_label_probs[OFFSET(0)].prob AS churn_probability
FROM ML.PREDICT(MODEL `project.ml.churn_boosted_tree`, TABLE `project.analytics.active_users`);
```

---

**Key Takeaway:** Feature importance reveals what your model learned. Compare model versions on the same test set. Log predictions historically so you can detect when the model starts to drift.

**→ Next:** [6.5 Export to Vertex AI](05-export-to-vertex.md)
