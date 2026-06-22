# 7.4 Remote Models & GenAI

**TL;DR:** BigQuery can call Gemini (and other Vertex AI models) directly from SQL. This lets you run batch text generation, classification, and summarisation over entire tables without leaving BigQuery.

---

## What you'll learn

- How to create a remote model that calls Gemini
- Batch text generation with ML.GENERATE_TEXT
- Practical patterns: classification, summarisation, entity extraction

---

## Setting up a remote model

First, create a BigQuery connection to Vertex AI (one-time setup, done in the Cloud Console):

```bash
bq mk --connection \
  --location=EU \
  --connection_type=CLOUD_RESOURCE \
  vertex-ai-connection
```

Then create the remote model in BigQuery:

```sql
CREATE OR REPLACE MODEL `project.ml.gemini_pro`
REMOTE WITH CONNECTION `project.eu.vertex-ai-connection`
OPTIONS (endpoint = 'gemini-1.5-flash-002');
```

`gemini-1.5-flash-002` is fast and cost-effective for batch processing. Use `gemini-1.5-pro` for more complex reasoning tasks.

---

## Pattern 1: Batch text classification

Classify a table of support tickets by category:

```sql
CREATE OR REPLACE TABLE `project.ml.ticket_classifications` AS
SELECT
  ticket_id,
  ticket_text,
  ml_generate_text_llm_result AS category
FROM ML.GENERATE_TEXT(
  MODEL `project.ml.gemini_pro`,
  (
    SELECT
      ticket_id,
      ticket_text,
      CONCAT(
        'Classify this support ticket into exactly one of these categories: ',
        '[billing, technical, account, shipping, other]. ',
        'Respond with only the category name, nothing else. ',
        'Ticket: ', ticket_text
      ) AS prompt
    FROM `project.support.tickets`
    WHERE created_date = CURRENT_DATE()
  ),
  STRUCT(
    0.0 AS temperature,      -- deterministic output
    10  AS max_output_tokens -- short output: just the category
  )
);
```

---

## Pattern 2: Summarisation at scale

Summarise long product reviews:

```sql
SELECT
  review_id,
  product_id,
  ml_generate_text_llm_result AS summary
FROM ML.GENERATE_TEXT(
  MODEL `project.ml.gemini_pro`,
  (
    SELECT
      review_id,
      product_id,
      CONCAT(
        'Summarise this product review in one sentence, focusing on the main sentiment and key reason: ',
        review_text
      ) AS prompt
    FROM `project.catalogue.reviews`
    WHERE LENGTH(review_text) > 200   -- only long reviews worth summarising
      AND review_date >= '2024-01-01'
  ),
  STRUCT(100 AS max_output_tokens, 0.3 AS temperature)
);
```

---

## Pattern 3: Structured entity extraction

Extract structured data from unstructured text:

```sql
SELECT
  ticket_id,
  JSON_VALUE(ml_generate_text_llm_result, '$.product_name')    AS product_name,
  JSON_VALUE(ml_generate_text_llm_result, '$.issue_type')      AS issue_type,
  JSON_VALUE(ml_generate_text_llm_result, '$.urgency')         AS urgency
FROM ML.GENERATE_TEXT(
  MODEL `project.ml.gemini_pro`,
  (
    SELECT
      ticket_id,
      CONCAT(
        'Extract from this support ticket: product name, issue type, and urgency (low/medium/high). ',
        'Respond ONLY with valid JSON in this format: ',
        '{"product_name": "...", "issue_type": "...", "urgency": "..."}. ',
        'Ticket: ', ticket_text
      ) AS prompt
    FROM `project.support.tickets`
    WHERE created_date = CURRENT_DATE()
  ),
  STRUCT(0.0 AS temperature, 100 AS max_output_tokens)
);
```

---

## Cost and rate limits

`ML.GENERATE_TEXT` is billed at Vertex AI token rates:
- Input tokens + output tokens, per Gemini model pricing
- For batch jobs, BigQuery automatically handles retries and rate limit backoff

Tips to control cost:
- Use `gemini-1.5-flash` not `pro` for straightforward classification
- Keep prompts short — input tokens are charged
- Use `max_output_tokens` to cap output length
- Filter to only process rows that actually need LLM processing

---

## QE Tip

LLM outputs are non-deterministic. Even with `temperature = 0`, output format can vary — Gemini may return "billing" one day and "Billing" or "billing issues" the next. Always post-process outputs: trim whitespace, lowercase, and validate against an allowed list before storing. Add a QA check that counts unexpected category values after each batch run.

---

**Key Takeaway:** `ML.GENERATE_TEXT` lets you call Gemini from SQL over entire tables. Use it for batch classification, summarisation, and entity extraction. Always validate and normalise LLM outputs — they are not structured data until you make them structured.

**→ Next:** [7.5 RAG Patterns with BigQuery](05-rag-patterns.md)
