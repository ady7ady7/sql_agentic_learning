# Daily SQL Practice Tasks

**Generated:** 2026-02-20
**Week 10, Day 5 Focus:** HackerRank Hard Final Puzzles — Multi-Pattern Combinations

---

## Task 1: 3-Level Hierarchy — Order Status by User Segment

**Scenario:**
Build a 3-level hierarchy combining orders and deliveries:
- Level 1: `'All Orders'`
- Level 2: Distinct delivery statuses from the `deliveries` table (dynamic, no hardcoded values)
- Level 3: For each status, the 3 users with the most orders in that delivery status (show user_id as text)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — status at Level 2, user_id at Level 3
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Pre-aggregate distinct statuses and top-3-users-per-status before the recursive CTE
- Join `deliveries` → `orders` to get user_id per delivery
- Use order count to rank users within each status
- Termination condition required

**Difficulty Rating:** 4/5


WITH RECURSIVE order_statuses_cnt AS (
SELECT 
	o.user_id,
	d.status,
	COUNT(o.id) AS order_cnt
FROM orders o
JOIN deliveries d ON o.id = d.order_id
GROUP BY o.user_id, d.status 
ORDER BY order_cnt DESC
),
ranked_statuses AS (
SELECT 
	*,
	ROW_NUMBER() OVER (PARTITION BY status ORDER BY order_cnt DESC) AS order_rank
FROM order_statuses_cnt
),
top_three_users_per_order_status AS (
SELECT * 
FROM ranked_statuses
WHERE order_rank <= 3
),
distinct_delivery_statuses AS (
SELECT DISTINCT status FROM deliveries
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Orders' AS name,
	NULL::text AS parent_name,
	'All Orders' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(dds.status::TEXT, ttu.user_id::TEXT),
	h.name,
	h.PATH || ' > ' || COALESCE(dds.status::TEXT, ttu.user_id::TEXT)
FROM HIERARCHY h
LEFT JOIN distinct_delivery_statuses dds ON h.LEVEL = 1
LEFT JOIN top_three_users_per_order_status ttu ON h.LEVEL = 2 AND h.name = ttu.status
WHERE h.LEVEL < 3
)
SELECT * 
FROM hierarchy



---

## Task 2: Order Gap Analysis per User (Gaps Pattern on Orders)

**Scenario:**
The retention team wants to understand ordering cadence. For each user who has placed at least 2 orders, calculate the average and maximum number of days between consecutive orders.

Then classify users into cadence segments:
- `frequent`: avg_days_between_orders < 30
- `regular`: avg_days_between_orders between 30 and 90 (inclusive)
- `infrequent`: avg_days_between_orders > 90

**Expected Output Columns:**
- `user_id` (integer)
- `order_count` (bigint)
- `avg_days_between_orders` (numeric) — rounded to 1 decimal
- `max_days_between_orders` (integer)
- `cadence_segment` (text)

**Requirements:**
- Use `orders` table
- Use LAG to compute days between consecutive orders per user
- Exclude users with only 1 order (no gap to compute)
- Order by `avg_days_between_orders ASC`

**Difficulty Rating:** 4/5


WITH users_orders_cnt AS (
SELECT 
	user_id,
	COUNT(id) AS orders_cnt
FROM orders o
GROUP BY user_id
HAVING count(id) > 1
),
users_order_dates AS (
SELECT
	uoc.user_id,
	uoc.orders_cnt,
	DATE_TRUNC('Day', o.created_at) AS order_date,
	LAG(DATE_TRUNC('Day', o.created_at)) OVER (PARTITION BY o.user_id ORDER BY o.created_at) AS previous_order_date
FROM users_orders_cnt uoc
JOIN orders o ON uoc.user_id = o.user_id
),
users_day_gaps AS (
SELECT 
	user_id,
	orders_cnt,
	order_date,
	previous_order_date,
	EXTRACT('Day' FROM order_date - previous_order_date) AS days_gap
FROM users_order_dates
WHERE previous_order_date IS NOT NULL
),
users_avg_gaps AS (
SELECT 
	*,
	AVG(days_gap) OVER (PARTITION BY user_id) AS average_user_gap
FROM users_day_gaps
),
users_orders_metrics AS (
SELECT 
	user_id,
	orders_cnt,
	ROUND(average_user_gap, 1) AS avg_days_between_orders,
	MAX(days_gap) AS max_days_between_orders
FROM users_avg_gaps
GROUP BY user_id, orders_cnt, average_user_gap
)
SELECT 
	*,
	CASE
		WHEN avg_days_between_orders < 3 THEN 'frequent' 
		WHEN avg_days_between_orders >= 3 AND avg_days_between_orders <= 6 THEN 'regular' 
		WHEN avg_days_between_orders > 6 THEN 'infrequent' 
	END AS cadence_segment
