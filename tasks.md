# Daily SQL Practice Tasks

**Generated:** 2026-05-20
**Week 23, Day 3 Focus:** Cohort retention retry + window functions depth + cross-schema

---

## Task 1: Cohort LEFT JOIN — 3-Month Retention (Retry)

**Scenario:**
For each registration cohort (month), find how many users placed at least one order during months 1–3 after registration (not including the registration month itself).

Show per cohort:
- `cohort_month` — DATE_TRUNC of users.created_at to month
- `cohort_size` — total users registered that month
- `retained_users` — distinct users who ordered in months 1, 2, or 3 after registration
- `retention_rate` — `retained_users / cohort_size` as decimal, rounded to 2

Order by `cohort_month ASC`.

**Tables:** `crappy_data_db.users`, `crappy_data_db.orders`

**Scaffolding — read carefully before writing:**

**Boundary logic (draw this on paper if needed):**
- Cohort month start = `DATE_TRUNC('month', users.created_at)` — call it `cohort_month`
- Month 0 = the registration month itself → **exclude**
- Month 1 starts at: `cohort_month + INTERVAL '1 month'`
- Month 3 ends at (exclusive): `cohort_month + INTERVAL '4 months'`
- So the JOIN condition is: `o.created_at >= cohort_month + INTERVAL '1 month' AND o.created_at < cohort_month + INTERVAL '4 months'`

**LEFT JOIN vs WHERE trap:**
- Put the date range condition in the JOIN ON clause, NOT in WHERE
- If you filter `WHERE o.id IS NOT NULL`, you've turned the LEFT JOIN into an INNER JOIN — cohorts with 0 retained users will disappear from results
- To keep all cohorts: use `COUNT(DISTINCT o.user_id)` — it naturally returns 0 when no orders match (NULLs are ignored by COUNT)

**Retention rate:**
- Retention = how many stayed → `retained_users / cohort_size`
- Churn = how many left → `(cohort_size - retained_users) / cohort_size`
- Make sure you're dividing retained, not the difference

**Difficulty Rating:** 5/5

---

## Task 2: Cumulative SUM with Frame — Running Revenue per User

**Scenario:**
For each order, show the running total revenue for that user up to and including that order.

Show:
- `user_id`
- `order_id`
- `created_at`
- `amount`
- `running_total` — cumulative SUM of amount for that user, ordered by created_at ASC, using an explicit window frame

Use an explicit frame: `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`

Exclude NULL amounts. Order by `user_id ASC`, `created_at ASC`.

**Tables:** `crappy_data_db.orders`

**Difficulty Rating:** 3/5

---

## Task 3: Cross-Schema — Top Seniority Levels by Polish City Users

**Scenario:**
Which seniority levels are most commonly listed for job offers in cities where we also have registered users?

Join `crappy_data_db.users` (city) to `job_db.oferty` (miasto) to find offers in cities that appear in our user base. Then aggregate by seniority level.

Show:
- `seniority` — nazwa from job_db.seniority
- `offer_count` — number of offers for that seniority in matched cities

Exclude NULL seniority_id, NULL miasto, NULL city. Only include cities that appear in both tables (INNER JOIN on city = miasto). Order by `offer_count DESC`.

**Tables:** `crappy_data_db.users`, `job_db.oferty`, `job_db.seniority`

**Difficulty Rating:** 4/5

---

## Submission Instructions

1. Task 1 — Cohort 3-month retention retry (5/5)
2. Task 2 — Running cumulative SUM with explicit frame (3/5)
3. Task 3 — Cross-schema seniority by shared cities (4/5)
