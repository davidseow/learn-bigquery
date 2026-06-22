# 8.1 Row-Level Security

**TL;DR:** Row access policies in BigQuery let different users see different rows in the same table — without creating separate tables per team or tenant. One table, filtered automatically based on who's querying.

---

## What you'll learn

- How row access policies work
- Creating and managing policies with SQL
- Common use cases: multi-tenant data, regional access control

---

## The problem

You have a `sales_orders` table with data for multiple regions: UK, DE, FR. Your UK analyst should only see UK data. The head of EMEA should see all three. Without row-level security, you either:
- Create separate tables per region (high maintenance)
- Rely on application-level filtering (risky — easy to bypass in ad-hoc queries)
- Trust users to self-filter (they won't always remember)

Row access policies solve this at the database level.

---

## How they work

A row access policy is a filter expression attached to a table. When a user queries the table, BigQuery automatically adds the filter based on who's running the query. The user never sees rows they don't have access to — the filtering is transparent.

---

## Creating a row access policy

```sql
-- UK analysts can only see UK orders
CREATE ROW ACCESS POLICY uk_orders_only
ON `project.sales.orders`
GRANT TO ('group:uk-analysts@company.com')
FILTER USING (region = 'UK');

-- EMEA leads can see all EMEA regions
CREATE ROW ACCESS POLICY emea_orders
ON `project.sales.orders`
GRANT TO ('group:emea-leads@company.com')
FILTER USING (region IN ('UK', 'DE', 'FR', 'NL', 'ES'));

-- Data platform engineers see everything (no filter = all rows)
CREATE ROW ACCESS POLICY all_orders_access
ON `project.sales.orders`
GRANT TO ('group:data-platform@company.com')
FILTER USING (TRUE);
```

Important: **if no policy matches a user, they see zero rows** (default deny). Always ensure there's a catch-all policy for your data team.

---

## Multi-tenant row access

For SaaS products where each tenant's data lives in shared tables:

```sql
-- Each tenant can only query their own data
-- SESSION_USER() returns the email of the currently executing user
-- Joined to a tenant mapping table
CREATE ROW ACCESS POLICY tenant_isolation
ON `project.app.events`
GRANT TO ('domain:company-customers.com')   -- all customer users
FILTER USING (
  tenant_id = (
    SELECT tenant_id
    FROM `project.auth.user_tenant_map`
    WHERE user_email = SESSION_USER()
  )
);
```

`SESSION_USER()` dynamically returns the identity running the query — this makes one policy work for all tenants.

---

## Listing and dropping policies

```sql
-- List all row access policies on a table
SELECT *
FROM `project.dataset.INFORMATION_SCHEMA.ROW_ACCESS_POLICIES`
WHERE table_name = 'orders';

-- Drop a policy
DROP ROW ACCESS POLICY uk_orders_only ON `project.sales.orders`;

-- Drop all policies on a table
DROP ALL ROW ACCESS POLICIES ON `project.sales.orders`;
```

---

## Limitations

- Row access policies slow down queries slightly — the filter is evaluated for every row scan
- Policies cannot use non-deterministic functions in FILTER USING except `SESSION_USER()`
- Views don't inherit the underlying table's row access policies — you need to consider this when creating views over protected tables
- Authorized views bypass row access policies — use with caution

---

## QE Tip

Test row access policies by impersonating users. In the BigQuery Console, use the "Identity" setting to run queries as another principal. Confirm that:
1. A UK analyst sees only UK rows
2. An EMEA lead sees all EMEA rows
3. A user with no policy match sees zero rows
4. A data engineer sees all rows

Write these as automated access-control tests — run them after any policy change to confirm the policy matrix is correct.

---

**Key Takeaway:** Row access policies filter table rows transparently based on who's querying. Use them for regional access control, multi-tenancy, and any use case where different users should see different data from the same table.

**→ Next:** [8.2 Scheduled Queries & Workflows](02-scheduled-queries.md)
