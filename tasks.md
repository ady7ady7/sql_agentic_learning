# Daily SQL Practice Tasks

**Generated:** 2026-03-24
**Week 15, Day 2 Focus:** Time-Proximity Variant + PIVOT + Self-Referencing CTE + Anti-Join Complex

---

## Task 1: Time-Proximity — Support Ticket Response Bursts

**Scenario:**
The support team wants to identify "response bursts" — periods within a ticket where messages arrive rapidly. Define a burst as a sequence of messages within the same ticket where each consecutive message arrives within **15 minutes** of the previous one.

For each burst show:

**Expected Output Columns:**
- `ticket_id` (bigint)
- `burst_id` (bigint) — sequential per ticket (1, 2, 3...)
- `burst_start` (timestamp)
- `burst_end` (timestamp)
- `message_count` (bigint)
- `duration_minutes` (numeric) — minutes from burst_start to burst_end, rounded to 1 decimal

**Requirements:**
- Use `chat_messages` table, `message_type = 'text'` only
- Gap threshold: 15 minutes between consecutive messages per ticket
- LAG → is_new_burst flag → SUM() OVER → GROUP BY pattern
- Only include bursts with at least 2 messages
- Order by `ticket_id ASC`, `burst_start ASC`

**Difficulty Rating:** 4/5

WITH tickets_msg_prev_responses AS (
SELECT 
	*,
	LAG(cm.created_at) OVER (PARTITION BY ticket_id ORDER BY created_at) AS prev_ticket_response
FROM crappy_data_db.chat_messages cm
WHERE message_type = 'text'
),
msgs_is_streaks AS (
SELECT 
	*,
	CASE WHEN prev_ticket_response IS NULL OR created_at - prev_ticket_response > INTERVAL '15 Minutes' THEN 1 ELSE 0 END AS is_new_streak
FROM tickets_msg_prev_responses
),
msgs_streak_ids AS (
SELECT 
	*,
	SUM(is_new_streak) OVER (PARTITION BY ticket_id ORDER BY created_at) AS streak_id
FROM msgs_is_streaks
)
SELECT
	ticket_id,
	streak_id,
	MIN(created_at) AS streak_start,
	MAX(created_at) AS streak_end,
	COUNT(*) AS message_count,
	EXTRACT('Minute' FROM MAX(created_at) - MIN(created_at)) AS duration_minutes
FROM msgs_streak_ids
GROUP BY ticket_id, streak_id
ORDER BY ticket_id, streak_start


I've changed the names from burst to streak, as it sounds more natural and matching here. DO NOT take away points for that.

---

## Task 2: Self-Referencing CTE — Find All Ancestors of a Node

**Scenario:**
Given the same category tree as before, write a query that finds **all ancestors** of a given node — traversing **upward** from child to parent, instead of the usual top-down direction.

Use this inline data:

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

Find all ancestors of node `id = 8` (iPhone). The result should include: Phones → Electronics → All Products.

**Expected Output Columns:**
- `id` (integer)
- `name` (text)
- `parent_id` (integer)
- `depth` (integer) — 1 = direct parent, 2 = grandparent, etc.

**Requirements:**
- Anchor: start with the direct parent of node 8 (`WHERE id = (SELECT parent_id FROM categories WHERE id = 8)`)
- Recursive: JOIN categories on `categories.id = cte.parent_id`
- Natural termination when parent_id IS NULL (root reached)
- Order by `depth ASC`

**Difficulty Rating:** 4/5

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
HIERARCHY AS (
SELECT 
	*,
	1 AS depth
FROM categories
WHERE id = 8
UNION ALL
SELECT 
c.id,
c.name,
c.parent_id,
h.DEPTH + 1
FROM HIERARCHY h
JOIN categories c ON h.parent_id = c.id
)
SELECT * FROM HIERARCHY
ORDER BY DEPTH

---

## Task 3: PIVOT + Anti-Join — Monthly Revenue from Users Who Never Had a Delivered Order

**Scenario:**
Combine what you've learned: first identify users who have never had a delivered order (anti-join), then pivot their monthly order revenue by the delivery status of their orders.

**Expected Output Columns:**
- `month` (date) — truncated to month
- `pending_revenue` (numeric) — rounded to 2 decimals
- `total_revenue` (numeric) — rounded to 2 decimals
- `user_count` (bigint) — distinct users contributing that month

**Requirements:**
- Use `users`, `orders`, `deliveries`
- First isolate users with no delivered orders using NOT EXISTS
- Then join to their orders and deliveries to pivot revenue by status
- Only include `status = 'pending'` for the pivot column (it will be the only status for these users)
- Order by `month ASC`

**Difficulty Rating:** 5/5

WITH users_without_delivered_orders AS (
SELECT o2.user_id 
FROM crappy_data_db.orders o2
WHERE NOT EXISTS (
SELECT 
	DISTINCT(user_id)
FROM crappy_data_db.deliveries d 
JOIN crappy_data_db.orders o ON d.order_id = o.id AND o.user_id = o2.user_id
WHERE d.status = 'delivered'
)
),
users_orders AS (
SELECT 
	uw.user_id AS USER,
	*,
	DATE_TRUNC('Month', o.created_at) AS month_
FROM users_without_delivered_orders uw
JOIN crappy_data_db.orders o ON uw.user_id = o.user_id
JOIN crappy_data_db.deliveries d ON o.id = d.order_id
)
SELECT 
	month_,
	SUM(amount) FILTER (WHERE status = 'pending') AS pending_revenue,
	SUM(amount) AS total_revenue,
	COUNT(DISTINCT(uo.user)) AS user_count
FROM users_orders uo
GROUP BY month_
ORDER BY month_

---

## Submission Instructions

1. Task 1 — Support ticket response bursts (4/5)
2. Task 2 — Self-referencing CTE traversal upward (4/5)
3. Task 3 — PIVOT + anti-join combined (5/5)
