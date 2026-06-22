# 4.2 Built-in Validation Techniques

**TL;DR:** BigQuery has several native mechanisms to validate data — from SQL-level assertions to schema metadata checks. Use them before reaching for an external framework.

---

## What you'll learn

- ASSERT for inline validation in SQL scripts
- Table constraints (NOT NULL, PRIMARY KEY, FOREIGN KEY)
- INFORMATION_SCHEMA for schema drift detection

---

## ASSERT: fail fast in SQL scripts

`ASSERT` raises an error if a condition is false. Use it in your data pipeline scripts to stop execution when data is unexpected:

```sql
-- Fail if no rows loaded for today
ASSERT (
  SELECT COUNT(*) FROM `project.analytics.events`
  WHERE event_date = CURRENT_DATE()
) > 0
AS 'events table has zero rows for today — pipeline may have failed';

-- Fail if duplicate primary keys exist
ASSERT (
  SELECT COUNT(*) = COUNT(DISTINCT order_id)
  FROM `project.sales.orders`
  WHERE order_date = CURRENT_DATE()
) AS 'Duplicate order_ids detected in today''s load';
```

If the assertion fails, the `ASSERT` statement raises an error and stops the script. Anything after it in a multi-statement transaction doesn't run.

---

## Inline validation in multi-statement scripts

BigQuery supports multi-statement transactions, so you can validate before committing:

```sql
BEGIN TRANSACTION;

-- Stage the data
INSERT INTO `project.sales.orders_staging`
SELECT * FROM `project.raw.orders_today`;

-- Validate before promoting
ASSERT (
  SELECT COUNT(*) FROM `project.sales.orders_staging`
) > (
  SELECT COUNT(*) FROM `project.sales.orders`
  WHERE order_date = CURRENT_DATE()
) AS 'Staging has fewer rows than current prod — rejecting load';

-- Promote only if validation passes
INSERT INTO `project.sales.orders`
SELECT * FROM `project.sales.orders_staging`;

COMMIT TRANSACTION;
```

If the ASSERT fails, the transaction rolls back — nothing gets written to production.

---

## Table constraints

BigQuery supports constraint declarations (informational — not enforced at write time):

```sql
CREATE TABLE `project.sales.orders`
(
  order_id    STRING NOT NULL,
  customer_id STRING NOT NULL,
  order_date  DATE   NOT NULL,
  total_usd   NUMERIC,
  PRIMARY KEY (order_id) NOT ENFORCED,
  FOREIGN KEY (customer_id) REFERENCES customers(customer_id) NOT ENFORCED
);
```

`NOT ENFORCED` means BigQuery records these constraints in metadata but doesn't check them on insert. This is still useful:
- Documentation — schema consumers know the intended uniqueness
- Query optimiser hints — the BigQuery query engine can use PK/FK hints to generate better join plans
- Tooling — dbt and other tools can read these constraints to generate tests automatically

---

## INFORMATION_SCHEMA for schema drift detection

When a source schema changes unexpectedly, detect it before the pipeline crashes:

```sql
-- Check if expected columns are still present
SELECT
  column_name,
  data_type,
  is_nullable
FROM `project.dataset.INFORMATION_SCHEMA.COLUMNS`
WHERE table_name = 'events'
ORDER BY ordinal_position;
```

Run this in CI before executing a transformation that depends on specific columns:

```python
expected_columns = {'event_id': 'STRING', 'user_id': 'STRING', 'event_date': 'DATE'}

actual = {
    row.column_name: row.data_type
    for row in client.query("""
        SELECT column_name, data_type
        FROM project.dataset.INFORMATION_SCHEMA.COLUMNS
        WHERE table_name = 'events'
    """)
}

missing = set(expected_columns) - set(actual)
type_mismatches = {
    col for col, dtype in expected_columns.items()
    if col in actual and actual[col] != dtype
}

assert not missing, f"Missing columns: {missing}"
assert not type_mismatches, f"Type mismatches: {type_mismatches}"
```

---

## QE Tip

ASSERT is your cheapest validation tool — it runs in the same SQL transaction as your pipeline and costs only the bytes the query scans. Use it for the three most critical invariants on every table: row count within expected range, no NULLs in NOT NULL columns, no duplicate primary keys. These three checks catch 80% of pipeline failures.

---

**Key Takeaway:** Use ASSERT for fail-fast validation inside pipeline scripts, table constraints for documentation and optimiser hints, and INFORMATION_SCHEMA checks for schema drift detection before transformations run.

**→ Next:** [4.3 dbt for BigQuery](03-dbt-basics.md)
