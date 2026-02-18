# Daily SQL Practice Tasks

**Generated:** 2026-02-18
**Week 10, Day 3 Focus:** Gaps-and-Islands Mastery (Scaffolded) + Hierarchy + Rolling Windows

---

## Task 1: 3-Level Hierarchy — Product Categories and Top Products

**Scenario:**
Build a 3-level hierarchy over product data:
- Level 1: `'All Products'`
- Level 2: Distinct category names from `product_categories` (pulled dynamically)
- Level 3: Top 3 most expensive products per category (show product name as text)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — category name at Level 2, product name at Level 3
- `parent_name` (text)
- `path` (text) — full path from root

**Requirements:**
- Pre-aggregate distinct categories and top-3-per-category before the recursive CTE
- Use `price DESC` to define "most expensive"
- Termination condition required
- No hardcoded category values

**Difficulty Rating:** 3/5


WITH RECURSIVE products_categories_rank AS (
SELECT 
	p.name AS product_name,
	pc.name AS category_name,
	pc.id AS category_id,
	p.price,
	RANK() OVER (PARTITION BY category_id ORDER BY price DESC) AS category_price_rank
FROM products p JOIN product_categories pc ON p.category_id  = pc.id
),
distinct_categories AS (
SELECT 
	DISTINCT id AS category_id,
	name AS category_name
FROM product_categories
),
top_three_products_per_category AS (
SELECT 
	*
FROM products_categories_rank
WHERE category_price_rank <= 3
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Products' AS name,
	NULL::TEXT AS parent_name,
	'All Products' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(dc.category_name::TEXT, ttp.product_name::TEXT),
	h.name,
	h.path || ' > ' || COALESCE(dc.category_name::TEXT, ttp.product_name::TEXT)
FROM HIERARCHY h
LEFT JOIN distinct_categories dc ON h.LEVEL = 1
LEFT JOIN top_three_products_per_category ttp ON h.name = ttp.category_name AND h.LEVEL = 2
WHERE H.LEVEL < 3
)
SELECT * FROM hierarchy


Everything works properly.

---

## Task 2: Gaps-and-Islands — Scaffolded Drill (User Sessions)

This task is broken into 3 sub-questions that build the pattern incrementally. Complete each step before moving to the next.

### Step A — Generate the streak key
Using `user_sessions_daily`, write a query that:
- Filters to active days only (the table only contains active days, so no filter needed)
- Assigns a `ROW_NUMBER()` per user ordered by date
- Computes `streak_key` as `date - (rn * INTERVAL '1 day')`

Output: `user_id`, `date`, `count_sessions`, `rn`, `streak_key`

No aggregation yet — just show the raw rows with the computed key. Pick user_id = 1 to inspect visually.

**Expected insight:** Dates within the same consecutive streak share the same `streak_key`. A gap in dates causes `streak_key` to shift.

---

### Step B — Aggregate streaks
Using Step A as a CTE, GROUP BY `(user_id, streak_key)` to produce one row per streak.

Output: `user_id`, `streak_key`, `streak_start` (MIN date), `streak_end` (MAX date), `streak_length` (COUNT), `avg_daily_sessions` (AVG, rounded to 2 decimals)

No filtering yet — show all streaks for all users.

---

### Step C — Pick the longest streak per user
Using Step B as a CTE, add a final step that:
- Ranks streaks per user by `streak_length DESC`, then `streak_end DESC` (most recent if tied)
- Keeps only rank = 1 (longest streak per user)
- Filters to users whose longest streak is >= 5 days
- Orders by `streak_length DESC`, `avg_daily_sessions DESC`

Final output: `user_id`, `streak_start`, `streak_end`, `streak_length`, `avg_daily_sessions`

**This is the complete solution to Day 2 Task 2 — assembled step by step.**

**Difficulty Rating:** 4/5 (scaffolded, but you must write all three steps)

WITH users_sessions_streak_keys AS (
SELECT 
	*,
	ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY date) AS rn,
	date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY date) * INTERVAL '1' DAY) AS streak_key
FROM user_sessions_daily
),
users_session_streaks_consecutive_days AS (
SELECT 
	user_id,
	streak_key,
	MIN(date) AS streak_start,
	MAX(date) AS streak_end,
	COUNT(*) AS streak_length,
	ROUND(AVG(count_sessions), 2) AS avg_daily_sessions
FROM users_sessions_streak_keys
GROUP BY user_id, streak_key
ORDER BY user_id
),
users_streak_ranks AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY user_id ORDER BY streak_length DESC, streak_end DESC) AS streak_rank
FROM users_session_streaks_consecutive_days
)
SELECT 
	user_id,
	streak_start,
	streak_end,
	streak_length,
	avg_daily_sessions
FROM users_streak_ranks 
WHERE streak_rank = 1


Nice, it makes sense now - we definitely MUST practice this pattern more.

---

## Task 3: 7-Day Rolling Order Revenue

**Scenario:**
The finance team wants a daily rolling revenue report. For each day in the `dates` table (within the range of actual order data), calculate the total order revenue for the past 7 days (including that day).

**Expected Output Columns:**
- `date` (date)
- `daily_revenue` (numeric) — total order revenue on that specific day, rounded to 2 decimals (0 if no orders)
- `rolling_7d_revenue` (numeric) — sum of daily_revenue over the past 7 days including today, rounded to 2 decimals

**Requirements:**
- Use `dates` table as the spine (left join orders to it)
- Only include dates within the min/max range of `orders.created_at`
- Use a window frame: `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`
- Days with no orders should show 0, not NULL
- Order by `date ASC`

**Difficulty Rating:** 4/5

WITH daily_orders_revenues AS (
SELECT 
	d.date,
	COALESCE(COUNT(o.id), 0) AS order_cnt,
	COALESCE(SUM(o.amount), 0) AS orders_revenue
FROM dates d
LEFT JOIN orders o ON d."date" = DATE(o.created_at)
GROUP BY d.date
ORDER BY d.date
)
SELECT 
	date,
	orders_revenue AS daily_revenue,
	ROUND(SUM(orders_revenue) OVER (ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)::NUMERIC, 2) AS rolling_7d_revenue
FROM daily_orders_revenues

Here, exactly as you wanted.

---

## Submission Instructions

1. Task 1 — Product category hierarchy (3/5)
2. Task 2 — Gaps-and-islands scaffolded drill, Steps A + B + C (4/5)
3. Task 3 — 7-day rolling revenue (4/5)
