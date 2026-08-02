# BigQuery Bite-Size Course

A self-paced course designed for mobile reading. Each lesson is 5–7 minutes. Work through modules in order or jump to what you need.

---

## Module 1: Foundations

| # | Lesson | Summary |
|---|--------|---------|
| ✅ 1.1 | [What is BigQuery](module-01-foundations/01-what-is-bigquery.md) | Serverless warehouse, columnar storage, when to use it |
| ✅ 1.2 | [Projects, Datasets, Tables](module-01-foundations/02-projects-datasets-tables.md) | Resource hierarchy and permissions model |
| ✅ 1.3 | [Your First Queries](module-01-foundations/03-first-queries.md) | SELECT, filter, aggregate — and understanding cost |

## Module 2: Partitioning & Clustering

| # | Lesson | Summary |
|---|--------|---------|
| ✅ 2.1 | [Why Partition?](module-02-partitioning-clustering/01-why-partition.md) | Query pruning: read less, pay less, go faster |
| ✅ 2.2 | [Time-Based Partitioning](module-02-partitioning-clustering/02-time-based-partitioning.md) | DATE/TIMESTAMP columns, expiry, partition filters |
| ✅ 2.3 | [Other Partition Types](module-02-partitioning-clustering/03-other-partition-types.md) | Integer range and ingestion-time partitioning |
| ✅ 2.4 | [Clustering](module-02-partitioning-clustering/04-clustering.md) | Sorted blocks, clustering keys, when it helps |
| ✅ 2.5 | [Partition + Cluster Together](module-02-partitioning-clustering/05-combine-partition-cluster.md) | The winning combo for most production tables |
| 2.6 | [QE: Testing Partitioned Tables](module-02-partitioning-clustering/06-qe-testing-partitions.md) | Validate pruning works and partitions are healthy |

## Module 3: Cost Optimisation

| # | Lesson | Summary |
|---|--------|---------|
| 3.1 | [How Pricing Works](module-03-cost-optimisation/01-pricing-model.md) | On-demand vs flat-rate, bytes billed explained |
| 3.2 | [Dry Runs & Estimation](module-03-cost-optimisation/02-dry-runs-estimation.md) | Check cost before running, in the UI and CLI |
| 3.3 | [Expensive Query Patterns to Avoid](module-03-cost-optimisation/03-avoid-expensive-queries.md) | SELECT *, LIMIT misconception, cross joins |
| 3.4 | [Materialized Views](module-03-cost-optimisation/04-materialized-views.md) | Pre-aggregate results that refresh automatically |
| 3.5 | [Slots & Reservations](module-03-cost-optimisation/05-slots-reservations.md) | Flat-rate pricing, flex slots, capacity planning |
| 3.6 | [QE: Cost Regression Testing](module-03-cost-optimisation/06-qe-cost-testing.md) | Catch query cost explosions before they hit prod |

## Module 4: Quality Engineering in BigQuery

| # | Lesson | Summary |
|---|--------|---------|
| 4.1 | [Why Data Quality Matters](module-04-quality-engineering/01-why-data-quality.md) | Silent failures cost more than noisy ones |
| 4.2 | [Built-in Validation Techniques](module-04-quality-engineering/02-built-in-validation.md) | ASSERT, constraints, INFORMATION_SCHEMA checks |
| 4.3 | [dbt for BigQuery](module-04-quality-engineering/03-dbt-basics.md) | Models, tests, documentation in 5 minutes |
| 4.4 | [Data Contracts & Expectations](module-04-quality-engineering/04-data-contracts.md) | Schema agreements between producer and consumer |
| 4.5 | [Monitoring Freshness & Completeness](module-04-quality-engineering/05-monitoring-freshness.md) | Know when data is late or missing before users do |

## Module 5: De-risking Changes in BigQuery

| # | Lesson | Summary |
|---|--------|---------|
| 5.1 | [Schema Evolution Strategies](module-05-derisking-changes/01-schema-evolution.md) | Backward-compatible changes, NULLABLE vs REQUIRED |
| 5.2 | [Blue-Green Table Swaps](module-05-derisking-changes/02-blue-green-tables.md) | Atomic cutover with zero downtime |
| 5.3 | [Snapshots & Time Travel](module-05-derisking-changes/03-snapshots-time-travel.md) | Recover from bad writes with FOR SYSTEM_TIME AS OF |
| 5.4 | [Safe Migration Patterns](module-05-derisking-changes/04-safe-migrations.md) | Shadow writes, parallel validation, gradual cutover |
| 5.5 | [Change Testing Checklist](module-05-derisking-changes/05-change-testing-checklist.md) | Before/after checks for every production change |

## Module 6: BigQuery ML Basics

| # | Lesson | Summary |
|---|--------|---------|
| 6.1 | [What is BQML?](module-06-bqml-basics/01-what-is-bqml.md) | Train ML models with SQL, when BQML makes sense |
| 6.2 | [Linear Regression](module-06-bqml-basics/02-linear-regression.md) | Predict numeric outcomes: CREATE MODEL walkthrough |
| 6.3 | [Classification Models](module-06-bqml-basics/03-classification.md) | Logistic regression and boosted trees |
| 6.4 | [Evaluate & Iterate](module-06-bqml-basics/04-evaluate-iterate.md) | ML.EVALUATE, confusion matrix, improving accuracy |
| 6.5 | [Export to Vertex AI](module-06-bqml-basics/05-export-to-vertex.md) | Take your model out of BigQuery for serving |

## Module 7: BigQuery for ML & GenAI

| # | Lesson | Summary |
|---|--------|---------|
| 7.1 | [Feature Engineering in SQL](module-07-ml-genai/01-feature-engineering-sql.md) | Window functions, date features, entity aggregations |
| 7.2 | [Embeddings with BQML](module-07-ml-genai/02-embeddings-bqml.md) | Generate text embeddings for semantic search |
| 7.3 | [Vector Search](module-07-ml-genai/03-vector-search.md) | Find nearest neighbours with VECTOR_SEARCH |
| 7.4 | [Remote Models & GenAI](module-07-ml-genai/04-remote-models-genai.md) | Call Gemini directly from SQL |
| 7.5 | [RAG Patterns with BigQuery](module-07-ml-genai/05-rag-patterns.md) | Retrieval-augmented generation using BQ as the store |
| 7.6 | [Vertex AI + BigQuery Integration](module-07-ml-genai/06-vertex-bigquery-integration.md) | Feature Store, pipelines, and the BQ connector |

## Module 8: Advanced Topics

| # | Lesson | Summary |
|---|--------|---------|
| 8.1 | [Row-Level Security](module-08-advanced/01-row-level-security.md) | Row access policies: filter what users can see |
| 8.2 | [Scheduled Queries & Workflows](module-08-advanced/02-scheduled-queries.md) | Automate recurring transforms inside BigQuery |
| 8.3 | [Data Lineage & Audit Logs](module-08-advanced/03-data-lineage-audit.md) | Track what ran, who changed what, and when |
| 8.4 | [Streaming vs Batch Ingestion](module-08-advanced/04-streaming-vs-batch.md) | Pick the right loading strategy for your use case |

---

## How to Use This Course

- **New to BigQuery?** Start at Module 1 and work through in order.
- **On-demand?** Jump to the module you need — each lesson is self-contained.
- **QE angle:** Every lesson has a QE Tip that connects the topic to testing and data quality.
- **SQL examples:** All SQL is runnable in the BigQuery Console against public datasets where possible.
