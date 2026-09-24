# SQL Tasks — 2026-09-24 (Week 39, Day 4)

**Dataset:** nq_data.ticks · crappy_data_db  
**Focus:** DISTINCT ON (hourly extremes) · LAG + LEAD combined

---

## Task 1 — Extreme Price per RTH Hour (DISTINCT ON)

**Difficulty: 4/5**

**Business question:**  
For each RTH trading hour (grouping by trade_date and the hour portion of ET time, e.g. 10:00, 11:00, ...), find the tick with the highest price of that hour. Use `DISTINCT ON`.

Filter to RTH only, exclude `side = 'N'`.

**Expected output columns:**  
`trade_date, hour_bucket, ts_event, price, size`

Order by `trade_date`, `hour_bucket`.


SELECT DISTINCT ON ((ts_event AT TIME ZONE 'America/New_York')::date, DATE_TRUNC('Hour', ts_event AT TIME ZONE 'America/New_York'))
	(ts_event AT TIME ZONE 'America/New_York')::date AS trade_date,
	ts_event AT TIME ZONE 'America/New_York' AS et_time,
	DATE_TRUNC('Hour', ts_event AT TIME ZONE 'America/New_York') AS HOUR,
	price,
	SIZE,
	side
FROM nq_data.ticks t
WHERE side != 'N' 
AND (ts_event AT TIME ZONE 'America/New_York')::TIME >= '9:30'
AND (ts_event AT TIME ZONE 'America/New_York')::TIME < '16:00'
ORDER BY trade_date, HOUR, price DESC

No need to do anything else with hour, as it's still tied to it's date every time






---

## Task 2 — Local Peak Transactions (LAG + LEAD)

**Difficulty: 4/5**

**Business question:**  
For each user, identify transactions that are a "local peak" — meaning the transaction's `amount` is strictly greater than both the immediately preceding and immediately following transaction (by `created_at`) for that same user.

Only include users with at least 3 transactions (so a peak comparison is meaningful).

**Expected output columns:**  
`user_id, id, created_at, amount, is_local_peak`

`is_local_peak` is boolean. Order by `user_id`, `created_at`.




WITH users_local_transactions AS (
SELECT 
	*,
	LAG(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_t,
	LEAD(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS next_t
FROM crappy_data_db.transactions t
)
SELECT 
	user_id,
	id,
	created_at,
	amount,
	(amount > prev_t) AND (amount > next_t) AS is_local_peak
FROM users_local_transactions
WHERE prev_t IS NOT NULL AND next_t IS NOT NULL
ORDER BY user_id, created_at

tHE 3 TRANSACTIONS factor is sorted by IS NOT NULL, no need to do any more checks





---

## Submission Instructions

Paste your queries below each task.
