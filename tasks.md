# Daily SQL Practice Tasks

**Generated:** 2026-04-08
**Week 17, Day 2 Focus:** YoY comparison + LAG offset + Cohort retention (5/5) + NTILE segmentation

---

## Task 1: Year-over-Year Revenue Comparison per Country

**Scenario:**
The finance team wants to compare order revenue between 2024 and 2025 for each country. They want the absolute revenue for each year side by side, the difference, and the percentage change.

**Expected Output Columns:**
- `country` (varchar)
- `revenue_2024` (numeric) — total order revenue in 2024, rounded to 2 decimals
- `revenue_2025` (numeric) — total order revenue in 2025, rounded to 2 decimals
- `revenue_diff` (numeric) — revenue_2025 minus revenue_2024, rounded to 2 decimals
- `pct_change` (numeric) — percentage change from 2024 to 2025, rounded to 1 decimal. Use NULLIF to guard against countries with no 2024 revenue.

**Requirements:**
- Use `orders` JOIN `users`
- Exclude NULL amounts and NULL countries
- Only include countries that have revenue in **both** years
- Order by `pct_change DESC`

**Difficulty Rating:** 3/5

WITH orders_users AS (
SELECT 
	*,
	EXTRACT('Year' FROM o.created_at) AS year_
FROM crappy_data_db.users u
JOIN crappy_data_db.orders o ON u.id = o.user_id
),
orders_countries_revenues AS (
SELECT 
	country,
	SUM(amount) FILTER (WHERE year_ = 2025) AS revenue_2025,
	round(SUM(amount) FILTER (WHERE year_ = 2024)::NUMERIC, 2) AS revenue_2024
FROM orders_users
GROUP BY country
)
SELECT 
	*,
	COALESCE(revenue_2025, 0) - COALESCE(revenue_2024, 0) AS revenue_diff,
	ROUND((COALESCE(revenue_2025::NUMERIC, 0) - COALESCE(revenue_2024, 0)) / revenue_2024::NUMERIC * 100::NUMERIC, 1) AS pct_change
FROM orders_countries_revenues
WHERE revenue_2025 IS NOT NULL AND revenue_2024 IS NOT NULL
ORDER BY pct_change DESC



---

## Task 2: Cohort Retention — Did Users Order in the Month After Registration? (5/5)

**Scenario:**
The growth team wants to measure first-month retention: of all users who registered in a given month, what percentage placed at least one order in the following calendar month?

For example: users who registered in 2024-10 — how many of them placed an order in 2024-11?

**Expected Output Columns:**
- `registration_month` (date) — truncated to month (e.g. 2024-10-01)
- `cohort_size` (integer) — total users registered in that month
- `retained_users` (integer) — users who placed at least one order in the month immediately following their registration month
- `retention_rate` (numeric) — retained_users / cohort_size as a percentage, rounded to 1 decimal

**Requirements:**
- Use `users` and `orders`
- Registration month comes from `users.created_at`
- "Following month" = DATE_TRUNC('month', created_at) + INTERVAL '1 month'
- A user is "retained" if they have at least one order where DATE_TRUNC('month', order.created_at) = their following month
- Only include cohorts with at least 5 users
- Order by `registration_month ASC`

**Difficulty Rating:** 5/5

WITH users_registration_dates AS (
SELECT 
	u.id AS user_id,
	DATE_TRUNC('Month', u.created_at) AS registration_month,
	DATE_TRUNC('Month', u.created_at) + INTERVAL '1 Month' AS next_month
FROM crappy_data_db.users u
WHERE u.created_at IS NOT NULL
),
cohorts AS (
SELECT 
	urd.registration_month,
	COUNT(DISTINCT(urd.user_id)) AS cohort_size,
	COUNT(DISTINCT(urd.user_id)) FILTER (WHERE DATE_TRUNC('Month', o.created_at) = next_month) AS retained_users
FROM users_registration_dates urd
JOIN crappy_data_db.orders o ON urd.user_id = o.user_id
GROUP BY urd.registration_month
)
SELECT 
	*,
	ROUND(retained_users::numeric / cohort_size::NUMERIC * 100, 1) AS retention_rate
FROM cohorts
WHERE cohort_size >= 5


Not that big of a deal for me - manageable task.

---

## Task 3: NTILE Segmentation — Transaction Amount Quartiles

**Scenario:**
The analytics team wants to divide all transactions into 4 equal buckets by amount, then report the min, max, and average amount per bucket — to understand how transaction values are distributed.

**Expected Output Columns:**
- `quartile` (integer) — 1 to 4 (1 = lowest amounts)
- `min_amount` (numeric) — minimum amount in this quartile, rounded to 2 decimals
- `max_amount` (numeric) — maximum amount in this quartile, rounded to 2 decimals
- `avg_amount` (numeric) — average amount in this quartile, rounded to 2 decimals
- `transaction_count` (integer)

**Requirements:**
- Use `transactions`, exclude NULL amounts
- NTILE(4) ordered by amount ASC (so quartile 1 = lowest)
- Order by `quartile ASC`

**Difficulty Rating:** 3/5

WITH transactions_quartiles AS (
SELECT 
	*,
	NTILE(4) OVER (ORDER BY t.amount) AS quartile
FROM crappy_data_db.transactions t
)
SELECT 
	quartile,
	ROUND(MIN(amount), 2) AS min_amount,
	ROUND(MAX(amount), 2) AS max_amount,
	ROUND(AVG(amount), 2) AS avg_amount,
	COUNT(*) AS transaction_count
FROM transactions_quartiles
GROUP BY quartile
ORDER BY quartile

This is just too easy.

---

## Submission Instructions

1. Task 1 — YoY revenue comparison per country with NULLIF pct_change guard (3/5)
2. Task 2 — Cohort retention: % of users who ordered in month after registration (5/5)
3. Task 3 — NTILE(4) transaction quartiles with min/max/avg per bucket (3/5)
