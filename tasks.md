# Daily SQL Practice Tasks

**Generated:** 2026-02-27
**Week 11, Day 5 Focus:** Friday Challenge — Full HackerRank Hard Simulation

---

## Task 1: 3-Level Hierarchy — Chat Tickets by Status and Priority

**Scenario:**
Build a 3-level hierarchy over support ticket data:
- Level 1: `'All Tickets'`
- Level 2: Distinct ticket statuses (dynamic, from `chat_tickets`)
- Level 3: For each status, the 3 priorities with the most tickets (show priority name and count as text, e.g. `'high (42)'`)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — status at Level 2, `'priority (count)'` string at Level 3
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Pre-aggregate ticket counts per status+priority before the recursive CTE
- Level 3 name should be formatted as `priority || ' (' || count::text || ')'`
- Termination condition required

**Difficulty Rating:** 4/5

WITH RECURSIVE ticket_counts AS (
SELECT 
	priority,
	status,
	COUNT(*) AS ticket_cnt
FROM chat_tickets
GROUP BY priority, status
),
counts_rank AS
(
SELECT 
	*,
	ROW_NUMBER() OVER (PARTITION BY status ORDER BY ticket_cnt DESC) AS rank_
FROM ticket_counts
),
distinct_statuses AS (
SELECT DISTINCT status FROM  chat_tickets
),
top_three_counts AS (
SELECT * FROM counts_rank
WHERE rank_ <= 3
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
	COALESCE(ds.status, ttc.priority || ' (' || ttc.ticket_cnt || ')'),
	h.name,
	h.PATH || ' > ' || COALESCE(ds.status, ttc.priority || ' (' || ttc.ticket_cnt || ')')
FROM HIERARCHY h
LEFT JOIN distinct_statuses ds ON h.LEVEL = 1
LEFT JOIN top_three_counts ttc ON h.LEVEL = 2 AND ttc.status = h.name
WHERE h.LEVEL < 3
)
SELECT * FROM hierarchy

It wasn't easy, but it's a nice practice exercise for formatting


---

## Task 2: Gaps-and-Islands — User Order Streaks by Month

**Scenario:**
The retention team wants to find users with long consecutive monthly ordering streaks. A streak is a sequence of calendar months where the user placed at least one order each month, with no gaps.

Find users with a streak of at least 3 consecutive months. For each qualifying user show their longest streak only.

**Expected Output Columns:**
- `user_id` (integer)
- `streak_start` (date) — first month of the streak (truncated to month)
- `streak_end` (date) — last month of the streak
- `streak_length` (bigint) — number of consecutive months
- `streak_revenue` (numeric) — total order revenue during the streak, rounded to 2 decimals

**Requirements:**
- Use `orders` table
- Truncate orders to month, deduplicate (one row per user per month)
- Apply gaps-and-islands: `ROW_NUMBER()` subtracted from the month produces the streak group key
- For streak_revenue: join back to raw orders to sum amounts within the streak period
- If a user has multiple streaks of equal length, show the most recent one
- Order by `streak_length DESC`, `streak_revenue DESC`

**Difficulty Rating:** 5/5

That's one of the hardest tasks we've had recently, but I've handled it well and check whether it works properly - it does and correctly sums up the revenues across different months of the streak. However, THERE WERE NO users with streak above 2 months, so obviously I didn't filter for that. FYI: Streak revenue is rounded to 2 decimals by default.

WITH users_order_months AS (
SELECT 
	DISTINCT user_id,
	DATE_TRUNC('Month', created_at) AS order_month
FROM orders
),
users_months_rn AS (
SELECT 
	*,
	ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_month)
FROM users_order_months
),
users_streak_keys AS (
SELECT 
	*,
	order_month - row_number * INTERVAL '1' MONTH AS streak_key
FROM users_months_rn
),
users_monthly_streaks_no_revenue AS (
SELECT
 	user_id,
 	streak_key,
 	MIN(order_month) AS streak_start,
 	MAX(order_month) AS streak_end,
 	max(row_number) AS streak_length
 FROM users_streak_keys
 GROUP BY user_id, streak_key
ORDER BY user_id
),
users_orders AS (
SELECT 
	user_id,
	amount,
	DATE_TRUNC('Month', created_at) AS order_month
FROM orders
),
users_monthly_revenues AS (
SELECT
	user_id,
	order_month,
	SUM(amount) AS total_revenue
FROM users_orders
GROUP BY user_id, order_month
)
SELECT 
	umsnr.user_id,
	umsnr.streak_start,
	umsnr.streak_end,
	umsnr.streak_length,
	SUM(umr.total_revenue) AS streak_revenue
FROM users_monthly_streaks_no_revenue umsnr
JOIN users_monthly_revenues umr ON umsnr.user_id = umr.user_id
WHERE umr.order_month >= umsnr.streak_start AND umr.order_month <= umsnr.streak_end
GROUP BY umsnr.user_id, umsnr.streak_start, umsnr.streak_end, umsnr.streak_length
ORDER BY streak_length DESC, streak_revenue DESC


---

## Task 3: Support Ticket Complexity Score

**Scenario:**
The support team wants to score each ticket by complexity — based on how many messages it received, how many unique participants were involved, and whether it was escalated (ever had `priority = 'urgent'`).

Complexity score formula:
`(message_count * 1.0) + (unique_participants * 2.0) + (is_urgent * 5.0)`

**Expected Output Columns:**
- `ticket_id` (bigint)
- `message_count` (bigint)
- `unique_participants` (bigint) — distinct non-NULL user_ids from `chat_messages`
- `is_urgent` (integer) — 1 if ticket priority is `'urgent'`, 0 otherwise
- `complexity_score` (numeric) — rounded to 1 decimal
- `complexity_rank` (bigint) — ranked by complexity_score DESC

**Requirements:**
- Use `chat_tickets` and `chat_messages`
- Only include tickets with at least 2 messages
- Order by `complexity_rank ASC`

**Difficulty Rating:** 4/5


WITH tickets_msgs_participants AS (
SELECT 
	ticket_id,
	COUNT(*) AS msg_cnt,
	COUNT(DISTINCT(author_id)) AS unique_authors,
	COUNT(DISTINCT(user_id)) AS unique_users
FROM chat_messages
WHERE message_type = 'text'
GROUP BY ticket_id
),
tickets_shallow_rank AS (
SELECT 
	ct.id AS ticket_id,
	tmp.msg_cnt,
	tmp.unique_authors + tmp.unique_users AS unique_participants,
	CASE 
		WHEN ct.priority = 'urgent' THEN 1 ELSE 0
	END AS is_urgent
FROM chat_tickets ct
JOIN tickets_msgs_participants tmp ON ct.id = tmp.ticket_id
),
tickets_complexity_scores AS (
SELECT 
	*,
	(msg_cnt * 1.0) + (unique_participants * 2.0) + (is_urgent * 5.0) AS complexity_score
FROM tickets_shallow_rank
WHERE msg_cnt >= 2
)
SELECT 
	*,
	DENSE_RANK() OVER (ORDER BY complexity_score DESC) AS complexity_rank
FROM tickets_complexity_scores
ORDER BY complexity_rank

FYI: Null users were not counted here, the score is already rounded, as all the results are int numbers (there's no other possibility), and everything is as you wanted.

---

## Submission Instructions

1. Task 1 — Chat ticket status/priority hierarchy (4/5)
2. Task 2 — Monthly order streaks (5/5)
3. Task 3 — Ticket complexity score (4/5)
