# 5.5 Change Testing Checklist

**TL;DR:** A 10-minute pre-deployment checklist that covers the most common failure modes for BigQuery changes. Print it, bookmark it, make it part of your team's deployment ritual.

---

## What you'll learn

- A structured pre-migration and post-migration checklist
- The most commonly skipped steps (and why they matter)
- How to make the checklist repeatable across your team

---

## Before you start any change

**Scope and impact:**
- [ ] Identify all consumers of this table (check `INFORMATION_SCHEMA.JOBS` for queries referencing this table in the last 30 days)
- [ ] Notify consuming teams with: what's changing, when, and what they need to do (if anything)
- [ ] Confirm rollback plan exists and is documented

**Safety net:**
- [ ] Create a named snapshot: `CREATE SNAPSHOT TABLE ... CLONE ...`
- [ ] Record current metrics: row count, partition count, max/min values of key columns
- [ ] Record current timestamp (for `FOR SYSTEM_TIME AS OF` if needed)

**Review:**
- [ ] Schema change reviewed by a teammate
- [ ] Change type confirmed (additive? breaking? requires migration window?)
- [ ] If breaking: migration window has been communicated and agreed

---

## During the change

- [ ] Run in staging/dev first
- [ ] Validate in staging before promoting to production
- [ ] If building a new table in parallel: shadow validation complete, zero discrepancies
- [ ] Pipeline runs to completion without errors

---

## Immediately after cutover

**Structural checks:**
- [ ] `INFORMATION_SCHEMA.COLUMNS` — expected columns present with correct types
- [ ] `INFORMATION_SCHEMA.PARTITIONS` — partition count and structure correct
- [ ] Row count matches expected (compared against pre-migration snapshot)

**Data correctness checks:**
- [ ] Key aggregates match pre-migration values (revenue totals, event counts, user counts)
- [ ] NULL rates on key columns are within expected range
- [ ] Duplicate check on primary key columns passes

**Consumer checks:**
- [ ] Run downstream dbt models: `dbt run --select downstream_model+`
- [ ] Check key dashboards load without errors
- [ ] Spot-check critical reports

---

## After 24–48 hours

- [ ] No anomaly alerts fired
- [ ] Business stakeholders confirm reports look correct
- [ ] Clean up: drop old table, delete snapshot if no longer needed
- [ ] Update documentation (data catalogue, schema file, data contract)

---

## Common skipped steps and their consequences

| Skipped step | Common consequence |
|-------------|-------------------|
| Notifying consumers | A downstream pipeline breaks silently in production |
| Recording row count before | No baseline to compare — hard to detect data loss |
| Schema snapshot | Can't restore to pre-migration state after 7 days |
| Running dbt tests | A model that worked before now fails on the new schema |
| Checking dashboards | Users notice the data is wrong before you do |
| Dropping old tables | Storage accumulates, and "temporary" tables become permanent |

---

## Making this repeatable

Store this checklist as a GitHub issue template or Confluence template that gets created for every production BigQuery change. The checklist only works if it's used consistently — not just on "big" changes.

```markdown
<!-- .github/ISSUE_TEMPLATE/bigquery_migration.md -->
---
name: BigQuery Table Migration
about: Checklist for any production BigQuery schema or pipeline change
---

## Change description

## Tables affected

## Consumer teams notified: [ ] Yes

## Checklist
- [ ] Snapshot created
- [ ] Row count recorded
...
```

---

## QE Tip

The two most commonly skipped steps are "notify consumers" and "drop old tables." The first causes incidents. The second causes storage sprawl — teams discover year-old "temp" tables that no one knows if they can delete. Add table cleanup as a recurring calendar reminder two weeks after every migration.

---

**Key Takeaway:** A 10-minute checklist before every BigQuery change prevents 80% of migration incidents. Make it a team ritual, not an individual responsibility.

**→ Next:** [6.1 What is BQML?](../module-06-bqml-basics/01-what-is-bqml.md)
