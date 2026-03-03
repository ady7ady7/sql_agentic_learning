# Daily SQL Practice Tasks

**Generated:** 2026-03-03
**Week 12, Day 2 Focus:** HackerRank Hard — Full Difficulty

---

## Task 1: 3-Level Hierarchy — Orders by Delivery Status and Top Users

**Scenario:**
Build a 3-level hierarchy over order/delivery data:
- Level 1: `'All Orders'`
- Level 2: Distinct delivery statuses (dynamic, from `deliveries`)
- Level 3: For each status, the 3 users with the highest total order amount under that delivery status (show user_id as text)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — status at Level 2, user_id at Level 3
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Join `deliveries → orders` to get user_id and amount per delivery status
- Pre-aggregate totals per user+status, then rank before the recursive CTE
- Termination condition required

**Difficulty Rating:** 4/5

WITH RECURSIVE distinct_statuses AS (
SELECT DISTINCT status FROM deliveries
),
users_delivery_statuses AS (
SELECT 
	o.user_id,
	d.status,
	SUM(o.amount) AS total_amount
FROM deliveries d
JOIN orders o ON d.order_id = o.id
GROUP BY o.user_id, d.status
ORDER BY o.user_id
),
users_statuses_rank AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY status ORDER BY total_amount DESC) AS status_rank
FROM users_delivery_statuses
),
top_three_orders_per_status AS (
SELECT 
	* 
FROM users_statuses_rank
WHERE status_rank <= 3
),
HIERARCHY AS (
SELECT
	1 AS level,
	'All Orders' AS name,
	NULL::TEXT AS parent_name,
	'All Orders' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(ds.status, tto.user_id::TEXT),
	h.name,
	h.PATH || ' < ' || COALESCE(ds.status, tto.user_id::TEXT)
FROM HIERARCHY h
LEFT JOIN distinct_statuses ds ON h.LEVEL = 1
LEFT JOIN top_three_orders_per_status tto ON h.LEVEL = 2 AND tto.status = h.name
WHERE h.LEVEL < 3
)
SELECT * FROM hierarchy


---

## Task 2: First and Repeat Purchaser Revenue Split

**Scenario:**
The growth team wants to understand how much revenue comes from first-time buyers vs repeat buyers each month.

For each month, classify each order as either a user's first-ever order (`first_purchase`) or a subsequent one (`repeat_purchase`), then aggregate revenue by month and purchase type.

**Expected Output Columns:**
- `month` (date) — truncated to month
- `purchase_type` (text) — `'first_purchase'` or `'repeat_purchase'`
- `order_count` (bigint)
- `total_revenue` (numeric) — rounded to 2 decimals
- `pct_of_monthly_revenue` (numeric) — this type's revenue as % of total revenue that month, rounded to 1 decimal

**Requirements:**
- Use `orders` table only
- Identify each user's first order using `MIN(created_at)` or `ROW_NUMBER()`
- `pct_of_monthly_revenue` requires a window SUM over the month partition
- Order by `month ASC`, `purchase_type ASC`

**Difficulty Rating:** 5/5

WITH orders_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
FROM orders
),
users_order_types AS (
SELECT 
	*,
	FIRST_VALUE(created_at) OVER (PARTITION BY month_, user_id ORDER BY created_at) AS first_order_time,
	CASE WHEN FIRST_VALUE(created_at) OVER (PARTITION BY month_, user_id ORDER BY created_at) = created_at THEN 'first_purchase' ELSE 'repeat_purchase' END AS purchase_type
FROM orders_months
),
users_order_types_total_revenue AS (
SELECT 
	*,
	SUM(amount) OVER (PARTITION BY month_, user_id, purchase_type ORDER BY created_at) AS total_revenue
FROM users_order_types
),
monthly_users_repeat_purchases_revenues AS (
SELECT 
	user_id,
	month_,
	COUNT(*) AS order_count,
	MAX(total_revenue) AS repeat_purchase_revenue
FROM users_order_types_total_revenue
GROUP BY user_id, month_
)
SELECT 
	uot.month_,
	uot.user_id,
	uot.total_revenue AS first_purchase_revenue,
	mur.repeat_purchase_revenue AS total_revenue,
	mur.order_count,
	CASE WHEN uot.total_revenue = mur.repeat_purchase_revenue THEN 100 ELSE ROUND((uot.total_revenue / (mur.repeat_purchase_revenue - uot.total_revenue))::NUMERIC * 100, 1) END AS first_purchases_pct_of_monthly_revenue
