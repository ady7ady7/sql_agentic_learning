# SQL Tasks — Week 32 Day 2

**Generated:** 2026-07-28
**Dataset:** crappy_data
**Focus:** Gaps-and-islands, running total with reset, percentile filter

---

## Task 1: User Activity Streaks
**Difficulty: 4/5**

**Business question:**
A "streak" is a consecutive sequence of days where a user had at least one session (count_sessions > 0). A streak ends when there's a day with 0 sessions or a gap in dates. For each user, find their longest streak (in days) and the start and end date of that streak.

**Expected output columns:**
`user_id, longest_streak_days, streak_start, streak_end`



WITH users_sessions AS (
	SELECT 
		*,
		LAG(date) OVER (PARTITION BY user_id ORDER BY date) AS prev_session_date,
		date - LAG(date) OVER (PARTITION BY user_id ORDER BY date) AS days_diff
	FROM crappy_data_db.user_sessions_daily usd 
),
users_streak_keys AS (
SELECT 
	*,
	CASE WHEN days_diff IS NULL OR days_diff = 1 THEN 0 ELSE 1 END AS streak_key
FROM users_sessions
),
users_streak_ids AS (
SELECT 
	*,
	SUM(streak_key) OVER (PARTITION BY user_id ORDER BY date) AS streak_id
FROM users_streak_keys 
),
users_streaks AS (
SELECT 
	user_id,
	streak_id,
	COUNT(*) AS streak_days,
	MIN(date) AS streak_start,
	MAX(date) AS streak_end
FROM users_streak_ids
GROUP BY user_id, streak_id
),
longest_streaks AS (
SELECT 
	user_id,
	MAX(streak_days) AS longest_streak_days
FROM users_streaks
GROUP BY user_id
)
SELECT DISTINCT ON (l.user_id)
	l.user_id,
	l.longest_streak_days,
	u.streak_start,
	u.streak_end
FROM longest_streaks l
JOIN users_streaks u ON l.user_id = u.user_id AND u.streak_days = l.longest_streak_days
ORDER BY l.user_id

Not sure if this is not overengineered, but it works just as you expected.


---

## Task 2: Running Transaction Total with Reset on Withdrawal
**Difficulty: 5/5**

**Business question:**
For each user, compute a running total of transaction amounts — but reset the counter to 0 every time a `withdrawal` transaction appears. Show every transaction row with its running total at that point.

**Expected output columns:**
`id, user_id, created_at, type, amount, running_total`

Actually not that difficult tbh, works just fine :)).

WITH users_transactions AS (
SELECT 
	*,
	CASE WHEN TYPE = 'withdrawal' THEN 1 ELSE 0 END AS is_withdrawal
FROM crappy_data_db.transactions t
),
users_withdrawals AS (
SELECT 
	*,
	SUM(is_withdrawal) OVER (PARTITION BY user_id ORDER BY created_at) AS withdrawal_id
FROM users_transactions
)
SELECT 
	id,
	user_id,
	created_at,
	TYPE,
	amount,
	SUM(amount) OVER (PARTITION BY user_id, withdrawal_id ORDER BY created_at) AS running_total
FROM users_withdrawals



---

## Task 3: Users Above 75th Percentile Average Order Value
**Difficulty: 3/5**

**Business question:**
Find all users whose average order amount is above the 75th percentile of average order amounts across all users. Do not use a subquery in the WHERE clause.

**Expected output columns:**
`user_id, avg_order_amount`

dONE, EASY ASF.
I could do it with PERCENTILE_CONT as well, but I decided to do the perecentile_rank


WITH users_avg_orders AS (
SELECT 
	o.user_id,
	ROUND(AVG(o.amount)::NUMERIC, 2) AS avg_order_amt
FROM crappy_data_db.orders o
GROUP BY o.user_id
),
users_percentile_ranks AS (
SELECT 
	*,
	percent_rank() OVER (ORDER BY avg_order_amt) AS percentile_rank
FROM users_avg_orders
)
SELECT * FROM users_percentile_ranks
WHERE percentile_rank >= 0.75

---

## Submission Instructions

Paste your queries and results below each task.
