# 7.6 Vertex AI + BigQuery Integration

**TL;DR:** BigQuery and Vertex AI are designed to work together. Data lives in BigQuery; models train on Vertex; predictions flow back. Understanding the integration points helps you build robust ML pipelines.

---

## What you'll learn

- The key integration points between BigQuery and Vertex AI
- Vertex AI Feature Store for centralised feature management
- How to connect a Vertex AI pipeline to BigQuery as source and sink

---

## The integration landscape

```
BigQuery                          Vertex AI
─────────────────────────────     ──────────────────────────────
Raw data & feature tables    ──►  Training jobs (Custom, AutoML)
Predictions stored back      ◄──  Deployed model endpoints
BQML remote models           ──►  Vertex AI model garden (Gemini)
Managed datasets             ←──► Vertex AI Datasets
Feature Store                ←──► BigQuery as feature source
```

BigQuery is the data backbone. Vertex AI provides the training platform, model registry, and serving infrastructure.

---

## Reading BigQuery data in a Vertex AI training job

```python
from google.cloud import bigquery, aiplatform
from google.cloud.aiplatform import Dataset

# Export training data from BigQuery to GCS (for large datasets)
bq_client = bigquery.Client()
job = bq_client.extract_table(
    'project.ml.churn_training',
    'gs://my-models-bucket/training/churn_*.csv',
    job_config=bigquery.ExtractJobConfig(destination_format='CSV')
)
job.result()

# Or: use BigQuery Storage API directly in your training script
from google.cloud.bigquery_storage import BigQueryReadClient
# (faster for large datasets — avoids GCS roundtrip)
```

For small-to-medium datasets (< 10 GB), use the BigQuery Python client directly in your training script. For large datasets, export to GCS CSV/Parquet first.

---

## Vertex AI Feature Store

Feature Store centralises feature computation and serves them online (low-latency) and offline (batch training):

```python
from google.cloud.aiplatform import featurestore

# Create a Feature Store
fs = featurestore.Featurestore.create(
    featurestore_id='customer_features',
    online_store_fixed_node_count=1,
    location='europe-west4'
)

# Create an entity type (e.g., customer)
entity_type = fs.create_entity_type(entity_type_id='customer')

# Ingest features from BigQuery
entity_type.ingest_from_bq(
    feature_ids=['sessions_30d', 'days_since_last_login', 'churn_score'],
    feature_time='snapshot_date',
    bq_source_uri='bq://project.ml.customer_features_weekly',
    entity_id_field='customer_id'
)
```

Features stored in Feature Store can be retrieved at training time (batch) or serving time (online, < 10ms). This prevents training-serving skew — the features used to train the model are identical to what the serving pipeline uses.

---

## Writing Vertex AI predictions back to BigQuery

After online predictions are made (e.g., churn scores for logged-in users), write them back to BigQuery for downstream use:

```python
from google.cloud import bigquery
import json

def log_prediction_to_bigquery(customer_id: str, prediction: dict):
    client = bigquery.Client()
    
    row = {
        'customer_id': customer_id,
        'prediction_timestamp': datetime.utcnow().isoformat(),
        'churn_probability': prediction['churn_probability'],
        'model_version': prediction['model_version']
    }
    
    errors = client.insert_rows_json('project.ml.online_predictions', [row])
    if errors:
        raise RuntimeError(f"BigQuery insert failed: {errors}")
```

Centralising predictions in BigQuery lets you join them with behavioural data for A/B analysis, cohort studies, and model performance monitoring.

---

## Vertex AI Pipelines with BigQuery steps

Vertex AI Pipelines (built on Kubeflow) orchestrate multi-step ML workflows. BigQuery components are natively supported:

```python
from kfp.v2 import dsl
from google.cloud.aiplatform import pipeline_jobs
from google_cloud_pipeline_components.v1.bigquery import BigqueryQueryJobOp

@dsl.pipeline(name='churn-training-pipeline')
def churn_pipeline():
    # Step 1: compute features in BigQuery
    feature_job = BigqueryQueryJobOp(
        project='my-company-prod',
        location='EU',
        query="""
            INSERT INTO project.ml.churn_training
            SELECT ... FROM project.analytics.events
            WHERE event_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
        """
    )
    
    # Step 2: train model (custom training job)
    train_job = ...  # depends on feature_job
    
    # Step 3: evaluate and register if better than current
    eval_job = ...  # depends on train_job
```

---

## QE Tip

The most common failure in Vertex + BigQuery integrations is **training-serving skew**: the features computed in the training query differ subtly from features computed at serving time (different window sizes, different NULL handling, different join logic). Use Feature Store to compute features once and serve the same values at training and serving time. Without Feature Store, at minimum keep the feature SQL in a single shared location that both pipelines reference.

---

**Key Takeaway:** BigQuery is Vertex AI's natural data backend. Use Feature Store to avoid training-serving skew, pipeline components for orchestration, and always write predictions back to BigQuery to enable downstream analytics and model monitoring.

**→ Next:** [8.1 Row-Level Security](../module-08-advanced/01-row-level-security.md)