FROM monthly_users_repeat_purchases_revenues mur
JOIN users_order_types_total_revenue uot ON mur.month_ = uot.month_ AND mur.user_id = uot.user_id AND uot.purchase_type = 'first_purchase'
ORDER BY uot.user_id


So it works perfectly fine and the logic is sound, but there are some differences, and data is shown in a bit different way, but it's still clear.

First of all, I don't have this division on first_purchase/repeat_purchase, but rather I calculated it on my own and divided the first_purchase_revenue from total_revenue. I run proper checks to subtract first_purchase_revenue for percent calculations in case it's different (because some months and users only had one purchase for a given month etc.).

This logic works, and I've named the rows properly to make it all sound and clear.
The differences have risen because I DIDN'T FOLLOW your instructions step by step, but rather ran my own thinking and reasoning process based on the base task instruction. I don't think it's bad as I wanted to stimulate thinking on my own. As a DA I will most likely not have an AI supervisor that will tell me how to do everything step by step, so that's my reasoning.

---

## Task 3: Session Engagement Deciles

**Scenario:**
The analytics team wants to segment users into 10 equal engagement buckets (deciles) based on their total session count across all time.

For each decile, show the number of users, the min/max/avg total sessions in that decile, and what percentage of all sessions that decile accounts for.

**Expected Output Columns:**
- `decile` (integer) — 1 (lowest) to 10 (highest)
- `user_count` (bigint)
- `min_sessions` (bigint)
- `max_sessions` (bigint)
- `avg_sessions` (numeric) — rounded to 1 decimal
- `pct_of_total_sessions` (numeric) — rounded to 1 decimal

**Requirements:**
- Use `user_sessions_daily`
- Aggregate total sessions per user first, then apply `NTILE(10)`
- `pct_of_total_sessions` = decile's total sessions / grand total sessions * 100
- Order by `decile ASC`

**Difficulty Rating:** 4/5

WITH user_session_cnt AS (
SELECT 
	user_id,
	SUM(count_sessions) AS total_sessions
FROM user_sessions_daily
GROUP BY user_id
),
user_session_deciles AS (
SELECT 
	*,
	NTILE(10) OVER (ORDER BY total_sessions) AS decile
FROM user_session_cnt
),
deciles_metrics AS (
SELECT 
	decile,
	COUNT(*) AS user_count,
	sum(total_sessions) AS total_sessions,
	MIN(total_sessions) AS min_sessions,
	MAX(total_sessions) AS max_sessions,
	AVG(total_sessions) AS avg_sessions,
	(SELECT SUM(count_sessions) FROM user_sessions_daily) AS sessions_grand_total
FROM user_session_deciles
GROUP BY decile
)
SELECT 
	*,
	decile,
	user_count,
	min_sessions,
	max_sessions,
	avg_sessions,
	ROUND(total_sessions / sessions_grand_total * 100, 1) AS pct_of_total_sessions
FROM deciles_metrics
ORDER BY decile

This task wasn't a problem for me, but I also enjoyed it as it required multi steps approach and some potential traps that I managed to avoid.

---

## Submission Instructions

1. Task 1 — Delivery status hierarchy with top users (4/5)
2. Task 2 — First vs repeat purchaser revenue split (5/5)
3. Task 3 — Session engagement deciles (4/5)
