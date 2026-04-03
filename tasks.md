# Daily SQL Practice Tasks

**Generated:** 2026-04-03
**Week 16, Day 5 Focus:** Time-proximity gaps-and-islands (session detection) + STDDEV anomaly scoring + Anti-join with NULL trap

---

## Task 1: Time-Proximity Gaps-and-Islands — User Session Detection (5/5)

**Scenario:**
The analytics team wants to group a user's daily sessions into "activity bursts" — consecutive days where the user had at least one session, with no gap of more than 2 days between them. Each burst should get a unique burst ID per user, and the team wants to see how long each burst lasted (in days) and the total sessions within it.

**Expected Output Columns:**
- `user_id` (integer)
- `burst_id` (integer) — sequential burst number per user (1, 2, 3…)
- `burst_start` (date) — first date of the burst
- `burst_end` (date) — last date of the burst
- `burst_days` (integer) — number of calendar days from start to end inclusive
- `total_sessions` (integer) — sum of count_sessions across all days in this burst

**Requirements:**
- Use `user_sessions_daily`
- Only include days where `count_sessions > 0`
- A new burst begins when the gap from the previous active day (for that user) exceeds 2 days
- Use LAG to detect gap, SUM OVER to build burst key, then GROUP BY to collapse
- Order by `user_id ASC`, `burst_start ASC`

**Hint:** The pattern is: LAG to get previous active date → compare gap → flag new burst → cumulative SUM of flags = burst group key → GROUP BY that key.

**Difficulty Rating:** 5/5

WITH user_sessions_lag AS (
SELECT 
	*,
	LAG(date) OVER (PARTITION BY user_id ORDER BY date) AS prev_date
FROM crappy_data_db.user_sessions_daily usd
),
streak_is_start AS (
SELECT 
	*,
	date - prev_date,
	CASE WHEN prev_date IS NULL OR date - prev_date <= 2 THEN 0 ELSE 1 END AS is_start
FROM user_sessions_lag
),
sessions_streak_ids AS (
SELECT 
	*,
	SUM(is_start) OVER (PARTITION BY user_id ORDER BY date) AS streak_id
FROM streak_is_start
)
SELECT 
	user_id,
	streak_id,
	MIN(date) AS streak_start,
	MAX(date) AS streak_end,
	COUNT(*) AS streak_days,
	SUM(count_sessions) AS total_sessions
FROM sessions_streak_ids
GROUP BY user_id, streak_id
ORDER BY user_id, streak_start

I changed column names as streak is more natural than burst.

---

## Task 2: STDDEV-Based Outlier Scoring — Transaction Volatility per User

**Scenario:**
The risk team wants to measure how volatile each user's transaction amounts are, and identify which individual transactions are statistical outliers (more than 2 standard deviations from the user's mean). They want a z-score for each transaction so analysts can rank transactions by how unusual they are.

**Expected Output Columns:**
- `id` (integer) — transaction id
- `user_id` (integer)
- `amount` (numeric)
- `user_avg` (numeric) — user's average transaction amount, rounded to 2 decimals
- `user_stddev` (numeric) — user's stddev of transaction amounts, rounded to 2 decimals
- `z_score` (numeric) — `(amount - user_avg) / NULLIF(user_stddev, 0)`, rounded to 2 decimals
- `is_outlier` (boolean) — true if ABS(z_score) > 2.0

**Requirements:**
- Use `transactions`, exclude NULL amounts and NULL user_ids
- Only include users with at least 5 transactions (to make stddev meaningful)
- NULLIF(user_stddev, 0) prevents division by zero for users with identical amounts
- Order by `ABS(z_score) DESC NULLS LAST`

**Difficulty Rating:** 4/5


WITH transactions_avg_std AS (
SELECT 
	*,
	ROUND(AVG(t.amount) OVER (PARTITION BY t.user_id), 2) AS user_avg,
	ROUND(STDDEV(t.amount) OVER (PARTITION BY t.user_id), 2) AS user_std
FROM crappy_data_db.transactions t 
),
transactions_cnt_users AS (
SELECT t.user_id, 
COUNT(*) AS transactions_cnt
FROM crappy_data_db.transactions t
GROUP BY t.user_id
)
SELECT 
	tas.id,
	tas.user_id,
	tas.amount,
	tas.user_avg,
	tas.user_std AS user_stddev,
	ABS(ROUND((tas.amount - tas.user_avg) / NULLIF(tas.user_std, 0), 2)) AS z_score,
	ABS(ROUND((tas.amount - tas.user_avg) / NULLIF(tas.user_std, 0), 2)) > 2.0 AS is_outlier
FROM transactions_avg_std tas
JOIN transactions_cnt_users tcu ON tas.user_id = tcu.user_id AND tcu.transactions_cnt >= 5
ORDER BY z_score DESC NULLS LAST


---

## Task 3: Anti-Join — Users Who Ordered But Never Transacted

**Scenario:**
The finance reconciliation team suspects there are users who placed orders but have no corresponding transactions on record. Find all users who have at least one order but zero transactions — using all three anti-join approaches: NOT IN, NOT EXISTS, and LEFT JOIN ... WHERE IS NULL.

**Expected Output Columns (for each approach):**
- `user_id` (integer)

**Requirements:**
- Use `orders` and `transactions` tables
- Write three separate queries producing the same result:
  1. Using `NOT IN` — then add a comment explaining when this breaks
  2. Using `NOT EXISTS`
  3. Using `LEFT JOIN ... WHERE IS NULL`
- For the NOT IN version: add a SQL comment explaining the NULL trap (what happens if any `user_id` in `transactions` is NULL, and why NOT IN silently returns 0 rows)
- Order by `user_id ASC` in all three

**Difficulty Rating:** 3/5

1. SELECT DISTINCT o.user_id
FROM crappy_data_db.orders o
WHERE o.user_id NOT IN
(SELECT t.user_id 
FROM crappy_data_db.transactions t
WHERE t.user_id IS NOT NULL
)

IT DOES NOT BREAK, AS I USED 'IS NOT NULL' to prevent from breaking :)).


2.

SELECT DISTINCT o.user_id
FROM crappy_data_db.orders o
WHERE NOT EXISTS
(SELECT t.user_id 
FROM crappy_data_db.transactions t
WHERE t.user_id  = o.user_id
)

3. 
SELECT DISTINCT o.user_id
FROM crappy_data_db.orders o
LEFT JOIN crappy_data_db.transactions t ON o.user_id = t.user_id
WHERE t.id IS NULL

I'm not a fan of this pattern though.

Got 35 rows in EVERY single query.



---

## Submission Instructions

1. Task 1 — Time-proximity burst detection with LAG + cumulative SUM (5/5)
2. Task 2 — STDDEV z-score outlier scoring with NULLIF guard (4/5)
3. Task 3 — Anti-join triple: NOT IN / NOT EXISTS / LEFT JOIN IS NULL (3/5)
