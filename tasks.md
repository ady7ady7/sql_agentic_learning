# Daily SQL Practice Tasks

**Generated:** 2026-03-31
**Week 16, Day 2 Focus:** NULLIF + Advanced Window Functions + Complex GROUP BY

---

## Task 1: NULLIF — Clean Revenue Averages

**Scenario:**
The finance team wants average transaction amounts per type, but the data contains some transactions with amount = 0 (placeholder entries that should be excluded from averages, not treated as real zero-value transactions).

Calculate the average transaction amount per type, excluding both NULLs and zero-amount entries.

Then show what the average would be **including** zeros — so both values appear side by side to demonstrate the difference.

**Expected Output Columns:**
- `type` (text)
- `avg_excl_zeros` (numeric) — average excluding 0-amount entries, rounded to 2 decimals
- `avg_incl_zeros` (numeric) — average including 0-amount entries (but still excluding NULLs), rounded to 2 decimals
- `zero_count` (bigint) — how many zero-amount entries exist for this type
- `transaction_count` (bigint) — total non-NULL transactions for this type

**Requirements:**
- Use `transactions` table
- Use `AVG(NULLIF(amount, 0))` for `avg_excl_zeros`
- Use `AVG(amount)` for `avg_incl_zeros` (NULLs excluded automatically by AVG)
- Order by `type ASC`

**Difficulty Rating:** 3/5

SELECT 
	TYPE,
	ROUND(AVG(nullif(amount, 0)), 2) AS avg_excl_zeros,
	ROUND(AVG(t.amount), 2) AS avg_incl_zeros,
	COUNT(*) AS transactions_count
FROM crappy_data_db.transactions t
GROUP BY TYPE
ORDER BY type


---

## Task 2: Advanced Window Functions — Transaction Percentile Distribution

**Scenario:**
The analytics team wants a full percentile breakdown of transaction amounts per type. For each transaction, show:
- Its percentile rank within its type (0.0 to 1.0)
- Which quartile it belongs to (1–4)
- How far it deviates from the type's mean in standard deviations (z-score)
- The running cumulative amount within its type ordered by amount ASC

**Expected Output Columns:**
- `id` (integer)
- `type` (text)
- `amount` (numeric)
- `percentile_rank` (numeric) — `PERCENT_RANK()` rounded to 3 decimals
- `quartile` (integer) — `NTILE(4)`
- `z_score` (numeric) — `(amount - AVG) / STDDEV`, rounded to 2 decimals
- `cumulative_amount` (numeric) — running SUM within type ordered by amount ASC, rounded to 2 decimals

**Requirements:**
- Use `transactions` table, exclude NULL amounts
- All window functions partitioned by `type`, ordered by `amount ASC`
- Exclude rows where `STDDEV = 0` (all amounts identical — z-score undefined)
- Order by `type ASC`, `amount ASC`

**Difficulty Rating:** 4/5

WITH transactions_perc_rank_quartiles AS (
SELECT 
	id,
	TYPE,
	amount,
	ROUND(percent_rank() OVER (PARTITION BY TYPE ORDER BY amount)::numeric, 3) AS percentile_rank,
	ntile(4) OVER (PARTITION BY TYPE ORDER BY amount DESC) AS quartile,
	stddev(amount) OVER (PARTITION BY TYPE) AS std_dev
FROM crappy_data_db.transactions t
ORDER BY percentile_rank DESC
),
transaction_types_avgs AS (
SELECT
	TYPE,
	ROUND(AVG(amount), 2) AS avg_type_amt
FROM transactions_perc_rank_quartiles
GROUP BY TYPE
)
SELECT 
	tpr.id,
	tpr.TYPE,
	tpr.amount,
	tpr.percentile_rank,
	tpr.quartile,
	(tpr.amount - tta.avg_type_amt) / std_dev AS z_score,
	SUM(tpr.amount) OVER (PARTITION BY tpr.TYPE ORDER BY tpr.amount) AS cumulative_amount
FROM transaction_types_avgs tta
JOIN transactions_perc_rank_quartiles tpr ON tta."type"  = tpr."type"


---

## Task 3: Complex GROUP BY — Order Revenue by Country and Age Group

**Scenario:**
The growth team wants to understand revenue distribution across countries and user age groups, but only for countries with at least 3 distinct ordering users.

**Expected Output Columns:**
- `country` (text)
- `age_group` (text) — `'under_30'`, `'30_to_50'`, `'over_50'`
- `order_count` (bigint)
- `total_revenue` (numeric) — rounded to 2 decimals
- `avg_revenue_per_order` (numeric) — rounded to 2 decimals, use `NULLIF` to guard against division by zero
- `pct_of_country_revenue` (numeric) — this age group's revenue as % of total country revenue, rounded to 1 decimal

**Requirements:**
- Use `orders` and `users` tables only
- Exclude NULL countries and NULL ages
- Only include countries where `COUNT(DISTINCT user_id) >= 3` — apply via HAVING
- `pct_of_country_revenue` requires a window SUM partitioned by country
- Order by `country ASC`, `total_revenue DESC`

**Difficulty Rating:** 4/5

WITH countries_user_cnt AS (
SELECT 
	country,
	COUNT(*) AS user_cnt
FROM crappy_data_db.users u
WHERE country IS NOT NULL
GROUP BY country
),
users_age_groups AS (
SELECT 
	u.id,
	u.country,
	CASE WHEN u.age < 30 THEN 'under_30' WHEN u.age >= 30 AND u.age <= 50 THEN '30_to_50' ELSE 'over_50' END AS age_group
FROM countries_user_cnt cu
JOIN crappy_data_db.users u ON cu.country = u.country
WHERE cu.user_cnt >= 3
),
orders_countries_age_stats AS (
SELECT 
	ua.country,
	ua.age_group,
	COUNT(o.id) AS order_count,
	SUM(o.amount) AS total_revenue,
	ROUND(AVG(NULLIF(o.amount, 0))::NUMERIC, 2) AS avg_revenue_per_order
FROM users_age_groups ua
JOIN crappy_data_db.orders o ON ua.id = o.user_id
GROUP BY ua.country, ua.age_group
),
countries_total_revenues AS (
SELECT 
	country,
	ROUND(SUM(total_revenue)::NUMERIC, 2) AS total_country_revenue
FROM orders_countries_age_stats
GROUP BY country
)
SELECT 
	oc.country,
	oc.age_group,
	oc.order_count,
	oc.total_revenue,
	oc.avg_revenue_per_order,
	ct.total_country_revenue,
	ROUND(oc.total_revenue::NUMERIC / ct.total_country_revenue * 100::NUMERIC, 1) AS pct_of_country_revenue
FROM orders_countries_age_stats oc
JOIN countries_total_revenues ct ON oc.country = ct.country
ORDER BY country, total_revenue DESC

I didn't need to use HAVING here, I used simple counts at the beginning instead.

---

## Submission Instructions

1. Task 1 — NULLIF clean averages (3/5)
2. Task 2 — Transaction percentile distribution (4/5)
3. Task 3 — Revenue by country and age group (4/5)
