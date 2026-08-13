# SQL Tasks — 2026-08-13 (Week 34, Day 4)

**Dataset:** user_sessions_daily / orders / users / transactions  
**Focus:** Gaps-and-islands (monthly) · Funnel with conversion time · STDDEV z-score

---

## Task 1 — Longest Monthly Activity Streak
**Difficulty: 4/5**

**Business question:**  
Using `user_sessions_daily`, find the longest consecutive-month streak for each user — where a "streak" is an unbroken sequence of months in which the user had at least one session (`count_sessions > 0`).

Show each user's longest streak length (in months). Only include users with a streak of at least 2 months.

**Expected output columns:**  
`user_id, longest_streak`

Order by `longest_streak DESC`, `user_id`.

**Hint:** The gaps-and-islands trick works differently at month granularity — you can't subtract row numbers from dates directly. Instead: assign a month number to each active month per user, then subtract a `ROW_NUMBER()` from it. Consecutive months will have the same difference.



WITH sessions_months AS (
SELECT
	*,
	DATE_TRUNC('Month', date) AS month
FROM crappy_data_db.user_sessions_daily usd
),
users_sessions_months AS (
SELECT 
	USER_ID,
	MONTH,
	SUM(count_sessions)
FROM sessions_months
GROUP BY user_id, MONTH
ORDER BY user_id
),
users_pm AS (
SELECT 
	*,
	LAG(month) OVER (PARTITION BY USER_ID ORDER BY MONTH) AS prev_month
FROM users_sessions_months
),
users_streak_keys AS (
SELECT 
	*,
	CASE WHEN EXTRACT(EPOCH FROM MONTH - prev_month) / 86400 < 32 THEN 0 ELSE 1 END AS streak_key
FROM users_pm
WHERE prev_month IS NOT NULL
),
users_streak_ids AS (
SELECT 
	*,
	sum(streak_key) OVER (PARTITION BY user_id ORDER BY month) AS streak_id
FROM users_streak_keys
),
users_streaks AS (
SELECT
	user_id,
	streak_id,
	MIN(month) AS streak_start,
	MAX(month) AS streak_end,
	COUNT(*) AS streak_length
FROM users_streak_ids
GROUP BY user_id, streak_id
)
SELECT 
	user_id,
	MAX(streak_length) AS longest_streak
FROM users_streaks
GROUP BY user_id
ORDER BY longest_streak DESC


Rather a long CTE query, but I've managed to do it my way and IMO it works perfectly fine





**Difficulty: 4/5**

---

## Task 2 — Funnel with Time to First Order
**Difficulty: 4/5**

**Business question:**  
Build a monthly registration cohort funnel. For each cohort (month of `users.created_at`), show:
- Total users registered that month (`cohort_size`)
- How many placed at least one order ever (`buyers`)
- Conversion rate: buyers / cohort_size
- Average days from registration to first order (only for users who ordered)

**Expected output columns:**  
`cohort_month, cohort_size, buyers, conversion_pct, avg_days_to_first_order`

Order by `cohort_month`.

**Difficulty: 4/5**

It was pretty long as well but I think I've handled it smoothly

WITH users_orders AS (
SELECT 
	*,
	DATE_TRUNC('Month', u.created_at) AS cohort_month
FROM crappy_data_db.users u
),
users_register_cohorts AS (
SELECT 
	cohort_month,
	COUNT(*) AS cohort_size
FROM users_orders
GROUP BY cohort_month
),
cohorts_orders_cnts AS (
SELECT 
	uo.id AS user_id,
	uo.created_at AS registration_time,
	cohort_month,
	MIN(o.created_at) AS first_order,
	COALESCE(COUNT(o.created_at), 0) AS orders_cnt
FROM users_orders uo
LEFT JOIN crappy_data_db.orders o ON uo.id = o.user_id 
GROUP BY uo.id, uo.created_at, cohort_month
),
cohorts_days_to_first_order AS (
SELECT 
	*,
	EXTRACT(EPOCH FROM first_order - coc.registration_time) / 86400 AS days_to_first_order
FROM cohorts_orders_cnts coc
)
SELECT 
	c.cohort_month,
	u.cohort_size,
	COUNT(*) FILTER (WHERE first_order IS NOT NULL) AS buyers,
	ROUND(COUNT(*) FILTER (WHERE first_order IS NOT NULL) / u.cohort_size::NUMERIC * 100, 2) AS conversion_pct,
	ROUND(AVG(days_to_first_order), 2) AS avg_days_to_first_order
FROM cohorts_days_to_first_order c
JOIN users_register_cohorts u ON c.cohort_month = u.cohort_month
GROUP BY c.cohort_month, u.cohort_size
ORDER BY COHORT_MONTH


---

## Task 3 — Spending Outliers (Z-Score Corrected)
**Difficulty: 3/5**

**Business question:**  
Find users whose total transaction amount is more than 2 standard deviations above the mean across all users.

Show each outlier's `user_id`, their total spending, and how many standard deviations above the mean they are (z-score).

Z-score formula: `(total_amount - mean) / stddev`

**Expected output columns:**  
`user_id, total_amount, z_score`

Order by `z_score DESC`.

**Difficulty: 3/5**

WITH USERS_spendings AS (
SELECT 
	user_id,
	SUM(amount) AS total_amount
FROM crappy_data_db.transactions t
GROUP BY user_id
),
mean_avg AS (
SELECT 
	round(STDDEV(total_amount), 2) AS STDEV,
	round(AVG(total_amount), 2) AS mean
FROM users_spendings
)
SELECT 
	u.user_id,
	u.total_amount,
	ROUND((u.total_amount - m.mean) / m.stdev, 2) AS z_score
FROM users_spendings U
CROSS JOIN mean_avg m
ORDER BY z_score DESC




---

## Submission Instructions

Paste your queries below each task.