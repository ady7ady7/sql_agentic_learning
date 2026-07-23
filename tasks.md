# SQL Tasks — Week 31 Day 4

**Generated:** 2026-07-23
**Dataset:** crappy_data (users, orders, transactions, products, orders_products, deliveries, chat_tickets, chat_messages)

---

## Task 1: Product Affinity — Co-purchase Pairs
**Difficulty: 3/5**

**Business question:**
Which product pairs are most frequently bought together in the same order? Return all pairs that appear together in at least 5 orders, with their co-purchase count. Order by frequency descending.

**Expected output columns:**
`product_id_1, product_id_2, co_purchase_count`


WITH orders_products_agg AS (
SELECT 
	op1.product_id AS pid1,
	op2.product_id AS pid2
FROM crappy_data_db.orders_products op1
JOIN crappy_data_db.orders_products op2 ON op1.order_id = op2.order_id
AND op1.product_id > op2.product_id
)
SELECT 
	pid1,
	pid2,
	COUNT(*) AS co_purchase_count
FROM orders_products_agg
GROUP BY pid1, pid2
HAVING COUNT(*) >= 5
ORDER BY co_purchase_count DESC



---

## Task 2: User Purchase Sessions
**Difficulty: 4/5**

**Business question:**
Define a "purchase session" as a cluster of orders where consecutive orders (per user) are no more than 7 days apart. Orders separated by more than 7 days start a new session.

For each user with at least 2 orders, return: total number of sessions, the size of their longest session (in orders), and the average number of days between orders across their entire order history.

**Expected output columns:**
`user_id, total_sessions, longest_session_orders, avg_days_between_orders`


WITH orders_users_po AS (
SELECT 
	*,
	LAG(created_at) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_order_created_at
FROM crappy_data_db.orders o
),
orders_streak_keys AS (
SELECT 
	*,
	CASE WHEN prev_order_created_at IS NULL OR ((created_at - prev_order_created_at) > INTERVAL '7' DAY) THEN 1 ELSE 0 END AS streak_key
FROM orders_users_po
),
users_streak_ids AS (
SELECT 
	*,
	COALESCE(ROUND(EXTRACT(EPOCH FROM created_at - prev_order_created_at) / 86400, 0), 0) AS days_between_orders,
	SUM(streak_key) OVER (PARTITION BY user_id ORDER BY created_at) AS streak_id
FROM orders_streak_keys
),
users_sessions AS (
SELECT 
	user_id,
	streak_id,
	COUNT(*) AS orders,
	AVG(days_between_orders) AS avg_days_between_orders
FROM users_streak_ids
GROUP BY user_id, streak_id
ORDER BY user_id
)
SELECT 
	user_id,
	COUNT(*) AS total_sessions,
	MAX(orders) AS longest_session_orders,
	AVG(avg_days_between_orders) AS avg_days_between_orders
FROM users_sessions
GROUP BY user_id

---

## Task 3: Support Ticket Resolution Funnel
**Difficulty: 4/5**

**Business question:**
For each ticket priority level, compute what % of tickets were resolved within 1 day, within 7 days, and what % were never resolved (status never reached `resolved` or `closed`). Base the time calculation on `created_at` and `updated_at`.

**Expected output columns:**
`priority, total_tickets, pct_resolved_1d, pct_resolved_7d, pct_never_resolved`


i'VE slightly modified the logic to check which speed_buckets do they fall in if resolved. That's done because honestly there are only 30 tickets in the database, and your approach wouldn't make sense. I simply added the avg_resolve_time to show some data. I think I've made up for the changes well with ym approach.


WITH tickets_agg AS (
SELECT 
	*,
	EXTRACT(EPOCH FROM updated_at - created_at) / 60 AS ticket_update_time_in_minutes,
	CASE WHEN status = 'resolved' THEN 1 ELSE 0 END AS resolved
FROM crappy_data_db.chat_tickets ct
),
resolve_time_buckets AS (
SELECT
	*,
	NTILE(2) OVER (ORDER BY ticket_update_time_in_minutes) AS ticket_resolve_speed_bucket
FROM tickets_agg
)
SELECT 
	priority,
	COUNT(*) AS total_tickets,
	AVG(ticket_update_time_in_minutes) FILTER (WHERE resolved = 1) AS avg_resolve_time,
	ROUND(COUNT(*) FILTER (WHERE ticket_resolve_speed_bucket = 1 AND resolved = 1) / COUNT(*)::NUMERIC * 100, 2) AS pct_resolved_faster,
	ROUND(COUNT(*) FILTER (WHERE ticket_resolve_speed_bucket = 2 AND resolved = 1) / COUNT(*)::NUMERIC * 100, 2) AS pct_resolved_slower,
	ROUND(COUNT(*) FILTER (WHERE resolved = 0) / COUNT(*)::NUMERIC * 100, 2) AS pct_never_resolved
FROM resolve_time_buckets
GROUP BY priority
---

## Submission Instructions

Paste your queries and results below each task.
