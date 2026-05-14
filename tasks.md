# Daily SQL Practice Tasks

**Generated:** 2026-05-14
**Week 22, Day 4 Focus:** LAG on job_db + NTILE spend tiers cross-table + gaps-and-islands posting gaps

---

## Task 1: LAG — Monthly Offer Count vs Previous Month

**Scenario:**
The team wants to track month-over-month changes in job listing volume.

For each month show:
- `month` — DATE_TRUNC to month
- `offer_count` — number of offers listed that month
- `prev_month_count` — offer count from the previous month via LAG(1)
- `mom_diff` — `offer_count - prev_month_count` (NULL if no prior month)

**Tables:** `job_db.oferty`

**Requirements:**
- Exclude NULL `data_wystawienia`
- Order by `month ASC`

**Difficulty Rating:** 3/5

WITH offers_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', o.data_wystawienia) AS month
FROM job_db.oferty o
WHERE o.data_wystawienia IS NOT NULL
),
months_offer_cnt AS (
SELECT 
	MONTH,
	COUNT(*) AS offer_count
FROM offers_months
GROUP BY MONTH
),
month_prev_month_cnt AS (
SELECT 
	*,
	LAG(offer_count) OVER (ORDER BY month) AS prev_month_count
FROM months_offer_cnt
)
SELECT 
	*,
	ABS(offer_count - prev_month_count) AS mom_diff
FROM month_prev_month_cnt


---

## Task 2: NTILE + Cross-Table — Transaction Spend Tiers vs Order Spend

**Scenario:**
The analytics team wants to bucket users into 5 tiers by total transaction amount, then see how their average order spend differs across tiers.

Show:
- `tier` — 1 (lowest) to 5 (highest) transaction spenders
- `user_count` — number of users in that tier
- `avg_order_spend` — average total order spend for users in that tier, rounded to 2 decimals

**Tables:** `crappy_data_db.transactions`, `crappy_data_db.orders`

**Requirements:**
- CTE 1: total transaction amount per user from `transactions` (exclude NULL amounts)
- CTE 2: apply `NTILE(5)` on total transaction amount to assign tiers
- CTE 3: total order spend per user from `orders` (exclude NULL amounts)
- JOIN CTE 2 to CTE 3 on user_id, then GROUP BY tier
- Order by `tier ASC`

**Difficulty Rating:** 4/5

WITH users_transactions AS (
SELECT 
	user_id,
	SUM(t.amount) AS total_transaction_spend
FROM crappy_data_db.transactions t
GROUP BY user_id
),
transaction_spend_tiers AS (
SELECT 
	*,
	NTILE(5) OVER (ORDER BY total_transaction_spend) AS tier
FROM users_transactions 
)
SELECT 
	ts.tier,
	COUNT(DISTINCT(o.user_id)) AS user_count,
	round(AVG(o.amount)::NUMERIC, 2) AS avg_order_spend
FROM crappy_data_db.orders o
JOIN transaction_spend_tiers ts ON o.user_id = ts.user_id
GROUP BY ts.tier
ORDER BY tier






---

## Task 3: Gaps-and-Islands — Monthly Posting Gaps per Platform

**Scenario:**
The team wants to find periods where a platform had no job listings for at least 2 consecutive months — posting gaps that might indicate platform inactivity.

For each gap show:
- `platform` — nazwa from platforma
- `gap_start` — the first month with no listings (the month after the last active month)
- `gap_end` — the last month with no listings (the month before the next active month)
- `gap_months` — length of the gap in months

Only return gaps of 2 or more months. Order by `gap_months DESC`, `platform ASC`.

**Tables:** `job_db.oferty`, `job_db.platforma`

**Requirements:**
- Use DATE_TRUNC('month', data_wystawienia) to get active months per platform
- Use LEAD to find the next active month
- Gap exists where next_active_month > current_month + INTERVAL '1 month'
- Exclude NULL data_wystawienia and NULL platforma_id

**Difficulty Rating:** 5/5

WITH offers_dates AS (
SELECT 
	*,
	date_trunc('MONTH', data_wystawienia) AS month_
FROM job_db.oferty o
WHERE o.data_wystawienia IS NOT NULL
),
offers_dates_prev_months AS (
SELECT 
	*,
	LEAD(month_) OVER (PARTITION BY platforma_id ORDER BY data_wystawienia) AS next_month
FROM offers_dates
),
offers_isnew AS (
SELECT 
	*,
	CASE WHEN next_month IS NULL OR next_month > month_ + INTERVAL '1' MONTH THEN 1 ELSE 0 END AS is_new
FROM offers_dates_prev_months
),
offers_streaks AS (
SELECT 
	*,
	sum(is_new) OVER (PARTITION BY platforma_id ORDER BY data_wystawienia) AS streak_key
FROM offers_isnew
),
platforms_streaks AS (
SELECT 
	platforma_id AS platform,
	streak_key,
	MIN(month_) AS gap_start,
	MAX(month_) AS gap_end,
	ROUND(EXTRACT(EPOCH FROM (MAX(month_) - MIN(month_))/ 2629743), 1) AS months
FROM offers_streaks
GROUP BY platforma_id, streak_key
)
SELECT * FROM platforms_streaks
WHERE months >= 2


Not the simplest task, but I've managed to do it.
FYI: There was literally only one such streak.

---

## Submission Instructions

1. Task 1 — LAG(1) monthly offer count MoM diff (3/5)
2. Task 2 — NTILE(5) transaction tiers + avg order spend per tier (4/5)
3. Task 3 — Gaps-and-islands: monthly posting gaps per platform (5/5)
