# Daily SQL Practice Tasks

**Generated:** 2026-02-24
**Week 11, Day 2 Focus:** HackerRank Hard — Correlated Subqueries + Advanced Aggregations + Hierarchy

---

## Task 1: 3-Level Hierarchy — Chat Ticket Priorities and Top Tickets

**Scenario:**
Build a 3-level hierarchy over support tickets:
- Level 1: `'All Tickets'`
- Level 2: Distinct priority levels from `chat_tickets` (dynamic)
- Level 3: For each priority, the 3 most recently updated tickets (show ticket ID as text)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — priority at Level 2, ticket ID at Level 3
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Pre-aggregate distinct priorities and top-3-per-priority before the recursive CTE
- Use `updated_at DESC` to define "most recently updated"
- Termination condition required

**Difficulty Rating:** 3/5

WITH RECURSIVE priorities AS (
SELECT 
DISTINCT priority 
FROM chat_tickets
),
recently_updated_tickets AS (
SELECT
	priority,
	created_at,
	id,
	RANK() OVER (PARTITION BY priority ORDER BY created_at DESC) AS recent_update_rank
FROM chat_tickets
),
top3_recently_updated_tickets AS (
SELECT 
* 
FROM recently_updated_tickets
WHERE recent_update_rank <= 3
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Tickets' AS name,
	NULL::TEXT AS parent_name,
	'All Tickets' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(pr.priority, t3r.id::TEXT),
	h.name,
	h.PATH || ' > ' || COALESCE(pr.priority, t3r.id::TEXT)
FROM HIERARCHY h
LEFT JOIN priorities pr ON h.LEVEL = 1
LEFT JOIN top3_recently_updated_tickets t3r ON h.name = t3r.priority AND h.LEVEL = 2
WHERE h.LEVEL < 3
)
SELECT * FROM hierarchy


---

## Task 2: Order Value Outliers (Correlated Subquery)

**Scenario:**
The analytics team wants to flag orders that are significantly above average for that user — specifically, orders where the amount is more than **1.5x the user's own average order value**.

For each such outlier order, show:

**Expected Output Columns:**
- `order_id` (integer)
- `user_id` (integer)
- `order_amount` (numeric) — rounded to 2 decimals
- `user_avg_order_value` (numeric) — that user's average order amount, rounded to 2 decimals
- `ratio` (numeric) — order_amount / user_avg_order_value, rounded to 2 decimals

**Requirements:**
- Use `orders` table only
- Exclude NULL amounts
- A correlated subquery OR a CTE-based approach is both acceptable — choose what feels right
- Order by `ratio DESC`

**Difficulty Rating:** 4/5

WITH avg_users_orders AS (
SELECT
	user_id,
	ROUND(AVG(amount)::NUMERIC, 2) AS user_avg_order_value
FROM ORDERS
GROUP BY user_id
),
users_avg_orders_comparison AS (
SELECT 
	auo.user_id,
	auo.user_avg_order_value,
	o.id AS order_id,
	o.amount AS order_amount,
	ROUND(o.amount::NUMERIC / auo.user_avg_order_value, 2) AS ratio
FROM avg_users_orders auo
JOIN orders o ON auo.user_id = o.user_id
)
SELECT * FROM users_avg_orders_comparison
WHERE ratio > 1.5
ORDER BY ratio DESC

Please mind THAT excluding null amounts is pointless, as every order has an amount.

---

## Task 3: Message Response Time Analysis

**Scenario:**
The support team wants to understand how quickly agents respond to users within each ticket. For each ticket, find the first user message and the first agent response after it, then calculate the response time in minutes.

A user message has `author_id IS NULL` (sent by the client).
An agent message has `user_id IS NULL` (sent by the agent).

**Expected Output Columns:**
- `ticket_id` (bigint)
- `first_user_message_at` (timestamp)
- `first_agent_response_at` (timestamp) — first agent message AFTER the first user message, NULL if none
- `response_time_minutes` (numeric) — minutes between the two, rounded to 1 decimal, NULL if no response

**Requirements:**
- Use `chat_messages` table
- First user message = earliest message where `author_id IS NULL`
- First agent response = earliest message where `user_id IS NULL` AND `created_at > first_user_message_at`
- Order by `response_time_minutes ASC NULLS LAST`

**Difficulty Rating:** 5/5
WITH users_first_msg AS (
SELECT 
	ticket_id,
	MIN(created_at) AS first_user_message_at
FROM chat_messages
WHERE message_type = 'text' AND author_id IS NULL
GROUP BY ticket_id
),
agents_response_times AS (
SELECT 
	ticket_id,
	MIN(created_at) AS first_agent_response_at
FROM chat_messages
WHERE message_type = 'text' AND USER_ID IS NULL
GROUP BY ticket_id
)
SELECT 
	ufm.ticket_id,
	ufm.first_user_message_at,
	art.first_agent_response_at,
	EXTRACT('Minute' FROM art.first_agent_response_at - ufm.first_user_message_at) AS response_time_minutes
FROM users_first_msg ufm
JOIN agents_response_times art ON ufm.ticket_id = art.ticket_id
ORDER BY response_time_minutes ASC


Again, there were no nulls SO I OMITTED the NULLS LAST in order by, as it was pointless.

---

## Submission Instructions

1. Task 1 — Chat ticket priority hierarchy (3/5)
2. Task 2 — Order value outliers (4/5)
3. Task 3 — Message response time analysis (5/5)
