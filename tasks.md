# Daily SQL Practice Tasks

**Generated:** 2026-04-13
**Week 18, Day 1 Focus:** LEFT JOIN + COALESCE base population + LEAD

---

## Task 1: LEFT JOIN + COALESCE — All Active Users with Order Metrics

**Scenario:**
The growth team wants a full list of all users with their order stats — but users who have never placed an order must still appear in the result with zeros, not be silently dropped.

**Expected Output Columns:**
- `user_id` (integer)
- `country` (varchar)
- `order_count` (integer) — 0 if no orders
- `total_spent` (numeric) — 0.00 if no orders
- `avg_order_value` (numeric) — NULL if no orders (can't average zero orders)

**Requirements:**
- Base: all users from `users` table
- LEFT JOIN aggregated order metrics onto the user base
- Use COALESCE to convert NULL order_count and total_spent to 0
- avg_order_value should remain NULL for users with no orders — do not COALESCE it to 0
- Exclude NULL countries
- Order by `total_spent DESC`

**Difficulty Rating:** 3/5

SELECT 
	u.id AS user_id,
	u.country,
	COALESCE(COUNT(o.id), 0) AS order_count,
	ROUND(COALESCE(SUM(o.amount), 0)::NUMERIC, 2) AS total_spent,
	ROUND(AVG(o.amount)::NUMERIC, 2) AS avg_order_value
FROM crappy_data_db.users u
LEFT JOIN crappy_data_db.orders o ON u.id = o.user_id
WHERE u.country IS NOT NULL
GROUP BY u.id, u.country
ORDER BY total_spent DESC




---

## Task 2: LEAD — Days Until Next Order per User

**Scenario:**
The retention team wants to understand ordering cadence: for each order, how many days until the same user placed their next order? Orders with no subsequent order should show NULL.

**Expected Output Columns:**
- `user_id` (integer)
- `order_id` (integer)
- `order_date` (date) — DATE(created_at)
- `next_order_date` (date) — date of the user's next order, NULL if none
- `days_until_next` (integer) — next_order_date minus order_date in days, NULL if no next order

**Requirements:**
- Use `orders`, exclude NULL amounts
- Use LEAD(DATE(created_at)) OVER (PARTITION BY user_id ORDER BY created_at ASC)
- Only include users with at least 2 orders
- Order by `user_id ASC`, `order_date ASC`

**Difficulty Rating:** 3/5

WITH orders_dates AS (
SELECT
	*,
	DATE(created_at) AS order_date,
	LEAD(DATE(created_at)) OVER (PARTITION BY user_id ORDER BY created_at) AS next_order_date
FROM crappy_data_db.orders o
),
orders_cnt AS (
SELECT 
user_id,
COUNT(*) AS order_cnt
FROM crappy_data_db.orders o
GROUP BY user_id
)
SELECT 
	od.user_id,
	od.id AS order_id,
	od.order_date,
	od.next_order_date,
	oc.order_cnt,
	CASE WHEN od.next_order_date IS NULL THEN NULL ELSE od.next_order_date - od.order_date END AS days_until_next 
FROM orders_dates od
JOIN orders_cnt oc ON od.user_id = oc.user_id AND oc.order_cnt >= 2
ORDER BY od.user_id, od.order_date


I've added order_cnt to make sure data prints properly. It's all good.

---

## Task 3: LEFT JOIN Chain — Active Users with Orders and Transactions (5/5)

**Scenario:**
The finance team wants to understand engagement depth: for every active user, show their order count, transaction count, and total transaction amount — even if they have no orders or no transactions. Then flag users who have orders but zero transactions (potential payment issues).

**Expected Output Columns:**
- `user_id` (integer)
- `order_count` (integer) — 0 if none
- `transaction_count` (integer) — 0 if none
- `total_transaction_amount` (numeric) — 0.00 if none
- `has_orders_no_transactions` (boolean) — true if order_count > 0 AND transaction_count = 0

**Requirements:**
- Base: all users from `users` table
- LEFT JOIN order metrics (pre-aggregated) onto users
- LEFT JOIN transaction metrics (pre-aggregated) onto users
- Both joins must be LEFT — no active user should be dropped
- COALESCE all counts and amounts to 0
- Order by `has_orders_no_transactions DESC`, `order_count DESC`

**Difficulty Rating:** 5/5

That's marked as 5/5 difficulty, but that was very easy for me.

WITH users_metrics AS (
SELECT 
	u.id AS user_id,
	COALESCE(COUNT(o.id), 0) AS order_count,
	COALESCE(COUNT(t.id), 0) AS transaction_count,
	COALESCE(SUM(t.amount), 0.00) AS total_transaction_amount
FROM crappy_data_db.users u
LEFT JOIN crappy_data_db.orders o ON u.id = o.user_id
LEFT JOIN crappy_data_db.transactions t ON u.id = t.user_id
GROUP BY u.id
)
SELECT 
	*,
	(order_count > 0) AND (transaction_count = 0) AS has_orders_no_transactions
FROM users_metrics 
ORDER BY has_orders_no_transactions DESC, order_count DESC




---

## Submission Instructions

1. Task 1 — LEFT JOIN + COALESCE: all active users with order metrics, zeros for no orders (3/5)
2. Task 2 — LEAD: days until next order per user (3/5)
3. Task 3 — LEFT JOIN chain: active users with orders + transactions, flag payment gap (5/5)
