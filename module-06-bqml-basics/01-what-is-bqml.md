# 6.1 What is BigQuery ML?

**TL;DR:** BigQuery ML (BQML) lets you train, evaluate, and serve ML models using SQL — without moving data out of BigQuery. Great for analysts who know SQL, less so for teams with complex modelling needs.

---

## What you'll learn

- What BQML can do and how it fits into the ML landscape
- Supported model types
- When to use BQML vs Vertex AI

---

## The core idea

Training an ML model traditionally requires: exporting data, setting up a Python environment, a training script, a separate serving infrastructure, and a way to get predictions back into BigQuery.

BQML compresses this to SQL:

```sql
-- Train
CREATE MODEL my_dataset.my_model OPTIONS (model_type = 'linear_reg') AS
SELECT feature1, feature2, label FROM training_data;

-- Evaluate
SELECT * FROM ML.EVALUATE(MODEL my_dataset.my_model, TABLE my_dataset.test_data);

-- Predict
SELECT * FROM ML.PREDICT(MODEL my_dataset.my_model, TABLE my_dataset.new_data);
```

Your data never leaves BigQuery. No Python, no infrastructure, no data exports.

---

## Supported model types

| Category | Model types |
|----------|------------|
| Regression | Linear regression, DNN regression, boosted tree regression |
| Classification | Logistic regression, DNN classifier, boosted tree classifier, random forest |
| Clustering | K-means |
| Time series | ARIMA_PLUS (forecasting) |
| Recommendation | Matrix factorisation |
| Anomaly detection | Autoencoder |
| NLP (via remote) | Text classification, text embedding, sentiment |
| Tabular (AutoML) | AutoML Tables (via Vertex) |

BQML also supports **remote models** — call Gemini, PaLM, or your own Vertex AI endpoint directly from SQL (covered in Module 7).

---

## When BQML makes sense

**Good fit:**
- You already have data in BigQuery and want a quick baseline model
- Your team is SQL-fluent but not Python-fluent
- The use case is standard: churn prediction, demand forecast, customer segmentation
- You want predictions back in BigQuery for downstream SQL queries
- Rapid prototyping before deciding whether to invest in a full ML platform

**Not a good fit:**
- You need deep learning with custom architectures
- Your model requires training data that can't fit in BigQuery
- You need real-time inference at low latency (< 100ms) — BQML is batch-oriented
- You need fine-grained control over hyperparameter tuning
- You already have a Python/Scikit-learn/TensorFlow codebase and team

---

## The BQML workflow

```
1. Prepare training data → BigQuery table with features + label column
2. CREATE MODEL with OPTIONS → train the model (stored in BigQuery)
3. ML.EVALUATE → check model quality metrics
4. Iterate: adjust features, hyperparameters, try different model types
5. ML.PREDICT → generate predictions on new data
6. (Optional) ML.EXPORT_MODEL → export to Vertex AI for serving
```

Each step is a SQL statement. The model itself is stored as a BigQuery object, visible in the dataset alongside tables.

---

## Cost

BQML charges for training compute separately from query costs:
- Linear/logistic regression: typically a few dollars for millions of rows
- Boosted trees, DNN: more expensive — check the pricing page for your model type
- Predictions: charged at regular on-demand query rates (bytes scanned)

Training is not free, but it's usually much cheaper than setting up and running a Vertex AI training job for the same task.

---

## QE Tip

Treat a BQML model like any other production artefact: it needs a training/test split, held-out evaluation metrics, and a documented performance baseline. A model that "works" with no evaluation metrics is not production-ready — it's a prototype with an uncertain expiry date.

---

**Key Takeaway:** BQML lets you train, evaluate, and predict with SQL. It's best for SQL-fluent teams doing standard ML tasks where data is already in BigQuery. For complex or real-time use cases, Vertex AI is the better choice.

**→ Next:** [6.2 Linear Regression](02-linear-regression.md)
