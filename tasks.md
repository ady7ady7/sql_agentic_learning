# Daily SQL Practice Tasks

**Generated:** 2026-03-04
**Week 12, Day 3 Focus:** HackerRank Hard — Exam Simulation Style

---

## Task 1: 3-Level Hierarchy — Transaction Types, Top Users, and Their Cities

**Scenario:**
Build a 3-level hierarchy over transaction data:
- Level 1: `'All Transactions'`
- Level 2: Distinct transaction types (dynamic, from `transactions`)
- Level 3: For each type, the 3 users with the highest total transaction amount — show their city (or `'Unknown'` if NULL)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — type at Level 2, city name (or `'Unknown'`) at Level 3
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Join `transactions → users` to get city per user
- Pre-aggregate total amount per user+type, rank, keep top 3
- Use `COALESCE(city, 'Unknown')` for NULL cities
- Termination condition required

WITH RECURSIVE transaction_types_amounts AS (
SELECT 
	user_id,
	TYPE,
	SUM(amount) AS transaction_amount
FROM transactions
GROUP BY user_id, TYPE
),
user_type_ranks AS (
SELECT 
	tta.user_id,
	u.city,
	tta."type",
	tta.transaction_amount,
	RANK() OVER (PARTITION BY TYPE ORDER BY transaction_amount DESC) AS type_rank
FROM transaction_types_amounts tta
JOIN users u ON u.id = tta.user_id
),
top_three_user_per_transaction_type AS (
SELECT * FROM user_type_ranks
WHERE type_rank <= 3
),
distinct_types AS (
SELECT 
DISTINCT TYPE
FROM transactions
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
	COALESCE(ds.TYPE, ttu.city),
	h.name,
	h.PATH || ' < ' || COALESCE(ds.TYPE, ttu.city)
FROM HIERARCHY h
LEFT JOIN distinct_types ds ON h.LEVEL = 1
LEFT JOIN top_three_user_per_transaction_type ttu ON h.LEVEL = 2 AND ttu.TYPE = h.name
WHERE h.LEVEL < 3
)
SELECT * FROM hierarchy



**Difficulty Rating:** 4/5

---

## Task 2: Global First Purchase vs Repeat — Monthly Revenue Split

**Scenario:**
The growth team wants to understand monthly revenue from brand-new buyers (placing their very first order ever) vs returning buyers (any order after their first).

For each month, show:

**Expected Output Columns:**
- `month` (date) — truncated to month
- `purchase_type` (text) — `'first_purchase'` or `'repeat_purchase'`
- `order_count` (bigint)
- `total_revenue` (numeric) — rounded to 2 decimals
- `pct_of_monthly_revenue` (numeric) — this type's % of total revenue that month, rounded to 1 decimal

**Requirements:**
- Use `orders` table only
- A user's **global first order** = the single earliest order across their entire history (not per month)
- All other orders by that user = `repeat_purchase`
- Aggregate to one row per month+type combination
- `pct_of_monthly_revenue`: window SUM partitioned by month, then divide
- Order by `month ASC`, `purchase_type ASC`

**Difficulty Rating:** 5/5


WITH orders_first_transactions AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_,
	FIRST_VALUE(created_at) OVER (PARTITION BY user_id ORDER BY created_at) AS first_transaction_time
FROM  orders
),
orders_purchase_types AS (
SELECT 
	*,
	CASE WHEN created_at = first_transaction_time THEN 'first_purchase' ELSE 'repeat_purchase' END AS purchase_type
FROM orders_first_transactions
),
monthly_purchase_type_revenues AS (
SELECT 
	month_,
	purchase_type,
	COUNT(id) AS order_count,
	SUM(amount) AS total_revenue
FROM orders_purchase_types
GROUP BY month_, purchase_type
ORDER BY month_
),
total_monthly_revenues AS (
SELECT 
	month_,
	SUM(amount) AS total_monthly_revenue
FROM orders_first_transactions
GROUP BY month_
)
SELECT 
	mpt.month_,
	mpt.purchase_type,
	mpt.order_count,
	mpt.total_revenue,
	round(mpt.total_revenue::numeric / tmr.total_monthly_revenue::numeric * 100, 1) || '%' AS pct_of_monthly_revenue
FROM monthly_purchase_type_revenues mpt
JOIN total_monthly_revenues tmr ON mpt.month_ = tmr.month_
ORDER BY mpt.month_, mpt.purchase_type

Here, it works perfectly.

---

## Task 3: Order Response Time — Time from Order to First Delivery Update

**Scenario:**
The operations team wants to measure fulfillment speed — specifically, how many hours pass between an order being placed and its delivery record being created.

**Expected Output Columns:**
- `order_id` (integer)
- `order_created_at` (timestamp)
- `delivery_created_at` (timestamp)
- `hours_to_fulfillment` (numeric) — rounded to 2 decimals using EPOCH
- `fulfillment_segment` (text):
  - `'same_day'`: fulfilled within 24 hours
  - `'next_day'`: 24 to 48 hours
  - `'delayed'`: more than 48 hours

**Requirements:**
- Use `orders` and `deliveries` tables
- Use `EXTRACT(EPOCH FROM ...) / 3600` for hours
- Only include orders that have a delivery record
- Order by `hours_to_fulfillment ASC`

**Difficulty Rating:** 3/5

WITH orders_deliveries AS (
SELECT 
	d.order_id,
	o.created_at AS order_created_at,
	d.created_at AS delivery_created_at
FROM orders o
JOIN deliveries d ON o.id = d.order_id
WHERE d.status = 'delivered'
),
deliveries_fulfillment_hours AS (
SELECT 
	*,
	EXTRACT('Epoch' FROM delivery_created_at - order_created_at)/3600 AS hours_to_fulfillment
FROM orders_deliveries
)
SELECT 
	*,
	CASE 
		WHEN hours_to_fulfillment < 24 THEN 'same_day'
		WHEN hours_to_fulfillment <= 48 THEN 'next_day'
		WHEN hours_to_fulfillment > 48 THEN 'delayed'
	END AS fulfillment_segment
FROM deliveries_fulfillment_hours
ORDER BY hours_to_fulfillment


---

## Submission Instructions

1. Task 1 — Transaction type/city hierarchy (4/5)
2. Task 2 — Global first purchase vs repeat monthly split (5/5)
3. Task 3 — Order to delivery fulfillment time (3/5)
