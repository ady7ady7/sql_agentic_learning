# Daily SQL Practice Tasks

**Generated:** 2026-02-16
**Week 10, Day 1 Focus:** Dynamic Hierarchies + HackerRank Hard Puzzles

---

## Task 1: 3-Level Hierarchy — Transaction Types from Real Data

**Scenario:**
Build a 3-level hierarchy where Level 2 is pulled dynamically from real data:
- Level 1: 'All Transactions'
- Level 2: Distinct transaction types from the `transactions` table (no hardcoded VALUES)
- Level 3: Top 3 transactions per type by amount DESC (show transaction ID as text)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — type name or transaction ID
- `parent_name` (text)
- `path` (text) — full path from root

**Requirements:**
- Pre-aggregate distinct types and top-3-per-type before the recursive CTE
- Carry the type name through Level 2 to use as the join key at Level 3
- Include path column
- Do not forget the termination condition

**Difficulty Rating:** 4/5

WITH RECURSIVE transactions_rank AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY TYPE ORDER BY amount DESC) AS transaction_rank
FROM transactions
),
top_three AS (
SELECT * FROM transactions_rank
WHERE transaction_rank <= 3
),
distinct_types AS (
SELECT DISTINCT TYPE FROM top_three
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Transactions' AS name,
	NULL::TEXT AS parent_name,
	'All Transactions' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(dt.TYPE::TEXT, tt.id::TEXT),
	h.name,
	h.PATH || ' > ' || COALESCE(dt.TYPE, tt.id::TEXT)
FROM HIERARCHY h
LEFT JOIN distinct_types dt ON h.LEVEL = 1
LEFT JOIN top_three tt ON h.name = tt."type" AND h.LEVEL = 2
WHERE h.LEVEL < 3
)
SELECT * FROM hierarchy


Finally got it, but it was difficult!


---

## Task 2: Consecutive Month Buyers

**Scenario:**
The retention team wants to identify users with strong purchasing streaks. Find users who placed orders in at least 3 consecutive calendar months. For each qualifying user, show their longest streak.

**Expected Output Columns:**
- `user_id` (integer)
- `streak_start` (date) — first month of the longest streak (truncated to month)
- `streak_end` (date) — last month of the longest streak
- `streak_length` (bigint) — number of consecutive months in the streak
- `streak_revenue` (numeric) — total order revenue during the streak, rounded to 2 decimals

**Requirements:**
- Use `orders` table
- A streak is a sequence of months with no gaps (each month follows the previous by exactly 1 month)
- Only show users whose longest streak is 3+ months
- If a user has multiple streaks of equal length, show the most recent one
- Order by streak_length DESC, streak_revenue DESC

**Difficulty Rating:** 5/5

WITH users_orders_months AS (
SELECT 
	*,
	date_trunc('Month', created_at) AS month_
FROM orders
),
users_order_cnt AS (
SELECT 
	user_id,
	month_,
	COUNT(id) AS order_cnt
FROM users_orders_months
GROUP BY user_id, month_
ORDER BY user_id, month_
),
prev_month_check AS (
SELECT 
	*,
	month_ - INTERVAL '1' MONTH AS valid_month,
	LAG(month_) OVER (PARTITION BY user_id) AS prev_month
FROM users_order_cnt
)
SELECT
	*,
	CASE WHEN valid_month = prev_month THEN 1 ELSE 0 END AS valid_streak_
FROM prev_month_check 
WHERE prev_month IS NOT NULL


I STOPPED HERE - BECAUSE THERE ARE NO SUCH STREAKS.
At this point I was able to visually scan data and there were no users with streaks MORE THAN 2 MONTHS - literally nobody. That would be the time I'd report it to the business. Anyway, I didn't feel like I'm making the smoothest approach here, but I might be wrong!

---

## Task 3: User Cohort Activation Funnel

**Scenario:**
The growth team wants to understand how quickly new users make their first purchase after registering. Group users by their registration month (cohort), then classify them based on time to firstc order.

**Expected Output Columns:**
- `cohort_month` (date) — registration month, truncated to month
- `total_users` (bigint) — users registered in that cohort
- `activated_within_30d` (bigint) — users who placed their first order within 30 days of registration
- `activated_31_to_90d` (bigint) — first order between 31 and 90 days after registration
- `never_activated` (bigint) — users who never placed any order
- `activation_rate_pct` (numeric) — % of cohort who activated within 90 days, rounded to 1 decimal

**Requirements:**
- Use `users` and `orders` tables
- First order = earliest order by created_at per user
- Users with no orders count as never_activated
- Order by cohort_month ASC

**Difficulty Rating:** 5/5


WITH cohort_months AS (
SELECT 
DATE_TRUNC('Month', created_at) AS cohort_month,
COUNT(id) AS total_users
FROM users u
GROUP BY DATE_TRUNC('Month', created_at)
ORDER BY cohort_month
),
users_first_orders AS (
SELECT 
	DISTINCT
	o.user_id,
	u.created_at AS acc_creation_time,
	FIRST_VALUE(o.created_at) OVER (PARTITION BY o.user_id) AS first_order_time
FROM users u
JOIN orders o ON u.id = o.user_id
),
users_activation_time AS (
SELECT 
	u.id AS user_id,
	u.created_at,
	ufo.first_order_time,
	COALESCE(EXTRACT('Day' FROM ufo.first_order_time - ufo.acc_creation_time), 0) AS activation_days
FROM users u LEFT JOIN users_first_orders ufo ON u.id = ufo.user_id
),
users_cohorts_check AS (
SELECT
	date_trunc('Month', created_at) AS creation_month,
	CASE WHEN activation_days > 20 THEN 1 ELSE 0 END AS activated_after_20d,
	CASE WHEN activation_days >= 0 AND activation_days <= 20 AND firsT_order_time IS NOT NULL THEN 1 ELSE 0 END AS activated_within_20d,
	CASE WHEN activation_days = 0 THEN 1 ELSE 0 END AS never_activated
FROM users_activation_time
),
grouped_monthly_cohorts AS (
SELECT 
creation_month,
SUM(activated_after_20d) AS activated_after_20d,
SUM(activated_within_20d) AS activated_within_20d,
SUM(never_activated) AS never_activated
FROM users_cohorts_check
GROUP BY creation_month
)
SELECT 
	cm.cohort_month,
	cm.total_users,
	gmc.activated_after_20d,
	gmc.activated_within_20d,
	gmc.never_activated,
	ROUND(((gmc.activated_after_20d + gmc.activated_within_20d)::NUMERIC / cm.total_users::NUMERIC) * 100, 1) || '%' AS activation_rate_pct
FROM grouped_monthly_cohorts gmc 
JOIN cohort_months cm ON gmc.creation_month = cm.cohort_month
ORDER BY cohort_month

I must admit that this was A REALLY DEMANDING and time consuming task. I had to stop for a few times and get back e.g. to include users that didn't make any orders, to make 100% sure that users who made the order the very same day that they registered will not be taken for users without any orders etc.

But I'm sure I've done it properly in the end as I've verified and checked data in between the steps. Pretty satisfying.

---

## Submission Instructions

1. Task 1 — Dynamic hierarchy from transactions (4/5)
2. Task 2 — Consecutive month buyer streaks (5/5)
3. Task 3 — Cohort activation funnel (5/5)
