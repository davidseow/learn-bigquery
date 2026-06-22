# 6.5 Export to Vertex AI

**TL;DR:** BQML is great for batch predictions inside BigQuery. When you need real-time serving, online predictions, or integration with an ML platform, export your model to Vertex AI.

---

## What you'll learn

- How to export a BQML model to Google Cloud Storage
- How to import and deploy it on Vertex AI
- When exporting makes sense vs staying in BigQuery

---

## Why export?

BQML predictions run as SQL queries — they're batch-oriented. If you need:
- Predictions in < 100ms (real-time API)
- A REST endpoint your app can call directly
- Integration with Vertex AI pipelines, monitoring, or A/B testing
- Canary deployments and traffic splitting between model versions

...then you need Vertex AI. Export your BQML model there.

---

## Step 1: Export the model to GCS

```sql
EXPORT MODEL `project.ml.churn_boosted_tree`
OPTIONS (
  uri = 'gs://my-company-models/churn/v2/'
);
```

BigQuery writes the model artefacts (in TensorFlow SavedModel format for most model types) to the GCS path. This is a one-line operation.

---

## Step 2: Import to Vertex AI Model Registry

```python
from google.cloud import aiplatform

aiplatform.init(project='my-company-prod', location='europe-west4')

model = aiplatform.Model.upload(
    display_name='churn-boosted-tree-v2',
    artifact_uri='gs://my-company-models/churn/v2/',
    serving_container_image_uri='europe-docker.pkg.dev/vertex-ai/prediction/tf2-cpu.2-12:latest'
)

print(f"Model registered: {model.resource_name}")
```

---

## Step 3: Deploy to an endpoint

```python
endpoint = model.deploy(
    machine_type='n1-standard-2',
    min_replica_count=1,
    max_replica_count=5,        # auto-scales with traffic
    traffic_percentage=100
)

# Make a prediction via the endpoint
prediction = endpoint.predict(
    instances=[{
        'sessions_last_30d': 3,
        'days_since_last_login': 12,
        'feature_adoption_score': 0.4,
        'subscription_plan': 'starter',
        'tenure_days': 180
    }]
)
print(prediction.predictions)
```

---

## BQML model format support

| BQML model type | Export format | Vertex AI compatible? |
|----------------|--------------|----------------------|
| Linear/logistic regression | TF SavedModel | Yes |
| Boosted tree | TF SavedModel | Yes |
| DNN classifier/regressor | TF SavedModel | Yes |
| K-means | TF SavedModel | Yes |
| ARIMA_PLUS (time series) | Not exportable | No (predict in BQ) |
| Remote model | N/A | N/A (it's already external) |

---

## Keeping batch and online predictions consistent

A common pattern: use BQML for daily batch predictions (churn scores for the CRM) and Vertex AI for real-time predictions (churn score at login).

Both must use the same model artefact to avoid inconsistency. Export once, use everywhere:

```
BQML training → export to GCS → Vertex AI (online serving)
                              ↘ BQML ML.PREDICT (batch)
```

You can load the same GCS model artefact back into BigQuery via `CREATE MODEL FROM` if needed:

```sql
CREATE OR REPLACE MODEL `project.ml.churn_from_vertex`
OPTIONS (model_type = 'tensorflow')
FROM 'gs://my-company-models/churn/v2/';
```

---

## QE Tip

Test your exported model's predictions against BQML's `ML.PREDICT` output on the same inputs before deploying. A single-row spot check that the online endpoint returns the same probability as `ML.PREDICT` confirms the export was clean. Discrepancies usually indicate a preprocessing mismatch.

---

**Key Takeaway:** `EXPORT MODEL` is one SQL statement. Use it when you need real-time serving, REST endpoints, or Vertex AI platform features. Keep both online and batch predictions on the same model artefact to avoid version drift.

**→ Next:** [7.1 Feature Engineering in SQL](../module-07-ml-genai/01-feature-engineering-sql.md)