FROM users_orders_metrics
ORDER BY avg_days_between_orders ASC
	

Here, again I've adjusted the cadence segments to match the reality of our data, as there was literally not a single user with an average gap above 20, so it wouldn't make any sense. As for the rest, I've adjusted everything to your needs - eliminated unnecessary users early in the query, then calculated gaps and assigned them to users based on window funcs and finally aggregated everything nicely with a final CASE WHEN statement to create the segments, with numbers matching the reality of our data.

---

## Task 3: The Friday Challenge — Power User Leaderboard

**Scenario:**
The growth team wants a comprehensive power user leaderboard combining session activity, order behavior, and transaction volume.

For each user, calculate:
- Total sessions across all days (`user_sessions_daily`)
- Total orders placed (`orders`)
- Total transaction amount (`transactions`)
- A composite score: `(total_sessions * 0.3) + (total_orders * 10) + (total_transaction_amount * 0.01)`, rounded to 2 decimals
- Their overall rank by composite score (highest first)
- Their percentile (using `PERCENT_RANK()`), rounded to 1 decimal, shown as a value between 0 and 100

Only include users who appear in **all three** tables (sessions, orders, transactions).

**Expected Output Columns:**
- `user_id` (integer)
- `total_sessions` (bigint)
- `total_orders` (bigint)
- `total_transaction_amount` (numeric) — rounded to 2 decimals
- `composite_score` (numeric)
- `rank` (bigint)
- `percentile` (numeric)

**Requirements:**
- Use `user_sessions_daily`, `orders`, `transactions`
- Aggregate each source separately in CTEs before joining
- Use INNER JOINs to enforce presence in all three tables
- Order by `rank ASC`

**Difficulty Rating:** 5/5

WITH users_session_cnt AS (
SELECT 
	user_id,
	SUM(count_sessions) AS total_sessions
FROM user_sessions_daily
GROUP BY user_id
),
users_orders_cnt AS (
SELECT 
	user_id,
	COUNT(*) AS total_orders
FROM orders
GROUP BY user_id
),
users_total_transaction_amts AS (
SELECT 
	user_id,
	SUM(amount) AS total_transaction_amount
FROM transactions t
GROUP BY user_id
),
users_combined_statistics AS (
SELECT 
	u.id AS user_id,
	COALESCE(usc.total_sessions, 0) AS total_sessions,
	COALESCE(uoc.total_orders, 0) AS total_orders,
	COALESCE(utt.total_transaction_amount, 0) AS total_transaction_amount,
	ROUND((usc.total_sessions * 0.3) + (uoc.total_orders * 10) + (utt.total_transaction_amount * 0.01), 2) AS composite_score
FROM users u
JOIN users_session_cnt usc ON u.id = usc.user_id
JOIN users_orders_cnt uoc ON u.id = uoc.user_id
JOIN users_total_transaction_amts utt ON u.id = utt.user_id
ORDER BY user_id
)
SELECT 
	*,
	RANK() OVER (ORDER BY composite_score DESC) AS rank,
	ROUND((PERCENT_RANK() OVER (ORDER BY composite_score)::NUMERIC * 100), 1)  AS percentile
FROM users_combined_statistics
ORDER BY RANK


What a cool task.
I instinctively started from LEFT JOINs and COALESCE to join all tables in the pre-final CTE and replaced NULLs with zeros, as I thought that would be the approach. I'm glad I've finally checked your requirements, especialyl that there were some other caveats - like the format of our percentile, which is kinda unusual, but also a great way to practice.



---

## Submission Instructions

1. Task 1 — Order status hierarchy (4/5)
2. Task 2 — Order gap analysis (4/5)
3. Task 3 — Power user leaderboard (5/5)
