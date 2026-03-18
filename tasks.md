# Daily SQL Practice Tasks

**Generated:** 2026-03-18
**Week 14, Day 3 Focus:** Self-Referencing CTE (Anchor Fix) + Time-Proximity Gaps-and-Islands Edge Case + PIVOT Reinforcement

---

## Task 1: Self-Referencing CTE — Fix the Anchor

**Scenario:**
Use the same category tree as yesterday:

```sql
WITH RECURSIVE categories (id, name, parent_id) AS (
    VALUES
    (1, 'All Products',  NULL::int),
    (2, 'Electronics',   1),
    (3, 'Clothing',      1),
    (4, 'Phones',        2),
    (5, 'Laptops',       2),
    (6, 'Men',           3),
    (7, 'Women',         3),
    (8, 'iPhone',        4),
    (9, 'Samsung',       4),
    (10, 'T-Shirts',     6)
)
```

This time, write the anchor by **selecting from the `categories` CTE** with `WHERE parent_id IS NULL` — do not hardcode any id or name. The query must work correctly even if the root node changes.

**Expected Output Columns:**
- `id` (integer)
- `name` (text)
- `depth` (integer) — 1 for root
- `path` (text) — separator ` -> `

**Requirements:**
- Anchor: `SELECT ... FROM categories WHERE parent_id IS NULL`
- Recursive: `JOIN categories ON categories.parent_id = cte.id`
- Natural termination
- Order by `path ASC`

**Difficulty Rating:** 2/5


WITH RECURSIVE categories (id, name, parent_id) AS (
    VALUES
    (1, 'All Products',  NULL::int),
    (2, 'Electronics',   1),
    (3, 'Clothing',      1),
    (4, 'Phones',        2),
    (5, 'Laptops',       2),
    (6, 'Men',           3),
    (7, 'Women',         3),
    (8, 'iPhone',        4),
    (9, 'Samsung',       4),
    (10, 'T-Shirts',     6)
),
hierarchy AS (
SELECT 
	*,
	name AS path
FROM categories
WHERE parent_id IS NULL
UNION ALL
SELECT
	c.id,
	c.name,
	c.parent_id,
	h.PATH || '->' || c.name
FROM HIERARCHY h
JOIN categories c ON c.parent_id = h.id
)
SELECT * FROM hierarchy

---

## Task 2: Time-Proximity Gaps-and-Islands — Session Windows

**Scenario:**
You have a stream of user events (transactions) and want to group them into "sessions" — bursts of activity where consecutive events are within 60 minutes of each other. If the gap between two consecutive events for the same user exceeds 60 minutes, it's a new session.

Use this inline data as your source:

```sql
WITH events (user_id, event_time) AS (
    VALUES
    (1, '2024-01-01 08:00'::timestamp),
    (1, '2024-01-01 08:30'::timestamp),
    (1, '2024-01-01 08:58'::timestamp),
    (1, '2024-01-01 09:03'::timestamp),
    (1, '2024-01-01 10:30'::timestamp),
    (1, '2024-01-01 11:45'::timestamp),
    (2, '2024-01-01 09:00'::timestamp),
    (2, '2024-01-01 09:50'::timestamp),
    (2, '2024-01-01 11:00'::timestamp)
)
```

Group events into sessions. The 08:58 and 09:03 events are only 5 minutes apart — they belong to the **same session** even though they straddle the hour boundary.

**Expected Output Columns:**
- `user_id` (integer)
- `session_id` (integer) — sequential session number per user (1, 2, 3...)
- `session_start` (timestamp) — first event in the session
- `session_end` (timestamp) — last event in the session
- `event_count` (bigint) — number of events in the session

**Requirements:**
- Use LAG to get the previous event time per user
- A new session starts when the gap to the previous event exceeds 60 minutes (or it's the user's first event)
- Use `SUM() OVER` to create a session group key (cumulative count of session-starts)
- Then GROUP BY the session key to produce one row per session
- Order by `user_id ASC`, `session_start ASC`

**Difficulty Rating:** 5/5

WITH events (user_id, event_time) AS (
    VALUES
    (1, '2024-01-01 08:00'::timestamp),
    (1, '2024-01-01 08:30'::timestamp),
    (1, '2024-01-01 08:58'::timestamp),
    (1, '2024-01-01 09:03'::timestamp),
    (1, '2024-01-01 10:30'::timestamp),
    (1, '2024-01-01 11:45'::timestamp),
    (2, '2024-01-01 09:00'::timestamp),
    (2, '2024-01-01 09:50'::timestamp),
    (2, '2024-01-01 11:00'::timestamp)
),
users_prev_events AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY user_id ORDER BY event_time) AS session_id,
	LAG(event_time) OVER (PARTITION BY user_id) AS prev_event_time
FROM events
),
users_events_new_sessions AS (
SELECT 
	*,
	CASE
		WHEN prev_event_time IS NULL OR event_time - prev_event_time > INTERVAL '60 minutes' THEN 1
		ELSE 0
	END AS is_new_session
FROM users_prev_events
),
users_events_session_keys AS (
SELECT 
	*,
	SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY event_time) AS session_key
FROM users_events_new_sessions
)
SELECT
	user_id,
	session_key,
	MIN(event_time) AS session_start,
	MAX(event_time) AS session_end,
	COUNT(*) AS event_count
FROM users_events_session_keys
GROUP BY user_id, session_key
ORDER BY user_id, session_start
	

Feels great and WE DEFINITELY NEED TO PRACTICE THIS PATTERN IN ALL DIFFERENT CONTEXTS

---

## Task 3: PIVOT — Order Count by Delivery Status per Month

**Scenario:**
Build a pivot showing how many orders were in each delivery status per month.

**Expected Output Columns:**
- `month` (date) — truncated to month
- `pending` (bigint)
- `delivered` (bigint)
- `total_orders_with_delivery` (bigint) — total across all statuses

**Requirements:**
- Use `orders` and `deliveries` tables
- Join on `order_id`
- Use `COUNT(*) FILTER (WHERE status = '...')` pattern
- Order by `month ASC`

**Difficulty Rating:** 3/5

WITH orders_deliveries_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', d.created_at) AS delivery_month
FROM crappy_data_db.orders o
JOIN crappy_data_db.deliveries d ON o.id = d.order_id
)
SELECT 
	delivery_month,
	COUNT(*) FILTER (WHERE status = 'pending') AS pending,
	COUNT(*) FILTER (WHERE status = 'delivered') AS delivered,
	COUNT(DISTINCT(order_id)) AS total_orders_with_delivery
FROM orders_deliveries_months
GROUP BY delivery_month
ORDER BY delivery_month

I've added DISTINCT here for total_orders_with_delivery, as otherwise we'd simply get duplicated orders as their status changes since there are many more than just these two statuses.


---

## Submission Instructions

1. Task 1 — Self-referencing CTE with non-hardcoded anchor (2/5)
2. Task 2 — Time-proximity session grouping (5/5)
3. Task 3 — Delivery status PIVOT by month (3/5)
