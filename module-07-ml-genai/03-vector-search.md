# 7.3 Vector Search

**TL;DR:** `VECTOR_SEARCH` finds the most semantically similar items in BigQuery by comparing embedding vectors. It's the engine behind semantic search and recommendation — no external vector database needed.

---

## What you'll learn

- How VECTOR_SEARCH works in BigQuery
- Building a semantic search system from scratch
- Approximate vs exact search and when each applies

---

## The core concept

Vector search answers: "Given this query embedding, which rows in the table have the most similar embeddings?"

Similarity is measured by **cosine distance** (or dot product): vectors pointing in the same direction are similar, opposite directions are dissimilar.

Use cases: semantic search, "more like this" recommendations, finding duplicate or near-duplicate content, retrieving context for RAG.

---

## Basic VECTOR_SEARCH

```sql
-- Find the 5 support tickets most similar to a query ticket
SELECT
  base.ticket_id,
  base.ticket_text,
  distance
FROM VECTOR_SEARCH(
  TABLE `project.ml.support_ticket_embeddings`,   -- the corpus to search
  'embedding',                                      -- the embedding column
  (
    -- The query: embed the search text first
    SELECT ml_generate_embedding_result AS embedding
    FROM ML.GENERATE_EMBEDDING(
      MODEL `project.ml.text_embedding_model`,
      (SELECT 'Customer cannot log in after password reset' AS content)
    )
  ),
  top_k => 5,
  distance_type => 'COSINE'
);
```

Results include `distance`: 0 = identical, 1 = completely dissimilar (for cosine distance).

---

## Full semantic search pipeline

A more realistic setup: embed a user's search query at query time and find the top results from a pre-embedded corpus.

```sql
-- 1. Embed the user's query
WITH query_embedding AS (
  SELECT ml_generate_embedding_result AS embedding
  FROM ML.GENERATE_EMBEDDING(
    MODEL `project.ml.text_embedding_model`,
    (SELECT @search_query AS content)   -- parameterised query
  )
),

-- 2. Search the corpus
search_results AS (
  SELECT
    base.article_id,
    base.title,
    base.summary,
    distance
  FROM VECTOR_SEARCH(
    TABLE `project.knowledge_base.article_embeddings`,
    'embedding',
    TABLE query_embedding,
    top_k => 10,
    distance_type => 'COSINE'
  )
)

-- 3. Return ranked results
SELECT article_id, title, summary, ROUND(1 - distance, 3) AS similarity_score
FROM search_results
ORDER BY distance ASC;
```

---

## Creating a vector index for faster search

For tables with millions of embeddings, create a vector index:

```sql
CREATE VECTOR INDEX articles_embedding_idx
ON `project.knowledge_base.article_embeddings`(embedding)
OPTIONS (
  index_type = 'IVF',             -- Inverted File Index
  distance_type = 'COSINE',
  ivf_options = '{"num_lists": 500}'  -- more lists = faster but less accurate
);
```

With an index, `VECTOR_SEARCH` uses approximate nearest neighbour (ANN) search — faster but returns approximate results. Without an index, it uses exact search (slower but 100% accurate).

For RAG use cases where missing a relevant document is costly, consider exact search on smaller corpora (< 1M rows) or tuning `num_lists` to balance accuracy and speed.

---

## Filtering alongside vector search

Combine vector similarity with SQL filters:

```sql
SELECT base.*, distance
FROM VECTOR_SEARCH(
  (
    -- Pre-filter the corpus before searching
    SELECT * FROM `project.knowledge_base.article_embeddings`
    WHERE category = 'billing'           -- only search billing articles
      AND published_date >= '2023-01-01' -- only recent content
  ),
  'embedding',
  TABLE query_embedding,
  top_k => 10
);
```

Pre-filtering the corpus reduces search space and improves both speed and relevance.

---

## QE Tip

Test vector search quality by building an evaluation set: 20–50 query/expected-result pairs representative of your use case. Run the search on each query and measure how often the expected result appears in the top 5 or top 10. This gives you a reproducible quality score you can track over time as you change embedding models, corpus content, or index settings.

---

**Key Takeaway:** `VECTOR_SEARCH` finds semantically similar items in BigQuery without an external vector database. Create a vector index for large corpora and combine with SQL filters to narrow the search space before comparing vectors.

**→ Next:** [7.4 Remote Models & GenAI](04-remote-models-genai.md)
