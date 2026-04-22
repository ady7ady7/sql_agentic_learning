# Daily SQL Practice Tasks

**Generated:** 2026-04-22
**Week 19, Day 3 Focus:** Type A fixed-hierarchy UNION ALL (no recursion) + PERCENT_RANK + gaps-and-islands

---

## Task 1: Type A Fixed-Hierarchy Rollup — Spend by Country → User

**Scenario:**
The finance team wants a 2-level breakdown of total order spend:
- **Level 1:** Total spend per country
- **Level 2:** Total spend per user (within their country)

Use the **Type A fixed-hierarchy CTE** pattern — meaning: two independent aggregation CTEs, then a plain `UNION ALL`. No `WITH RECURSIVE`, no self-joins.

**Expected Output Columns:**
- `level` (integer — 1 or 2)
- `label` (text — country name for level 1, `'User #' || user_id` for level 2)
- `total_spend` (numeric, rounded to 2 decimals)

**Order by:** `level ASC`, `total_spend DESC`

**Tables:** `orders`, `users`

**Note:** Exclude NULL amounts. For level 2, only include users with at least 1 order.

**Difficulty Rating:** 3/5


WITH RECURSIVE countries_revenues AS (
SELECT 
	u.country,
	SUM(o.amount) AS country_revenue
FROM crappy_data_db.users u
JOIN crappy_data_db.orders o ON u.id = o.user_id
WHERE u.country IS NOT NULL
GROUP BY u.country
),
users_revenues AS (
SELECT 
	o.user_id,
	u.country,
	SUM(o.amount) AS user_revenue
FROM crappy_data_db.users u
JOIN crappy_data_db.orders o ON u.id = o.user_id
WHERE u.country IS NOT NULL
GROUP BY o.user_id, u.country
),
hierarchy AS (
SELECT 
	1 AS LEVEL,
	COUNTRY::TEXT AS LABEL,
	country_revenue AS total_spend
FROM countries_revenues
UNION ALL
SELECT
	h.LEVEL + 1,
	ur.user_id::TEXT,
	ur.user_revenue
FROM hierarchy h
LEFT JOIN users_revenues ur ON h.label = ur.country
WHERE h.LEVEL < 2
)
SELECT * FROM HIERARCHY
ORDER BY LEVEL, total_spend DESC



---

## Task 2: PERCENT_RANK — Order Value Percentile Bands

**Scenario:**
The analytics team wants to understand how each order ranks within its user's order history by value.

For every order, show:
- `order_id`
- `user_id`
- `amount`
- `percentile` — `PERCENT_RANK()` of this order within the user's orders by amount (0.0 to 1.0), rounded to 2 decimals
- `band` — label based on percentile:
  - `'top 10%'` if percentile >= 0.9
  - `'top 25%'` if percentile >= 0.75
  - `'mid'` if percentile >= 0.25
  - `'bottom 25%'` otherwise

**Tables:** `orders`

**Requirements:**
- Exclude NULL amounts
- Order by `user_id ASC`, `amount DESC`

**Difficulty Rating:** 4/5

WITH users_orders_rank AS (
SELECT 
	id AS order_id,
	user_id,
	amount,
	PERCENT_RANK() OVER (PARTITION BY user_id ORDER BY o.amount ) AS percentile
FROM crappy_data_db.orders o
)
SELECT 
	*,
	CASE WHEN percentile >= 0.9 THEN 'top 10%' 
	WHEN percentile >= 075 THEN 'top 25%' 
	WHEN percentile >= 0.25 THEN 'mid' ELSE 'bottom'
	END AS band
FROM users_orders_rank
ORDER BY user_id, amount DESC

---

## Task 3: Gaps-and-Islands — Monthly Order Streaks per User

**Scenario:**
The retention team wants to find users with long consecutive-month ordering streaks — months where a user placed at least one order, with no gap months in between.

For each user, find their **longest streak** of consecutive calendar months with at least one order.

**Expected Output Columns:**
- `user_id`
- `streak_start` (date — first month of the streak, DATE_TRUNC to month)
- `streak_end` (date — last month of the streak)
- `streak_months` (integer — length of the streak)

Only return each user's single longest streak. If tied, return the most recent one.

**Tables:** `orders`

**Requirements:**
- Use DATE_TRUNC('month', ...) to collapse orders to months
- Use the gaps-and-islands approach: ROW_NUMBER to identify streak groups
- Order by `streak_months DESC`, `user_id ASC`

**Difficulty Rating:** 5/5

WITH orders_months AS (
SELECT 
	*,
	DATE_TRUNC('month', created_at) AS MONTH,
	LAG(DATE_TRUNC('month', created_at)) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_month
FROM crappy_data_db.orders o
),
users_new_streaks AS (
SELECT 
	*,
	CASE WHEN prev_month IS NULL OR month - prev_month > INTERVAL '31 Days' THEN 1 ELSE 0 END AS is_new_streak
FROM orders_months
),
users_streak_ids AS (
SELECT 
	*,
	SUM(is_new_streak) OVER (PARTITION BY user_id ORDER BY created_at) AS streak_id
FROM users_new_streaks
),
users_streaks AS (
SELECT 
	user_id,
	MIN(month) AS streak_start,
	MAX(month) AS streak_end,
	COUNT(DISTINCT(month)) AS streak_months
FROM users_streak_ids
GROUP BY user_id, streak_id
ORDER BY streak_months DESC, user_id
),
streaks_rn AS (
SELECT
	*,
	ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY streak_months DESC, streak_start)
FROM users_streaks
)
SELECT 
	user_id,
	streak_start,
	streak_end,
	streak_months
FROM streaks_rn
WHERE ROW_NUMBER = 1

---

## Submission Instructions

1. Task 1 — Type A 2-level UNION ALL rollup, no recursion (3/5)
2. Task 2 — PERCENT_RANK percentile bands per user (4/5)
3. Task 3 — Longest consecutive monthly order streak per user (5/5)
