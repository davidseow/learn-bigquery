# 7.2 Embeddings with BQML

**TL;DR:** Embeddings convert text (or images) into numeric vectors that capture semantic meaning. BigQuery ML can generate embeddings for your text data at scale using Google's models — all from SQL.

---

## What you'll learn

- What embeddings are and why they matter for ML and GenAI
- How to generate text embeddings using BQML remote models
- How to store and use embeddings in BigQuery

---

## What are embeddings?

An embedding is a list of numbers (a vector) that represents the meaning of a piece of text. Texts with similar meaning have similar vectors — even if they use different words.

Examples of semantic similarity:
- "customer cancellation request" ↔ "user wants to cancel their account" → similar vectors
- "product defect complaint" ↔ "item arrived broken" → similar vectors
- "product defect complaint" ↔ "annual revenue report" → very different vectors

Embeddings enable: semantic search, clustering similar documents, recommendation systems, and retrieval-augmented generation (RAG).

---

## Step 1: Create a remote model for embeddings

BQML connects to Google's embedding model via a remote model object:

```sql
-- Create a connection to Vertex AI first (done once, in the Cloud Console or bq CLI)
-- Then create the remote model:
CREATE OR REPLACE MODEL `project.ml.text_embedding_model`
REMOTE WITH CONNECTION `project.eu.vertex-ai-connection`
OPTIONS (endpoint = 'text-embedding-005');
```

`text-embedding-005` is Google's current general-purpose text embedding model (768 dimensions). Check the Vertex AI documentation for the latest model name.

---

## Step 2: Generate embeddings

```sql
-- Generate embeddings for a table of support tickets
CREATE OR REPLACE TABLE `project.ml.support_ticket_embeddings`
AS
SELECT
  ticket_id,
  ticket_text,
  ml_generate_embedding_result AS embedding
FROM ML.GENERATE_EMBEDDING(
  MODEL `project.ml.text_embedding_model`,
  (
    SELECT
      ticket_id,
      CONCAT(subject, '. ', body) AS content   -- concatenate relevant text fields
    FROM `project.support.tickets`
    WHERE created_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  ),
  STRUCT(TRUE AS flatten_json_output)
);
```

Each row gets a `embedding` column containing a FLOAT64 ARRAY of 768 values.

---

## Step 3: Inspect the output

```sql
SELECT
  ticket_id,
  ticket_text,
  ARRAY_LENGTH(embedding) AS embedding_dims,
  embedding[OFFSET(0)]    AS first_dimension
FROM `project.ml.support_ticket_embeddings`
LIMIT 5;
```

---

## Cost and throughput

Embedding generation is charged at Vertex AI rates (not BigQuery query rates). For large tables:
- Process in batches — BQML handles batching automatically but has rate limits
- For tables > 100k rows, expect the job to take several minutes
- Consider scheduling overnight embedding refreshes for large corpora

---

## What to do with embeddings

Once you have embeddings stored, you can:

1. **Vector search** — find the most similar items (covered in the next lesson)
2. **Clustering** — group similar documents using K-means BQML model
3. **Classification** — use embeddings as features in a classifier (semantically rich numeric features)
4. **RAG retrieval** — retrieve relevant documents to augment a GenAI prompt

---

## Multimodal embeddings

For images (e.g., product photos):

```sql
CREATE OR REPLACE MODEL `project.ml.image_embedding_model`
REMOTE WITH CONNECTION `project.eu.vertex-ai-connection`
OPTIONS (endpoint = 'multimodalembedding@001');

-- Generate embeddings from GCS image URIs
SELECT *
FROM ML.GENERATE_EMBEDDING(
  MODEL `project.ml.image_embedding_model`,
  (SELECT product_id, image_gcs_uri AS uri FROM `project.catalogue.products`)
);
```

---

## QE Tip

Embeddings are opaque — you can't eyeball a 768-dimensional vector to know if it's correct. Test them by checking that semantically similar text in your domain produces high cosine similarity (> 0.85) and semantically unrelated text produces low similarity (< 0.5). Build a small "sanity set" of known-similar and known-different pairs and run them through the model as a smoke test after any model version change.

---

**Key Takeaway:** BQML generates text (and image) embeddings using Google's models via `ML.GENERATE_EMBEDDING`. Store embeddings in a BigQuery table — they unlock semantic search, clustering, and RAG patterns entirely within BigQuery.

**→ Next:** [7.3 Vector Search](03-vector-search.md)
