# 6.3 Classification Models in BQML

**TL;DR:** Classification predicts which category something belongs to — churn vs retained, fraud vs legitimate, high-value vs low-value. BQML offers logistic regression, boosted trees, and random forest via SQL.

---

## What you'll learn

- Logistic regression for binary classification
- Boosted tree classifier for higher accuracy
- How to interpret classification-specific metrics

---

## The use case

Predict whether a user will churn in the next 30 days (binary classification: 1 = churns, 0 = stays).

---

## Step 1: Prepare training data

For binary classification, the label column must be `BOOL`, `INT64` (0/1), or `STRING` ('true'/'false'):

```sql
CREATE OR REPLACE TABLE `project.ml.churn_training` AS
SELECT
  -- Features: behaviour in last 30 days
  sessions_last_30d,
  avg_session_duration_sec,
  days_since_last_login,
  support_tickets_last_30d,
  feature_adoption_score,
  subscription_plan,
  tenure_days,
  
  -- Label: did they churn in the following 30 days?
  churned AS label
FROM `project.analytics.user_activity_snapshots`
WHERE snapshot_date BETWEEN '2023-01-01' AND '2024-03-01'
  AND churned IS NOT NULL;
```

---

## Option A: Logistic regression (fast, interpretable)

```sql
CREATE OR REPLACE MODEL `project.ml.churn_logistic`
OPTIONS (
  model_type       = 'logistic_reg',
  input_label_cols = ['label'],
  auto_class_weights = TRUE  -- handles imbalanced classes (more non-churners than churners)
)
AS SELECT * FROM `project.ml.churn_training`;
```

Logistic regression is fast to train and easy to explain to stakeholders. Use it as your baseline.

---

## Option B: Boosted tree classifier (higher accuracy)

```sql
CREATE OR REPLACE MODEL `project.ml.churn_boosted_tree`
OPTIONS (
  model_type              = 'boosted_tree_classifier',
  input_label_cols        = ['label'],
  num_parallel_tree       = 1,
  max_tree_depth          = 6,
  num_trees               = 100,
  learn_rate              = 0.1,
  auto_class_weights      = TRUE
)
AS SELECT * FROM `project.ml.churn_training`;
```

Boosted trees are slower to train but typically more accurate on tabular data. Good when accuracy matters more than interpretability.

---

## Evaluating classification models

```sql
-- Overall metrics
SELECT
  precision,
  recall,
  accuracy,
  f1_score,
  log_loss,
  roc_auc
FROM ML.EVALUATE(
  MODEL `project.ml.churn_boosted_tree`,
  TABLE `project.ml.churn_test_set`   -- held-out test set
);
```

| Metric | What it tells you |
|--------|------------------|
| Precision | Of predicted churners, how many actually churned? |
| Recall | Of actual churners, how many did we catch? |
| F1 | Harmonic mean of precision and recall |
| ROC AUC | Overall discrimination ability (0.5 = random, 1.0 = perfect) |

For churn, **recall** usually matters more than precision — missing a churner (false negative) is more costly than a false positive.

---

## Confusion matrix

```sql
SELECT * FROM ML.CONFUSION_MATRIX(
  MODEL `project.ml.churn_boosted_tree`,
  TABLE `project.ml.churn_test_set`
);
```

Returns a matrix of predicted vs actual classes. Look for false negatives (missed churners) specifically.

---

## Generating predictions with probability scores

```sql
SELECT
  user_id,
  predicted_label,           -- '1' or '0'
  predicted_label_probs[OFFSET(0)].prob AS churn_probability
FROM ML.PREDICT(
  MODEL `project.ml.churn_boosted_tree`,
  (
    SELECT * FROM `project.analytics.active_users_current_week`
  )
)
ORDER BY churn_probability DESC;
```

Use the probability score (not just the binary prediction) for prioritisation — target the 10% highest-risk users first.

---

## QE Tip

Test your model against the **most recent** data you have — not a random sample. If you train on 2023 data and the test set is also random-sampled from 2023, you might have an optimistic estimate. A more realistic test is: train on Jan–Oct 2023, test on Nov–Dec 2023. Temporal separation is essential for time-sensitive predictions like churn.

---

**Key Takeaway:** Use logistic regression for a fast, interpretable baseline. Use boosted tree classifier for better accuracy. Always evaluate on temporally held-out test data and check both precision and recall based on your business cost of errors.

**→ Next:** [6.4 Evaluate & Iterate](04-evaluate-iterate.md)
