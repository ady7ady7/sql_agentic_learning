# Daily SQL Practice Tasks

**Generated:** 2026-03-30
**Week 16, Day 1 Focus:** Type B Recursive CTE + Anti-Join NULL Trap + Time-Proximity Variant

---

## Task 1: Type B Recursive CTE — Full Referral Chain Traversal

**Scenario:**
Use this inline referral data to traverse the full referral tree — unlimited depth, natural termination:

```sql
WITH RECURSIVE referrals (id, name, referred_by) AS (
    VALUES
    (1,  'Alice',   NULL::int),
    (2,  'Bob',     1),
    (3,  'Carol',   1),
    (4,  'Dave',    2),
    (5,  'Eve',     2),
    (6,  'Frank',   3),
    (7,  'Grace',   4),
    (8,  'Hank',    4),
    (9,  'Ivy',     6),
    (10, 'Jack',    9)
)
```

For each person, show their full path from root and their depth level.

**Expected Output Columns:**
- `id` (integer)
- `name` (text)
- `referred_by` (integer)
- `depth` (integer) — 1 for root
- `path` (text) — e.g. `'Alice -> Bob -> Dave -> Grace'`

**Requirements:**
- Anchor: `WHERE referred_by IS NULL` — do NOT hardcode the root
- Recursive: `JOIN referrals ON referrals.referred_by = cte.id`
- Path separator: ` -> `
- Natural termination — no LEVEL limit
- Order by `path ASC`

**Difficulty Rating:** 3/5

WITH RECURSIVE referrals (id, name, referred_by) AS (
    VALUES
    (1,  'Alice',   NULL::int),
    (2,  'Bob',     1),
    (3,  'Carol',   1),
    (4,  'Dave',    2),
    (5,  'Eve',     2),
    (6,  'Frank',   3),
    (7,  'Grace',   4),
    (8,  'Hank',    4),
    (9,  'Ivy',     6),
    (10, 'Jack',    9)
),
HIERARCHY AS (
SELECT 
	*,
	1 AS DEPTH,
	'Alice' AS path
FROM referrals 
WHERE referred_by IS NULL
UNION ALL
SELECT 
	r.id,
	r.name,
	r.referred_by,
	h.DEPTH + 1,
	h.PATH || '->' || R.NAME
FROM HIERARCHY h JOIN referrals r ON h.id = r.referred_by
)
SELECT * FROM hierarchy


---

## Task 2: Anti-Join NULL Trap — The NOT IN Failure

**Scenario:**
This task demonstrates why `NOT IN` breaks silently when the subquery contains NULLs.

Use this inline data:

```sql
WITH all_users (user_id, name) AS (
    VALUES (1, 'Alice'), (2, 'Bob'), (3, 'Carol'), (4, 'Dave'), (5, 'Eve')
),
orders (order_id, user_id) AS (
    VALUES (101, 1), (102, 2), (103, NULL), (104, 4)
)
```

**Step A:** Write `NOT IN` to find users with no orders. Observe the result — it returns 0 rows despite user 3 (Carol) and user 5 (Eve) clearly having no orders.

WITH all_users (user_id, name) AS (
    VALUES (1, 'Alice'), (2, 'Bob'), (3, 'Carol'), (4, 'Dave'), (5, 'Eve')
),
orders (order_id, user_id) AS (
    VALUES (101, 1), (102, 2), (103, NULL), (104, 4)
)
SELECT DISTINCT user_id FROM all_users
WHERE user_id NOT IN (SELECT DISTINCT user_id FROM orders)

It shows no users, as you've said

BUT:

WITH all_users (user_id, name) AS (
    VALUES (1, 'Alice'), (2, 'Bob'), (3, 'Carol'), (4, 'Dave'), (5, 'Eve')
),
orders (order_id, user_id) AS (
    VALUES (101, 1), (102, 2), (103, NULL), (104, 4)
)
SELECT DISTINCT user_id FROM all_users
WHERE user_id NOT IN (SELECT DISTINCT user_id FROM orders WHERE user_id IS NOT NULL)

Now it correctly shows id 3 and 5, so this is just one caveat to remember, that we need IS NOT NULL here!


