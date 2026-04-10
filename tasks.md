# Daily SQL Practice Tasks

**Generated:** 2026-04-10
**Week 17, Day 4 Focus:** Type A recursive CTE (3-level) + FIRST_VALUE frame spec + complex multi-window scoring (5/5)

---

## Task 1: Type A Recursive CTE — 3-Level Product Category Revenue Rollup

**Scenario:**
The finance team wants a 3-level revenue rollup: individual product → product category → grand total. Each level should show the label, the revenue at that level, and the percentage of the grand total.

**Expected Output Columns:**
- `level` (integer) — 1 = product, 2 = category, 3 = grand total
- `label` (varchar) — product name, category name, or 'Grand Total'
- `revenue` (numeric) — sum of price × quantity for this node, rounded to 2 decimals
- `pct_of_total` (numeric) — revenue as % of grand total, rounded to 1 decimal

**Requirements:**
- Use `products`, `product_categories`, `orders_products`
- Only include rows where price IS NOT NULL
- Structure as three pre-aggregated CTEs (product_revenue, category_revenue, grand_total), then UNION ALL them together with a level column
- Order by `level ASC`, `revenue DESC`

**Difficulty Rating:** 4/5

WITH product_revenues AS (
SELECT 
	p.id AS product_id,
	p.category_id AS category_id,
	p.name AS LABEL,
	SUM(p.price * op.quantity) AS REVENUE
FROM crappy_data_db.products p
JOIN crappy_data_db.orders_products op ON op.product_id = p.id
GROUP BY p.id, p.name
),
categories_revenues AS (
SELECT 
	pc.id AS category_id,
	pc.name AS category_name,
	SUM(op.quantity * p.price) AS category_revenue
FROM crappy_data_db.products p
JOIN crappy_data_db.orders_products op ON op.product_id = p.id
JOIN crappy_data_db.product_categories pc ON p.category_id = pc.id
GROUP BY pc.id, pc.name
),
total_revenue AS (
SELECT 
	SUM(p.price * op.quantity) AS grand_total
FROM crappy_data_db.products p
JOIN crappy_data_db.orders_products op ON op.product_id = p.id
)
SELECT 
	1 AS LEVEL,
	pr.LABEL AS LABEL,
	pr.revenue AS revenue,
	ROUND(revenue / tr.grand_total * 100, 2) AS pct_of_total
FROM product_revenues pr
JOIN total_revenue tr ON tr.grand_total > pr.revenue
UNION ALL
SELECT 
	2 AS level,
	cr.category_name,
	cr.category_revenue,
	ROUND(cr.category_revenue / tr.grand_total * 100, 2)
FROM categories_revenues cr
JOIN total_revenue tr ON tr.grand_total > cr.category_revenue
UNION ALL
SELECT 
	3 AS LEVEL,
	'grand_total',
	tr.grand_total,
	ROUND(tr.grand_total / tr.grand_total * 100, 2)
FROM total_revenue tr

This was quite weird, but I've done it.


---

## Task 2: FIRST_VALUE — Each User's First and Most Recent Transaction Type

**Scenario:**
The product team wants to know, for each user, what transaction type they started with and what their most recent transaction type was — to detect whether users' behaviour has shifted over time.

**Expected Output Columns:**
- `user_id` (integer)
- `first_type` (text) — transaction type of the user's earliest transaction
- `last_type` (text) — transaction type of the user's most recent transaction
- `total_transactions` (integer)
- `shifted` (boolean) — true if first_type != last_type

**Requirements:**
- Use `transactions`, exclude NULL types and NULL user_ids
- Use `FIRST_VALUE(type ORDER BY created_at ASC)` for first_type
- Use `FIRST_VALUE(type ORDER BY created_at DESC)` for last_type
- Collapse to one row per user — use a CTE with window functions, then GROUP BY or DISTINCT
- Only include users with at least 3 transactions
- Order by `user_id ASC`

