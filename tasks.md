# Daily SQL Practice Tasks

**Generated:** 2026-04-21
**Week 19, Day 2 Focus:** dominant_type via CTE + Type A recursive CTE rollup + NULLIF safe division

---

## Task 1: dominant_type via Two CTEs

**Scenario:**
The finance team wants to know each user's dominant transaction type — the type they use most often.

For each user who has at least 1 transaction, show:
- `user_id`
- `dominant_type` — the transaction type with the highest count for that user

**Tables:** `transactions`

**Requirements:**
- CTE 1: `GROUP BY user_id, type` to get a count per user per type
- CTE 2: use `RANK() OVER (PARTITION BY user_id ORDER BY cnt DESC)` to rank types per user
- Final SELECT: filter to rank = 1

**Difficulty Rating:** 3/5

WITH users_cnt_filters AS (
SELECT 
	user_id,
	COUNT(*) FILTER (WHERE TYPE = 'deposit') AS deposit_count,
	COUNT(*) FILTER (WHERE TYPE = 'withdrawal') AS withdrawal_count,
	COUNT(*) FILTER (WHERE TYPE = 'purchase') AS purchase_count,
	COUNT(*) FILTER (WHERE TYPE = 'transfer') AS transfer_count,
	COUNT(*) FILTER (WHERE TYPE = 'payment') AS payment_count
FROM crappy_data_db.transactions t
GROUP BY user_id
),
transaction_type_cnts AS (
SELECT 
	user_id,
	TYPE,
	COUNT(*) AS transaction_cnt
FROM crappy_data_db.transactions t
GROUP BY user_id, TYPE
),
transaction_types_rank AS (
SELECT
	*,
	RANK() OVER (PARTITION BY user_id ORDER BY transaction_cnt DESC, type) AS type_rank
FROM transaction_type_cnts
)
SELECT 
	uc.user_id,
	uc.deposit_count,
	uc.withdrawal_count,
	uc.purchase_count,
	uc.transfer_count,
	uc.payment_count,
	ttr.TYPE AS dominant_type
FROM users_cnt_filters uc
JOIN transaction_types_rank ttr ON uc.user_id = ttr.user_id AND ttr.type_rank = 1






---

## Task 2: 3-Level Revenue Rollup — Category → Product → Order Line

**Scenario:**
The product team wants a unified 3-level revenue breakdown in a single result set:
- **Level 1:** Total revenue per product category
- **Level 2:** Total revenue per product (within its category)
- **Level 3:** Total revenue per order line (`quantity × product.price`)

Use the **Type A fixed-hierarchy CTE** pattern:
- CTE 1: order-line level revenue
- CTE 2: product level (sum of line revenues)
- CTE 3: category level (sum of product revenues)
- Final `UNION ALL`: combine all three with a `level` integer and a `label` text column

**Expected Output Columns:**
- `level` (integer — 1, 2, or 3)
- `label` (text — category name / product name / `'Order line #' || order_id`)
- `revenue` (numeric, rounded to 2 decimals)

**Order by:** `level ASC`, `revenue DESC`

**Tables:** `orders_products`, `products`, `product_categories`

**Difficulty Rating:** 5/5


WITH RECURSIVE products_orders_revenues AS (
SELECT 
	op.order_id,
	op.product_id,
	SUM(p.price * op.quantity) AS order_line_revenue
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.products p ON op.product_id = p.id
GROUP BY op.order_id, op.product_id
),
products_total_revenues AS (
SELECT 
	op.product_id,
	p.category_id,
	p.name AS product_name,
	pc.name AS category_name,
	SUM(p.price * op.quantity) AS product_revenue
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.products p ON op.product_id = p.id
JOIN crappy_data_db.product_categories pc ON p.category_id  = pc.id
GROUP BY op.product_id, p.category_id, p.name, pc.name
),
categories_revenues AS (
SELECT 
	p.category_id,
	pc.name AS category_name,
	SUM(p.price * op.quantity) AS category_revenue
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.products p ON op.product_id = p.id
JOIN crappy_data_db.product_categories pc ON p.category_id  = pc.id
GROUP BY p.category_id, pc.name
),
HIERARCHY AS (
SELECT 
	1 AS LEVEL,
	cr.category_name::TEXT AS LABEL,
	cr.category_revenue AS revenue
FROM categories_revenues cr
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(ptr.product_name::TEXT, 'Order line #' || por.order_id),
	COALESCE(ptr.product_revenue, por.order_line_revenue)
FROM HIERARCHY h 
LEFT JOIN products_total_revenues ptr ON h.label = ptr.category_name
LEFT JOIN products_orders_revenues por ON h.LEVEL = 2
WHERE H.LEVEL < 3
)
SELECT * FROM HIERARCHY
ORDER BY LEVEL, revenue DESC

Frankly, it's weird to do such combinations. It doesn't seem tobe real.

---

## Task 3: NULLIF — Safe Metrics on Dirty Data

**Scenario:**
The analytics team is computing per-user engagement metrics from `user_sessions_daily`, but the data has a known issue: some rows have `count_sessions = 0` (days the user was technically logged but had no activity). These zeros skew averages.

For each user, compute:
- `user_id`
- `active_days` — number of days where `count_sessions > 0`
- `total_sessions` — sum of all sessions
- `avg_sessions_per_active_day` — `total_sessions / active_days`, but use `NULLIF` to avoid division by zero for users with no active days (return NULL instead of crashing)

**Tables:** `user_sessions_daily`

**Requirements:**
- Use `NULLIF(active_days, 0)` in the division
- Only include users who appear in the table (at least 1 row, even if all zeros)

**Difficulty Rating:** 3/5

WITH user_session_metrics AS (
SELECT 
	user_id,
	COUNT(*) AS active_days,
	SUM(count_sessions) AS total_sessions,
	ROUND(AVG(count_sessions), 1) AS avg_sessions
FROM crappy_data_db.user_sessions_daily usd
GROUP BY user_id
)
SELECT 
	*,
	ROUND(total_sessions/ active_days::NUMERIC, 1) AS avg_sessions_per_active_day
FROM user_session_metrics

FYI: There are no such cases that there's a day listed for a given user with 0 sessions. I'd have to do an artificial aggregation on every day and LEFT JOIN to achieve such a session, but there wouldn't be a point for that. I double checked simply using AVG in the first CTE and then manually dividing the sessions, but obviously they've given the same results. Anyway, I've achieved what you wanted, no NULLIF needed here.

A simply one-off approach would work the same way here


SELECT 
	user_id,
	COUNT(*) AS active_days,
	SUM(count_sessions) AS total_sessions,
	ROUND(AVG(count_sessions), 1) AS avg_sessions
FROM crappy_data_db.user_sessions_daily usd
GROUP BY user_id


---

## Submission Instructions

1. Task 1 — Conditional aggregation + dominant_type via second CTE (4/5)
2. Task 2 — 3-level revenue rollup, Type A CTE pattern (5/5)
3. Task 3 — NULLIF safe division on session data (3/5)
