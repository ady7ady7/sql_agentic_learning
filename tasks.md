# Daily SQL Practice Tasks

**Generated:** 2026-02-13
**Week 9, Day 5 Focus:** Hierarchy + HackerRank-Style Multi-CTE Puzzles

---

## Task 1: 3-Level Hierarchy — Delivery Statuses from Real Data

**Scenario:**
Build a 3-level hierarchy where Level 2 comes from **real data** (not hardcoded):
- Level 1: 'All Deliveries'
- Level 2: Distinct delivery statuses pulled from the `deliveries` table (e.g., 'pending', 'delivered', etc.)
- Level 3: Specific order IDs for each status, limited to the top 3 orders by amount per status

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — status name or order identifier
- `parent_name` (text)
- `path` (text) — full path from root

**Requirements:**
- Level 2 nodes should be dynamically pulled from `deliveries.status` (no hardcoded VALUES)
- Level 3 should show actual order IDs (as text), joined via `deliveries.order_id` → `orders`
- Limit Level 3 to top 3 orders per status by `orders.amount` DESC
- Include the path column showing the full hierarchy trail

**Difficulty Rating:** 4/5

WITH RECURSIVE HIERARCHY AS (
SELECT
	1 AS LEVEL,
	NULL::TEXT AS id,
	'All Deliveries'::TEXT AS name,
	NULL::TEXT AS amount,
	NULL::TEXT AS parent_name,
	'All Deliveries'::TEXT AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	d.order_id::TEXT,
	COALESCE(d.status, o.id::TEXT)::TEXT,
	COALESCE(NULL, o.amount)::TEXT,
	h.name,
	h.PATH || ' > ' || h.name
FROM HIERARCHY h
LEFT JOIN deliveries d ON h.LEVEL = 1
LEFT JOIN orders o ON (o.id)::TEXT = h.id AND h.LEVEL = 2 
)
SELECT 
	LEVEL,
	name,
	parent_name,
	path
FROM HIERARCHY


I tried to add amount ordering, but the queries take forever and I just can't wait anymore at some point and cancel it. I'm not sure why that's the case, but it's annoying

---

## Task 2: Pareto Analysis — Revenue Concentration (80/20 Rule)

**Scenario:**
The business wants to understand revenue concentration. Apply the Pareto Principle: identify which users contribute to the top 80% of total revenue, and label them as "Key Accounts" vs "Standard Accounts."

**Expected Output Columns:**
- `user_id` (integer)
- `total_revenue` (numeric) — sum of all order amounts for this user
- `revenue_rank` (integer) — rank by total revenue descending
- `revenue_share_pct` (numeric) — this user's share of overall revenue, rounded to 2 decimals
- `cumulative_share_pct` (numeric) — running cumulative % of total revenue, rounded to 2 decimals
- `account_type` (text) — 'Key Account' if within top 80% cumulative, 'Standard Account' otherwise

**Requirements:**
- Use `orders` table
- Calculate each user's total revenue and their percentage share of overall revenue
- Calculate a running cumulative share (ordered by revenue DESC)
- Label users whose cumulative share is still within 80% as 'Key Account'
- Order by revenue_rank ASC

**Difficulty Rating:** 5/5


WITH users_revenues AS (
SELECT 
	user_id,
	SUM(amount) AS total_user_revenue,
	(SELECT ROUND(SUM(amount)::NUMERIC, 2) FROM orders) AS total_revenue
FROM orders
GROUP BY user_id
),
users_rev_rank AS (
SELECT 
	*,
	RANK() OVER (ORDER BY total_user_revenue DESC) AS revenue_rank,
	ROUND(total_user_revenue::NUMERIC / total_revenue::NUMERIC * 100, 2) AS revenue_share_pct
FROM users_revenues
)
SELECT 
	*,
	SUM(revenue_share_pct) OVER (ORDER BY revenue_share_pct DESC) AS cumulative_share_pct,
	CASE WHEN (SUM(revenue_share_pct) OVER (ORDER BY revenue_share_pct DESC)) < 80 THEN 'Key Account' ELSE 'Standard Account' END AS account_type
FROM users_rev_rank


---

## Task 3: Support Ticket Complexity Scoring

**Scenario:**
The support team wants to identify complex tickets. Score each ticket based on message count and conversation duration, then rank them within each priority level.

**Expected Output Columns:**
- `ticket_id` (bigint)
- `priority` (text)
- `status` (text) — ticket status
- `message_count` (bigint) — total messages in the ticket
- `duration_hours` (numeric) — hours between first and last message, rounded to 1 decimal
- `complexity_score` (numeric) — `message_count * (duration_hours + 1)`, rounded to 1 decimal
- `priority_rank` (integer) — rank within priority level by complexity_score DESC

**Requirements:**
- Use `chat_tickets` and `chat_messages` tables
- Only count tickets that have at least 2 messages
- Duration = time between earliest and latest message in the ticket
- The `+1` in complexity formula prevents zero-duration tickets from scoring 0
- Order by priority, then priority_rank ASC

**Difficulty Rating:** 5/5

WITH tickets_priority_duration AS (
SELECT 
cm.ticket_id,
ct.priority,
MAX(cm.created_at) - MIN(cm.created_at) AS conversation_duration
FROM chat_tickets ct
JOIN chat_messages cm ON ct.id  = cm.ticket_id
GROUP BY cm.ticket_id, ct.priority 
),
tickets_msg_cnt AS (
SELECT 
	ticket_id,
	COUNT(id) AS messages_cnt
FROM chat_messages
WHERE message_type = 'text'
GROUP BY ticket_id
)
SELECT 
	tpd.ticket_id,
	tpd.priority,
	tpd.conversation_duration,
	tmc.messages_cnt,
	tmc.messages_cnt * EXTRACT('Minute' FROM tpd.conversation_duration) AS complexity_score,
	RANK() OVER (PARTITION BY tpd.priority ORDER BY (tmc.messages_cnt * EXTRACT('Minute' FROM tpd.conversation_duration)) DESC) AS rank
FROM tickets_priority_duration tpd
JOIN tickets_msg_cnt tmc ON tpd.ticket_id = tmc.ticket_id
ORDER BY priority

One caveat here - I extracted minutes AS almost all tickets were solved within one hour, so I didn't see a point in it - perhaps it's wrong - let me know, but my logic is definitelly flawless and makes a lot of sense in this context.

---

## Submission Instructions

1. Task 1 — Hierarchy with real-data Level 2 nodes (4/5)
2. Task 2 — Pareto / revenue concentration analysis (5/5)
3. Task 3 — Support ticket complexity scoring (5/5)
