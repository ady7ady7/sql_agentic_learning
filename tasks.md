# Daily SQL Practice Tasks

**Generated:** 2026-02-17
**Week 10, Day 2 Focus:** Gaps-and-Islands, Session Engagement Analysis, Product Category Ranking

---

## Task 1: 3-Level Hierarchy — Delivery Statuses by Month

**Scenario:**
Build a 3-level hierarchy over delivery data:
- Level 1: `'All Deliveries'`
- Level 2: Distinct delivery statuses from the `deliveries` table (pulled dynamically, no hardcoded VALUES)
- Level 3: The 3 most recent deliveries per status (show delivery ID as text)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — status name at Level 2, delivery ID at Level 3
- `parent_name` (text)
- `path` (text) — full path from root

**Requirements:**
- Pre-aggregate distinct statuses and top-3-per-status before the recursive CTE
- Use `created_at DESC` to define "most recent"
- Do not forget the termination condition
- Do not hardcode status values

**Difficulty Rating:** 3/5

WITH RECURSIVE delivery_ranks AS (
SELECT 
	*,
	DENSE_RANK() OVER (PARTITION BY status ORDER BY created_at DESC)
FROM deliveries
),
three_recent_deliveries_by_status AS (
SELECT 
	*
FROM delivery_ranks
WHERE DENSE_RANK <= 3
),
distinct_statuses AS (
SELECT DISTINCT status FROM deliveries
),
HIERARCHY AS (
SELECT
	1 AS level,
	'All Deliveries' AS name,
	NULL::TEXT AS parent_name,
	'All Deliveries' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(ds.status, trd.id::TEXT),
	h.name,
	h.PATH || ' > ' || COALESCE(ds.status, trd.id::TEXT)
FROM HIERARCHY h
LEFT JOIN distinct_statuses ds ON h.LEVEL = 1
LEFT JOIN three_recent_deliveries_by_status trd ON trd.status = h.name
WHERE h.LEVEL < 3
)
SELECT * FROM HIERARCHY

---

## Task 2: User Session Streaks (Gaps-and-Islands)

**Scenario:**
The engagement team wants to identify "power users" — users with long streaks of consecutive days where they had at least one session (`count_sessions > 0`).

Find users whose longest active streak is at least 5 consecutive days. For each qualifying user, show their longest streak only.

**Expected Output Columns:**
- `user_id` (integer)
- `streak_start` (date)
- `streak_end` (date)
- `streak_length` (bigint) — number of consecutive active days
- `avg_daily_sessions` (numeric) — average `count_sessions` during the streak, rounded to 2 decimals

**Requirements:**
- Use `user_sessions_daily` table
- Active day = `count_sessions > 0`
- A streak is a sequence of consecutive calendar days with no gaps
- If a user has multiple streaks of equal max length, show the most recent one
- Order by `streak_length DESC`, `avg_daily_sessions DESC`

**Hint:** The gaps-and-islands pattern — subtract `ROW_NUMBER()` from the date to create a streak group key. Dates within the same streak will share the same group key.

**Difficulty Rating:** 5/5


WITH users_dates AS (
SELECT 
	user_id,
	date,
	LAG(date) OVER (PARTITION BY user_id ORDER BY date) AS prev_session_date,
	count_sessions
FROM user_sessions_daily usd
ORDER BY user_id
),
users_dates_rn AS (
SELECT 
	*,
	ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY date) AS rn
FROM users_dates
WHERE prev_session_date IS NOT NULL
),
users_streak_keys AS (
SELECT 
	*,
	date - rn * INTERVAL '1' DAY AS streak_key
FROM users_dates_rn
)
SELECT 
	user_id,
	streak_key,
	MIN(date) AS streak_start,
	MAX(date) AS streak_end,
	COUNT(*) AS streak_length,
	ROUND(AVG(count_sessions), 2) AS avg_daily_sessions
FROM users_streak_keys
GROUP BY user_id, streak_key
HAVING COUNT(*) >= 5
ORDER BY streak_length DESC, avg_daily_sessions DESC


Completed after a long struggle, but I definitely don't feel strong with this pattern AND I'M FEELING like it looks a bit weird e.g. streak_keys differ from the actual streak_starts and I wouldn't trust this with 100% trust score.

---

## Task 3: Category Revenue Ranking with Rolling Comparison

**Scenario:**
The product team wants a monthly revenue leaderboard by product category.

For each month and category, calculate:
- Total revenue from that category in that month (`quantity × price`)
- The category's rank that month (by revenue, highest first)
- Revenue from the same category in the previous month (LAG)
- Month-over-month revenue change (current − previous), NULL if no previous month

Show only months and categories where total revenue > 0.

**Expected Output Columns:**
- `month` (date) — truncated to month
- `category_name` (text)
- `monthly_revenue` (numeric) — total revenue, rounded to 2 decimals
- `revenue_rank` (bigint) — rank within that month (1 = highest revenue)
- `prev_month_revenue` (numeric) — previous month's revenue for same category, rounded to 2 decimals, NULL if none
- `mom_change` (numeric) — month-over-month change, rounded to 2 decimals, NULL if no previous month

**Requirements:**
- Use `orders`, `orders_products`, `products`, `product_categories`
- Revenue = `orders_products.quantity × products.price`
- Order by `month ASC`, `revenue_rank ASC`

**Difficulty Rating:** 4/5


WITH categories_monthly_revenues AS (
SELECT 
	DATE_TRUNC('Month', o.created_at) AS month_,
	pc."name" AS category_name,
	SUM(p.price * op.quantity) AS monthly_revenue
FROM orders_products op
JOIN orders o ON op.order_id = o.id
JOIN products p ON op.product_id = p.id
JOIN product_categories pc ON p.category_id = pc.id
GROUP BY DATE_TRUNC('Month', o.created_at), pc.name
)
SELECT 
	month_,
	category_name,
	monthly_revenue,
	RANK() OVER (PARTITION BY month_ ORDER BY monthly_revenue DESC) AS revenue_rank,
	COALESCE(LAG(monthly_revenue) OVER (PARTITION BY category_name ORDER BY month_), NULL) AS prev_month_revenue,
	monthly_revenue - COALESCE(LAG(monthly_revenue) OVER (PARTITION BY category_name ORDER BY month_), NULL) AS mom_change
FROM categories_monthly_revenues


Your requirements were met in this case and the order also matches as I've checked everything. I could add one more CTE for more clarity, but I decieded to make it the most efficient instead and did one-liner with mom_change.

It all works and definitely satisfies all the requirements

---

## Submission Instructions

1. Task 1 — 3-level delivery hierarchy (3/5)
2. Task 2 — User session streaks (5/5)
3. Task 3 — Category revenue ranking with rolling comparison (4/5)
