# 7.5 RAG Patterns with BigQuery

**TL;DR:** Retrieval-Augmented Generation (RAG) grounds LLM answers in your actual data. BigQuery can serve as the retrieval layer — embedding your knowledge base, finding relevant chunks via vector search, and passing them to Gemini to answer questions.

---

## What you'll learn

- The three steps of RAG: Retrieve, Augment, Generate
- How to implement each step in BigQuery SQL
- How to build a simple Q&A pipeline over your internal knowledge base

---

## What RAG solves

LLMs know what they were trained on — not your internal documents, product catalogue, or support history. RAG solves this by:
1. **Retrieving** relevant snippets from your data at query time
2. **Augmenting** the prompt with those snippets as context
3. **Generating** an answer that's grounded in your actual data

Without RAG, the LLM hallucinates. With RAG, it answers based on evidence.

---

## Step 1: Build the knowledge base

Chunk your documents and embed them (from Lesson 7.2):

```sql
-- Chunk articles into paragraphs (simplified — real chunking is more nuanced)
CREATE OR REPLACE TABLE `project.knowledge_base.article_chunks` AS
SELECT
  article_id,
  chunk_index,
  chunk_text,
  CONCAT(title, ': ', chunk_text) AS text_for_embedding
FROM `project.knowledge_base.articles`,
  UNNEST(SPLIT(body, '\n\n')) AS chunk_text WITH OFFSET AS chunk_index
WHERE LENGTH(chunk_text) > 100;  -- skip very short paragraphs

-- Generate embeddings
CREATE OR REPLACE TABLE `project.knowledge_base.chunk_embeddings` AS
SELECT
  article_id,
  chunk_index,
  chunk_text,
  ml_generate_embedding_result AS embedding
FROM ML.GENERATE_EMBEDDING(
  MODEL `project.ml.text_embedding_model`,
  TABLE `project.knowledge_base.article_chunks`
);
```

---

## Step 2: Retrieve relevant chunks

Given a user question, embed it and find the most relevant chunks:

```sql
-- Retrieve top 5 chunks most relevant to the user's question
WITH question_embedding AS (
  SELECT ml_generate_embedding_result AS embedding
  FROM ML.GENERATE_EMBEDDING(
    MODEL `project.ml.text_embedding_model`,
    (SELECT @user_question AS content)
  )
)
SELECT
  base.article_id,
  base.chunk_text,
  distance
FROM VECTOR_SEARCH(
  TABLE `project.knowledge_base.chunk_embeddings`,
  'embedding',
  TABLE question_embedding,
  top_k => 5,
  distance_type => 'COSINE'
)
ORDER BY distance ASC;
```

---

## Step 3: Augment and generate

Combine the retrieved chunks into a prompt and call Gemini:

```sql
WITH question_embedding AS (
  SELECT ml_generate_embedding_result AS embedding
  FROM ML.GENERATE_EMBEDDING(
    MODEL `project.ml.text_embedding_model`,
    (SELECT @user_question AS content)
  )
),
retrieved_context AS (
  SELECT STRING_AGG(base.chunk_text, '\n\n---\n\n' ORDER BY distance) AS context
  FROM VECTOR_SEARCH(
    TABLE `project.knowledge_base.chunk_embeddings`,
    'embedding',
    TABLE question_embedding,
    top_k => 5,
    distance_type => 'COSINE'
  )
)
SELECT ml_generate_text_llm_result AS answer
FROM ML.GENERATE_TEXT(
  MODEL `project.ml.gemini_pro`,
  (
    SELECT CONCAT(
      'Answer the question using ONLY the context provided below. ',
      'If the answer is not in the context, say "I don''t have information on that." ',
      '\n\nContext:\n', context,
      '\n\nQuestion: ', @user_question
    ) AS prompt
    FROM retrieved_context
  ),
  STRUCT(0.2 AS temperature, 500 AS max_output_tokens)
);
```

---

## Production considerations

**Chunking strategy:** The quality of RAG depends heavily on how you chunk documents. Too small = insufficient context. Too large = irrelevant noise in the prompt. 200–500 tokens per chunk is a typical starting point.

**Reranking:** Vector search may return semantically similar but less relevant chunks. Consider a reranking step using a cross-encoder model or Gemini itself to score relevance before building the context window.

**Context window limits:** Gemini 1.5 has a large context window, but more context = higher cost. Start with top 3–5 chunks.

**Refreshing embeddings:** Re-embed the knowledge base when articles are updated. Track `last_modified_time` to only re-embed changed documents.

---

## QE Tip

Evaluate RAG quality with a question/answer test set. Write 20–30 questions with known correct answers, run them through your RAG pipeline, and measure answer quality. Track two metrics: **retrieval accuracy** (did the right chunks get retrieved?) and **answer accuracy** (is the generated answer correct?). This separates retrieval bugs from generation bugs — they have different fixes.

---

**Key Takeaway:** BigQuery can host the full RAG pipeline — embed your knowledge base, retrieve with VECTOR_SEARCH, augment the prompt, and generate with ML.GENERATE_TEXT. The quality depends on chunking strategy and evaluation; invest in both.

**→ Next:** [7.6 Vertex AI + BigQuery Integration](06-vertex-bigquery-integration.md)