**Difficulty Rating:** 3/5

WITH users_transactions AS (
SELECT 
	*,
	FIRST_VALUE(type) OVER (PARTITION BY user_id ORDER BY t.created_at) first_transaction_type,
	FIRST_VALUE(type) OVER (PARTITION BY user_id ORDER BY t.created_at DESC) AS last_transaction_type
FROM crappy_data_db.transactions t
)
SELECT 
	user_id,
	first_transaction_type,
	last_transaction_type,
	COUNT(*) AS transactions,
	first_transaction_type = last_transaction_type AS shifted
FROM users_transactions
GROUP BY user_id, 	first_transaction_type, last_transaction_type
HAVING COUNT(*) >= 3
ORDER BY user_id


---

## Task 3: Multi-Window Scoring — User Engagement Score (5/5)

**Scenario:**
The growth team wants a composite engagement score for each user, combining three signals:
1. **Order frequency score**: NTILE(4) on total order count — quartile 4 = most orders
2. **Spend score**: NTILE(4) on total order amount — quartile 4 = highest spend
3. **Session score**: NTILE(4) on total session count — quartile 4 = most sessions

Final `engagement_score` = sum of the three NTILE values (max 12, min 3).

They then want to rank users by engagement_score and flag the top 10% using PERCENT_RANK.

**Expected Output Columns:**
- `user_id` (integer)
- `order_freq_score` (integer) — NTILE(4) on order count
- `spend_score` (integer) — NTILE(4) on total spend
- `session_score` (integer) — NTILE(4) on session count
- `engagement_score` (integer) — sum of the three scores
- `engagement_pct_rank` (numeric) — PERCENT_RANK() on engagement_score, rounded to 3 decimals
- `is_top_10pct` (boolean) — true if engagement_pct_rank >= 0.90

**Requirements:**
- Use `orders`, `user_sessions_daily`, and `users` (to get the full user list as base)
- A user with no orders gets order_freq_score = 1, spend_score = 1 (treat missing as lowest tier)
- A user with no sessions gets session_score = 1
- NTILE must be computed over all users (not just those with orders/sessions)
- Only include users where is_active = TRUE
- Order by `engagement_score DESC`, `user_id ASC`

**Difficulty Rating:** 5/5

WITH orders_metrics AS (
SELECT 
	user_id,
	SUM(o.amount) AS total_spent,
	COUNT(*) AS order_cnt
FROM crappy_data_db.orders o
JOIN crappy_data_db.users u ON o.user_id = u.id
WHERE u.is_active = True
GROUP BY o.user_id
),
sessions_totals AS (
SELECT 
	user_id,
	SUM(count_sessions) AS total_sessions
FROM crappy_data_db.user_sessions_daily usd
GROUP BY user_id
),
users_combined_metrics AS (
SELECT 
	om.user_id,
	NTILE(4) OVER (ORDER BY om.order_cnt) AS order_freq_score,
	NTILE(4) OVER (ORDER BY om.total_spent) AS spend_score,
	NTILE(4) OVER (ORDER BY st.total_sessions) AS session_score
FROM orders_metrics om
JOIN sessions_totals st ON om.user_id = st.user_id
),
users_engagement_score AS (
SELECT 
	*,
	order_freq_score + spend_score + session_score AS engagement_score,
	ROUND(percent_rank() OVER (ORDER BY (order_freq_score + spend_score + session_score))::NUMERIC, 3) AS engagement_pct_rank
FROM users_combined_metrics
)
SELECT 
	*,
	engagement_pct_rank >= 0.9 AS is_top_10pct
FROM users_engagement_score
ORDER BY engagement_score DESC

---

## Submission Instructions

1. Task 1 — Type A 3-level revenue rollup with UNION ALL (4/5)
2. Task 2 — FIRST_VALUE first and last transaction type per user (3/5)
3. Task 3 — Composite engagement score from three NTILE signals + PERCENT_RANK (5/5)
