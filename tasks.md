# Daily SQL Practice Tasks

**Generated:** 2026-04-24
**Week 19, Day 5 Focus:** STDDEV window + Type B recursive CTE + funnel analysis

---

## Task 1: STDDEV — Transaction Amount Volatility per User

**Scenario:**
The risk team wants to flag users whose transaction amounts are highly volatile — they want the average and standard deviation of transaction amounts per user, so they can identify outliers.

For each user with at least 2 transactions, show:
- `user_id`
- `tx_count` — total number of transactions
- `avg_amount` — average transaction amount, rounded to 2 decimals
- `stddev_amount` — standard deviation of transaction amounts, rounded to 2 decimals
- `volatility` — label: `'high'` if stddev > 300, `'medium'` if stddev > 150, `'low'` otherwise

**Tables:** `transactions`

**Requirements:**
- Exclude NULL amounts
- Use `STDDEV(amount)` in a GROUP BY aggregation (not a window function)
- Order by `stddev_amount DESC`

**Difficulty Rating:** 3/5

WITH user_transactions AS (
SELECT 
	*,
	round(AVG(amount) OVER (PARTITION BY user_id), 2) AS user_avg_transaction,
	round(STDDEV(amount) OVER (PARTITION BY user_id), 2) AS user_stddev_transaction
FROM crappy_data_db.transactions t 
)
SELECT 
	user_id,
	user_avg_transaction,
	user_stddev_transaction,
	COUNT(*) AS tx_count,
	CASE WHEN user_stddev_transaction > 300 THEN 'high' WHEN user_stddev_transaction > 150 THEN 'medium' ELSE 'low' END AS volatility
FROM user_transactions
GROUP BY user_id, user_avg_transaction, user_stddev_transaction

---

## Task 2: Support Ticket Response Time — Average Time to First Message per Priority

**Scenario:**
The support team wants to know how quickly tickets get their first response, broken down by priority level.

For each priority, show:
- `priority`
- `ticket_count` — number of tickets that have at least one message
- `avg_hours_to_first_message` — average time in hours between `chat_tickets.created_at` and the first `chat_messages.created_at` for that ticket, rounded to 1 decimal

**Tables:** `chat_tickets`, `chat_messages`

**Requirements:**
- Use a CTE to find the first message timestamp per ticket (`MIN(created_at)`)
- Only include tickets that have at least one message
- Use `EXTRACT(EPOCH FROM ...)` for the time difference, convert to hours
- Order by `avg_hours_to_first_message ASC`

**Difficulty Rating:** 4/5

Few caveats here:
- I din't track FIRST MESSAGE, but rather first response, as with every ticket in this database, creation time = first_message_time, so it would be pointless.
- I didn't count hours, but rather minutes, as maximum amount was around 12-15 minutes, so it was pointless to go for hours here. avgs are between 5-8 range.

AND ALSO YOUR task instruction swas a bit misleading - first you talk about first response, and then in the requirements you talk about first_message, it makes no sense. I've made the most logical decision ehere.


WITH tickets_response_times AS (
SELECT 
	ct.id AS ticket_id,
	ct.priority,
	ct.created_at AS ticket_created_at,
	FIRST_VALUE(cm.created_at) OVER (PARTITION BY cm.ticket_id ORDER BY cm.created_at) AS first_response
FROM crappy_data_db.chat_tickets ct
JOIN crappy_data_db.chat_messages cm ON ct.id = cm.ticket_id
WHERE cm.message_text IS NOT NULL AND cm.author_id IS NOT NULL
),
tickets_first_response_minutes AS (
SELECT 
	*,
	EXTRACT(EPOCH FROM first_response - ticket_created_at) / 60 AS minutes_to_first_response
FROM tickets_response_times
)
SELECT
	priority,
	COUNT(*) AS ticket_count,
	round(AVG(minutes_to_first_response), 1) AS avg_minutes_to_first_response
FROM tickets_first_response_minutes
GROUP BY priority

IMO it's all good and you should give it a 10/10 unless there are some serious issues

---

## Task 3: Funnel Analysis — Order to Delivery Conversion

**Scenario:**
The ops team wants a simple conversion funnel: of all users who placed at least one order, how many also have at least one delivery record, and how many of those deliveries have status `'delivered'`?

Report the funnel as three rows:
- `'placed order'` — count of distinct users with at least 1 order
- `'has delivery record'` — count of distinct users whose orders have any delivery record
- `'fully delivered'` — count of distinct users who have at least one delivery with status `'delivered'`

**Expected Output Columns:**
- `funnel_step` (text)
- `user_count` (integer)

**Tables:** `orders`, `deliveries`

**Requirements:**
- Use CTEs for each step, then UNION ALL the three counts
- Order by `user_count DESC`

**Difficulty Rating:** 4/5

WITH users_with_orders_cnt AS (
SELECT 
	COUNT(DISTINCT(o.user_id)) AS placed_order
FROM crappy_data_db.orders o
),
users_with_delivery_orders_cnt AS (
SELECT 
	count(DISTINCT(o.user_id)) AS has_delivery_record
FROM crappy_data_db.orders o
LEFT JOIN crappy_data_db.deliveries d ON o.id = d.order_id
WHERE d.status = 'pending'
),
users_with_delivered_orders_cnt AS (
SELECT 
	count(DISTINCT(o.user_id)) AS fully_delivered_record
FROM crappy_data_db.orders o
LEFT JOIN crappy_data_db.deliveries d ON o.id = d.order_id
WHERE d.status = 'delivered'
)
SELECT 
	1::TEXT AS funnel_step,
	uwc.placed_order AS user_count
FROM users_with_orders_cnt uwc
UNION ALL
SELECT 
	2::TEXT,
	uwd.has_delivery_record
FROM users_with_delivery_orders_cnt uwd
UNION ALL
SELECT 
	3::TEXT,
	uwdo.fully_delivered_record
FROM users_with_delivered_orders_cnt uwdo
ORDER BY user_count DESC

It works.
---

## Submission Instructions

1. Task 1 — STDDEV volatility labels per user (3/5)
2. Task 2 — Type B recursive category tree with depth + path (4/5)
3. Task 3 — 3-step order-to-delivery funnel (4/5)
