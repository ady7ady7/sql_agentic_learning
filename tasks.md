# Daily SQL Practice Tasks

**Generated:** 2026-03-20
**Week 14, Day 5 Focus:** Time-Proximity Drill + PIVOT Complex + Self-Referencing CTE Reinforcement

---

## Task 1: Time-Proximity Gaps-and-Islands — Chat Message Bursts

**Scenario:**
The support team wants to group chat messages into "conversation bursts" — clusters of messages within the same ticket where consecutive messages arrive within 10 minutes of each other. A gap > 10 minutes means a new burst starts.

For each burst, show the ticket_id, burst number, start/end time, message count, and how many unique senders were involved.

**Expected Output Columns:**
- `ticket_id` (bigint)
- `burst_id` (bigint) — sequential burst number per ticket (1, 2, 3...)
- `burst_start` (timestamp)
- `burst_end` (timestamp)
- `message_count` (bigint)
- `unique_senders` (bigint) — distinct non-NULL user_ids in the burst

**Requirements:**
- Use `chat_messages` table
- Gap threshold: 10 minutes between consecutive messages within the same ticket
- Use LAG → is_new_burst flag → SUM() OVER burst_key → GROUP BY pattern
- `burst_id` should be sequential per ticket using RANK() or DENSE_RANK()
- Only include bursts with at least 2 messages
- Order by `ticket_id ASC`, `burst_start ASC`

**Difficulty Rating:** 4/5


WITH ticket_msgs AS (
SELECT 
	*,
	LAG(cm.created_at) OVER (PARTITION BY cm.ticket_id ORDER BY cm.created_at) AS prev_msg_time
FROM crappy_data_db.chat_messages cm
WHERE cm.message_type = 'text'
),
ticket_burst_starts AS (
SELECT 
	*,
	CASE WHEN prev_msg_time IS NULL OR created_at - prev_msg_time > INTERVAL '10 Minutes' THEN 1 ELSE 0
	END AS burst_start
FROM ticket_msgs
),
ticket_burst_ids AS (
SELECT 
	*,
	SUM(burst_start) OVER (PARTITION BY ticket_id ORDER BY created_at) AS burst_id
FROM ticket_burst_starts
),
ticket_msg_bursts AS (
SELECT
	ticket_id,
	burst_id,
	MIN(created_at) AS burst_start,
	MAX(created_at) AS burst_end,
	COUNT(*) AS message_count,
	COUNT(DISTINCT(user_id, author_id)) AS unique_senders
FROM ticket_burst_ids
GROUP BY ticket_id, burst_id
)
SELECT * FROM ticket_msg_bursts
WHERE message_count > 2


I've checked and sorting is also on point.

---

## Task 2: Self-Referencing CTE — Find All Subordinates of a Manager

**Scenario:**
Using the same employee inline data as before, write a query that finds **all direct and indirect subordinates** of a given manager — for example, all employees who report (directly or indirectly) to employee id = 2 (Violet Green).

Use this data:

```sql
WITH RECURSIVE employees (id, first_name, last_name, manager_id) AS (
    VALUES
    (1, 'Madeline', 'Ray',     NULL::int),
    (2, 'Violet',   'Green',   1),
    (3, 'Alton',    'Vasquez', 1),
    (4, 'Geoffrey', 'Delgado', 1),
    (5, 'Allen',    'Garcia',  2),
    (6, 'Marian',   'Daniels', 2),
    (7, 'Tricia',   'Wong',    3),
    (8, 'Bruce',    'Grant',   3),
    (9, 'Darin',    'Burke',   4),
    (10,'Bob',      'Freeman', 5)
)
```

**Expected Output Columns:**
- `id` (integer)
- `first_name` (text)
- `last_name` (text)
- `depth` (integer) — levels below the starting manager (1 = direct report)
- `path` (text) — from the starting manager down

**Requirements:**
- Anchor: start with direct reports of manager_id = 2 (not the manager themselves)
- Recursive: keep joining employees to the CTE on manager_id = cte.id
- Natural termination
- Order by `path ASC`

**Difficulty Rating:** 3/5


WITH RECURSIVE employees (id, first_name, last_name, manager_id) AS (
    VALUES
    (1, 'Madeline', 'Ray',     NULL::int),
    (2, 'Violet',   'Green',   1),
    (3, 'Alton',    'Vasquez', 1),
    (4, 'Geoffrey', 'Delgado', 1),
    (5, 'Allen',    'Garcia',  2),
    (6, 'Marian',   'Daniels', 2),
    (7, 'Tricia',   'Wong',    3),
    (8, 'Bruce',    'Grant',   3),
    (9, 'Darin',    'Burke',   4),
    (10,'Bob',      'Freeman', 5)
),
hierarchy AS (
SELECT 
	id,
	first_name,
	last_name,
	manager_id,
	1 AS DEPTH,
	manager_id || '->' || id AS path
FROM employees
WHERE manager_id = 2
UNION ALL
SELECT
	e.id,
	e.first_name,
	e.last_name,
	h.id,
	h.DEPTH + 1,
	h.PATH || '->' || e.id
FROM hierarchy h
JOIN employees e ON h.id = e.manager_id
)
SELECT * FROM hierarchy




---

## Task 3: PIVOT — Support Ticket Priority Distribution by Status

**Scenario:**
The support team wants a cross-tab of ticket priorities across statuses — how many tickets of each priority exist for each status.

**Expected Output Columns:**
- `status` (text)
- `low` (bigint)
- `medium` (bigint)
- `high` (bigint)
- `urgent` (bigint)
- `total` (bigint)

**Requirements:**
- Use `chat_tickets` table
- Use `COUNT(*) FILTER (WHERE priority = '...')` pattern
- Order by `status ASC`

**Difficulty Rating:** 3/5

SELECT 
	status,
	COUNT(*) FILTER (WHERE priority = 'low') AS low,
	COUNT(*) FILTER (WHERE priority = 'medium') AS medium,
	COUNT(*) FILTER (WHERE priority = 'high') AS high,
	COUNT(*) FILTER (WHERE priority = 'urgent') AS urgent,
	COUNT(*) AS total
FROM crappy_data_db.chat_tickets
GROUP BY status
ORDER BY status

Wow, this is so easy and useful - everything in such a short query. Definitely more of this in more complex scenarios

---

## Submission Instructions

1. Task 1 — Chat message burst clustering (4/5)
2. Task 2 — All subordinates of a manager (3/5)
3. Task 3 — Ticket priority PIVOT by status (3/5)