**Step B:** Write `NOT EXISTS` for the same question. Observe it returns Carol and Eve correctly.

WITH all_users (user_id, name) AS (
    VALUES (1, 'Alice'), (2, 'Bob'), (3, 'Carol'), (4, 'Dave'), (5, 'Eve')
),
orders (order_id, user_id) AS (
    VALUES (101, 1), (102, 2), (103, NULL), (104, 4)
)
SELECT DISTINCT au.user_id FROM all_users au
WHERE NOT EXISTS (SELECT DISTINCT user_id FROM orders WHERE user_id = au.user_id)


**Step C:** Write `LEFT JOIN ... WHERE IS NULL` for the same question.

WITH all_users (user_id, name) AS (
    VALUES (1, 'Alice'), (2, 'Bob'), (3, 'Carol'), (4, 'Dave'), (5, 'Eve')
),
orders (order_id, user_id) AS (
    VALUES (101, 1), (102, 2), (103, NULL), (104, 4)
)
SELECT DISTINCT au.user_id FROM all_users au
LEFT JOIN orders o ON au.user_id = o.user_id WHERE o.user_id IS NULL

But honestly, THIS solution seems the most unnatural to me. It looks weird and unnatural. I'd rather omit it in the future.


**Step D:** Explain in a comment WHY `NOT IN` fails here.

I already explained, that not necessarily it has to fail.
It fails because NULL in the subquery disqualifies all of user_ids from the original table.

**Expected Output** (Steps B and C):
- `user_id` (integer)
- `name` (text)

**Difficulty Rating:** 3/5

---

## Task 3: Time-Proximity — Support Ticket Message Bursts on Real Data

**Scenario:**
Group `chat_messages` per ticket into conversation bursts where consecutive messages (any type) are within **20 minutes** of each other. Find tickets with at least 3 bursts.

**Expected Output Columns:**
- `ticket_id` (bigint)
- `burst_count` (bigint) — number of bursts in this ticket
- `total_messages` (bigint)
- `first_message_at` (timestamp)
- `last_message_at` (timestamp)

**Requirements:**
- Use `chat_messages` table
- Include all message types (no filter on message_type)
- LAG → is_new_burst flag (gap > 20 min) → SUM() OVER → aggregate per ticket
- Only include tickets with burst_count >= 3
- Order by `burst_count DESC`, `total_messages DESC`

**Difficulty Rating:** 4/5

WITH ticket_messages AS (
SELECT 
	*,
	LAG(cm.created_at) OVER (PARTITION BY ticket_id ORDER BY created_at) AS prev_ticket_time
FROM crappy_data_db.chat_messages cm
WHERE cm.message_type = 'text'
),
ticket_streak_starts AS (
SELECT 
	*,
	CASE WHEN prev_ticket_time IS NULL OR created_at - prev_ticket_time > INTERVAL '20 Minutes' THEN 1 ELSE 0 END AS is_start
FROM ticket_messages
),
ticket_streak_ids AS (
SELECT 
	*,
	SUM(is_start) OVER (PARTITION BY ticket_id ORDER BY created_at) AS streak_id
FROM ticket_streak_starts
),
ticket_streaks AS (
SELECT 
	ticket_id,
	streak_id,
	COUNT(*) AS total_messages,
	MIN(created_at) AS first_message_at,
	MAX(created_at) AS last_message_at
FROM ticket_streak_ids
GROUP BY ticket_id, streak_id
)
SELECT 
	ticket_id,
	COUNT(*) AS streak_count,
	SUM(total_messages) AS total_messages,
	MIN(first_message_at) AS first_message_at,
	MAX(last_message_at) AS last_message_at
FROM ticket_streaks
GROUP BY ticket_id
ORDER BY streak_count DESC, total_messages DESC

This was a bit unintuitive.
We didn't HAVE A SINGLE ticket with more than 2 streaks, so I kept data the way it is.


---

## Submission Instructions

1. Task 1 — Type B recursive CTE referral chain (3/5)
2. Task 2 — NOT IN NULL trap — all four steps (3/5)
3. Task 3 — Chat message bursts on real data (4/5)
