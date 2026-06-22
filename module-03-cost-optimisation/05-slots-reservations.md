# 3.5 Slots & Reservations

**TL;DR:** Slots are BigQuery's unit of compute. On flat-rate pricing, you buy a pool of slots shared across all queries. Understanding slot capacity helps you predict performance and control costs at scale.

---

## What you'll learn

- What a slot is and how queries consume them
- Reservation types: on-demand, flex, and committed
- How to allocate slots across teams or workloads

---

## What is a slot?

A slot is a virtual CPU that BigQuery uses to execute query steps. BigQuery automatically parallelises work across as many slots as needed (and available).

- A small query (< 1 GB) typically uses a handful of slots for < 1 second
- A large query (5 TB, complex aggregations) may use thousands of slots simultaneously

On **on-demand** pricing, you share a pool of slots with other Google Cloud customers. You have no guaranteed capacity — BigQuery allocates what it can.

On **flat-rate** pricing, you have dedicated slots. Your queries never compete with other customers.

---

## Reservation types

| Type | Commitment | Cancellable | Best for |
|------|-----------|-------------|----------|
| On-demand | None | N/A | Getting started, ad-hoc, unpredictable volume |
| Flex slots | 60 seconds | After 60s | Short bursts, month-end reporting, peak workloads |
| Standard (1-year) | 1 year | No | Consistent production workloads |
| Enterprise (1/3-year) | Multi-year | No | Maximum discount, large-scale committed use |

---

## Creating a reservation (Console or bq CLI)

```bash
# Create a 500-slot reservation
bq mk \
  --reservation \
  --location=EU \
  --slots=500 \
  my-prod-reservation

# Create an assignment linking a project to the reservation
bq mk \
  --reservation_assignment \
  --reservation_id=my-org:EU.my-prod-reservation \
  --assignee_id=my-project \
  --assignee_type=project \
  --job_type=QUERY
```

Once assigned, queries in `my-project` draw from the 500-slot pool instead of on-demand capacity.

---

## Slot utilisation monitoring

```sql
-- See slot utilisation over the last 7 days
SELECT
  TIMESTAMP_TRUNC(period_start, HOUR) AS hour,
  SUM(slot_ms) / 1000 AS total_slot_seconds,
  SUM(slot_ms) / 1000 / 3600 AS avg_slots_used
FROM `region-eu.INFORMATION_SCHEMA.JOBS_TIMELINE_BY_PROJECT`
WHERE period_start > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
GROUP BY 1
ORDER BY 1;
```

If `avg_slots_used` consistently approaches your reservation size, queries are queuing — time to add slots or optimise queries.

---

## Workload management: splitting reservations

You can create multiple reservations and assign them to different projects or job types:

```
my-org reservation pool
├── prod-queries (300 slots) → assigned to project: company-data-prod
├── dev-queries  (100 slots) → assigned to project: company-data-dev
└── ml-training  (200 slots) → assigned to project: company-ml-training
```

This prevents a runaway dev query from starving production dashboards.

---

## On-demand idle slot sharing

If you have a committed reservation but query volume is low at night, you can configure **idle capacity sharing** — unused slots from your reservation become available to on-demand queries from other users, earning credits back. This is optional and configurable.

---

## QE Tip

Before buying a reservation, query `INFORMATION_SCHEMA.JOBS` to understand your actual slot consumption patterns over 30 days. Many teams over-provision because they measure peak load rather than average load. Flex slots can handle peak demand cheaply without committing to year-round capacity.

---

**Key Takeaway:** On flat-rate, you buy slots — dedicated compute shared across your queries. Use multiple reservations to isolate production from development, and monitor slot utilisation before committing to capacity.

**→ Next:** [3.6 QE: Cost Regression Testing](06-qe-cost-testing.md)
