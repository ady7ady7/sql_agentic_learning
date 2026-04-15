# Daily SQL Practice Tasks

**Generated:** 2026-04-15
**Week 18, Day 3 Focus:** Funnel analysis (5/5) + gaps-and-islands monthly streaks + PERCENT_RANK with conditional bucket

---

## Task 1: Funnel Analysis — Registration → Order → Transaction (5/5)

**Scenario:**
The growth team wants to measure how many users complete each stage of the engagement funnel:
- **Stage 1**: Registered (all users)
- **Stage 2**: Placed at least one order
- **Stage 3**: Made at least one transaction

They want the absolute count at each stage and the drop-off rate from the previous stage.

**Expected Output Columns:**
- `stage` (integer) — 1, 2, or 3
- `stage_name` (text) — 'Registered', 'Ordered', 'Transacted'
- `user_count` (integer) — number of users reaching this stage
- `dropoff_pct` (numeric) — % of users lost vs previous stage, rounded to 1 decimal. Stage 1 = 0.0.

**Requirements:**
- Base population: all users from `users`
- Stage 2: users who have at least one row in `orders`
- Stage 3: users who have at least one row in `transactions`
- Use LEFT JOIN + COUNT DISTINCT to build each stage count — do NOT use subqueries in WHERE
- `dropoff_pct` = (prev_stage_count - current_stage_count) / prev_stage_count * 100
- Use LAG to compute dropoff from the previous stage row
- Final result: 3 rows, one per stage
- Order by `stage ASC`

**Difficulty Rating:** 5/5


WITH RECURSIVE registered_count AS (
SELECT
	'Registered' AS stage_name,
	COUNT(u.id) AS user_count
FROM crappy_data_db.users u
),
ordered_count AS (
SELECT 
	'Ordered' AS stage_name,
	COUNT(DISTINCT(o.user_id)) AS user_count
FROM crappy_data_db.orders o
),
transacted_count AS (
SELECT 
	'Transacted' AS stage_name,
	COUNT(DISTINCT(t.user_id)) AS user_count
FROM crappy_data_db.transactions t
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	rc.stage_name,
	rc.user_count,
	0.0 AS dropoff_pct
FROM registered_count rc
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(oc.stage_name, tc.stage_name),
	COALESCE(oc.user_count, tc.user_count),
	COALESCE(ROUND((1 - oc.user_count / h.user_count::NUMERIC) * 100, 1), ROUND((1 - Tc.user_count / h.user_count::NUMERIC) * 100, 1))
FROM HIERARCHY h
LEFT JOIN ordered_count oc ON h.LEVEL = 1
LEFT JOIN transacted_count tc ON h.LEVEL = 2
WHERE H.LEVEL < 3
)
SELECT * FROM hierarchy


I used recursive cte instead, a pattern we've practiced before. Imo it's very effective here.



---

## Task 2: Gaps-and-Islands — Monthly Order Streaks per User

**Scenario:**
The retention team wants to find users who placed orders in consecutive calendar months — and measure how long their longest ordering streak was (in months).

**Expected Output Columns:**
- `user_id` (integer)
- `streak_start` (date) — first month of the streak (DATE_TRUNC to month)
- `streak_end` (date) — last month of the streak
- `streak_length` (integer) — number of consecutive months in the streak

**Requirements:**
- Use `orders`, exclude NULL amounts
- Collapse to one row per user per month first (a user with 3 orders in the same month counts as 1 active month)
- Use the ROW_NUMBER subtraction trick to identify streak groups
- Only return each user's **longest** streak (if tied, return the earliest)
- Only include users with a streak of at least 2 months
- Order by `streak_length DESC`, `user_id ASC`

**Difficulty Rating:** 4/5


WITH orders_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month
FROM crappy_data_db.orders o
),
orders_prev_months AS (
SELECT 
	*,
	LAG(month) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_order_month
FROM orders_months
),
orders_new_streak_markers AS (
SELECT 
	*,
	CASE WHEN prev_order_month IS NULL OR prev_order_month - MONTH > INTERVAL '1' Month THEN 1 ELSE 0 END AS is_new_streak
FROM orders_prev_months
),
orders_streak_keys AS (
SELECT
	*,
	SUM(is_new_streak) OVER (PARTITION BY user_id ORDER BY created_at) AS streak_key
FROM orders_new_streak_markers 
),
users_streaks AS (
SELECT 
	user_id,
	MIN(month) AS streak_start,
	MAX(month) AS streak_end,
	COUNT(DISTINCT(month)) AS streak_length
FROM orders_streak_keys
GROUP BY user_id, streak_key
HAVING COUNT(DISTINCT(month)) >= 2
ORDER BY streak_length DESC, user_id
)
SELECT * FROM users_streaks

I used different pattern to identify patterns properly, IMO it's more universal and works in more contexts, but prove me wrong. I didn't read your instructions as I wanted to think myself, but I followed your objective and requirements, so you should credit me properly for that. Imo it's a better solution as well.

---

## Task 3: PERCENT_RANK — Transaction Amount Buckets with Percentile Labels

**Scenario:**
The analytics team wants to label each transaction with a human-readable percentile band based on amount — so analysts can quickly see if a transaction is in the bottom, middle, or top of the distribution.

**Expected Output Columns:**
- `id` (integer)
- `user_id` (integer)
- `amount` (numeric)
- `pct_rank` (numeric) — PERCENT_RANK() on amount, rounded to 3 decimals
- `band` (text) — 'bottom 25%' if pct_rank < 0.25, 'middle 50%' if < 0.75, 'top 25%' otherwise

**Requirements:**
- Use `transactions`, exclude NULL amounts and NULL user_ids
- PERCENT_RANK ordered by amount ASC (global, no partition)
- Order by `amount DESC`

**Difficulty Rating:** 3/5

WITH transactions_pct_rank AS (
SELECT 
	*,
	round(percent_rank() OVER (ORDER BY amount)::NUMERIC, 3) AS pct_rank
FROM crappy_data_db.transactions t 
WHERE t.amount IS NOT NULL AND t.user_id IS NOT NULL
)
SELECT 
	id,
	user_id,
	amount,
	pct_rank,
	CASE 
	WHEN pct_rank < 0.25 THEN 'bottom 25%' 
	WHEN pct_rank < 0.75 THEN 'middle 50%' ELSE 'top 25%' 
	END AS band
FROM transactions_pct_rank
ORDER BY amount DESC


---

## Submission Instructions

1. Task 1 — Funnel analysis: registered → ordered → transacted with dropoff % (5/5)
2. Task 2 — Monthly order streaks per user, longest streak only (4/5)
3. Task 3 — PERCENT_RANK transaction amount bands (3/5)
