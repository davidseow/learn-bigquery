# 5.1 Schema Evolution Strategies

**TL;DR:** BigQuery allows some schema changes with zero risk, others require careful coordination, and a few are impossible without a full table rebuild. Know which is which before you touch a production table.

---

## What you'll learn

- Which schema changes are safe vs dangerous
- The difference between NULLABLE and REQUIRED columns
- How to add columns safely without breaking consumers

---

## The schema change risk matrix

| Change | Risk | Approach |
|--------|------|---------|
| Add a NULLABLE column | Zero risk | `ALTER TABLE ADD COLUMN` — safe any time |
| Rename a column | High — breaks all queries | Deprecate-and-add pattern (see below) |
| Change column type (widening) | Medium | `STRING → JSON` works; `INT64 → FLOAT64` may lose precision |
| Change column type (narrowing) | Very high | Never in-place; requires table rebuild |
| Remove a column | High — breaks all queries | Deprecate, wait for consumers to migrate, then remove |
| Add a NOT NULL column | High | Requires default or backfill first |
| Reorder columns | Low — BigQuery is columnar, order is cosmetic | Avoid anyway to not confuse consumers |
| Change partition column | Extreme | Requires full table rebuild |

---

## Adding a NULLABLE column

The only truly safe in-place change:

```sql
ALTER TABLE `project.analytics.events`
ADD COLUMN IF NOT EXISTS device_type STRING;
```

Existing rows get NULL for the new column. All existing queries continue working because they don't reference the new column. Consumers can opt in when ready.

---

## Renaming a column: the deprecate-and-add pattern

Never rename a column in-place. Instead:

**Step 1:** Add the new column and populate it:
```sql
ALTER TABLE `project.sales.orders`
ADD COLUMN IF NOT EXISTS customer_id STRING;

UPDATE `project.sales.orders`
SET customer_id = client_id
WHERE TRUE;
```

**Step 2:** Announce the migration. Give consumers 30+ days to update their queries from `client_id` to `customer_id`.

**Step 3:** Drop the old column after all consumers have migrated:
```sql
ALTER TABLE `project.sales.orders`
DROP COLUMN IF EXISTS client_id;
```

---

## The NULLABLE vs REQUIRED distinction

BigQuery has two nullability modes:
- `NULLABLE`: column can be NULL (the default for `ALTER TABLE ADD COLUMN`)
- `REQUIRED`: column cannot be NULL — enforced at write time by the schema

You can relax `REQUIRED → NULLABLE` with no risk:
```sql
-- Safe: loosening the constraint
ALTER TABLE `project.sales.orders`
ALTER COLUMN order_id DROP NOT NULL;
```

You cannot tighten `NULLABLE → REQUIRED` on an existing column if the column contains NULLs. You'd need to backfill NULLs first, then recreate the table.

---

## Changing the partition column

This is the most disruptive change in BigQuery. The partition column is baked into the table's physical structure — you cannot change it in-place.

The only approach:
1. Create a new table with the correct partition column
2. Copy all data from the old table to the new
3. Validate the new table
4. Swap the table (blue-green pattern — covered in the next lesson)
5. Delete the old table

For a multi-TB table, step 2 can take hours and cost real money. Plan it carefully.

---

## Tracking schema changes in version control

Store your table DDL in a migrations file (like database migration scripts):

```sql
-- migrations/V001__create_events_table.sql
CREATE TABLE IF NOT EXISTS `project.analytics.events` ( ... );

-- migrations/V002__add_device_type_column.sql
ALTER TABLE `project.analytics.events`
ADD COLUMN IF NOT EXISTS device_type STRING;
```

Run migrations in order on each environment. This gives you an audit trail of every structural change.

---

## QE Tip

Before any schema change to a production table, query `INFORMATION_SCHEMA.COLUMN_FIELD_PATHS` or `JOBS` to identify which queries reference the column you're changing. This is your impact blast radius:

```sql
SELECT DISTINCT query
FROM `region-eu.INFORMATION_SCHEMA.JOBS`
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  AND LOWER(query) LIKE '%client_id%'   -- column you're about to rename
LIMIT 50;
```

---

**Key Takeaway:** Adding NULLABLE columns is always safe. Renames and deletions require the deprecate-and-add pattern with a migration window. Changing the partition column requires a full table rebuild.

**→ Next:** [5.2 Blue-Green Table Swaps](02-blue-green-tables.md)
