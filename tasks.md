# SQL Tasks — 2026-09-08 (Week 37, Day 2)

**Dataset:** transactions / users · nq_data.ticks  
**Focus:** LATERAL (top N per group, new context) · Intraday cumulative VWAP (session-reset)

---

## Task 1 — Top 3 Spenders per City (LATERAL)

**Difficulty: 4/5**

**Business question:**  
For each city, find the top 3 users by total transaction amount. Use `CROSS JOIN LATERAL` — the subquery should aggregate each user's total spending, correlated to the outer city, ordered and limited inside the LATERAL subquery.

Only include cities with at least 3 distinct spending users.

**Expected output columns:**  
`city, user_id, total_amount`

Order by `city`, `total_amount DESC`.


WITH cities_filter AS (
SELECT
	u.city,
	COUNT(DISTINCT(t.user_id)) AS user_cnt
FROM crappy_data_db.users u 
JOIN crappy_data_db.transactions t ON u.id = t.user_id
WHERE city IS NOT null
GROUP BY u.city
HAVING COUNT(DISTINCT(t.user_id)) >= 3
),
users_cities_transactions AS (
SELECT 
	u.city,
	t.*
FROM crappy_data_db.users u
CROSS JOIN LATERAL (
SELECT 
	t.user_id,
	SUM(t.amount) AS total_amount
FROM crappy_data_db.transactions t
JOIN cities_filter c ON c.city = u.city
WHERE t.user_id = u.id AND c.city = u.city
GROUP BY u.city, t.user_id
ORDER BY total_amount DESC
LIMIT 3
) t
WHERE u.city IS NOT NULL
)
SELECT * FROM users_cities_transactions
ORDER BY city, total_amount DESC

Tu mi się strasznie coś pokićkało - wróćmy jutro do tego ze scaffoldem. I po co w ogóle tego używać tutaj LATERALA? 



---

## Task 2 — Intraday Cumulative VWAP (Session-Reset)

**Difficulty: 5/5**

**Business question:**  
For each RTH session, calculate a running (cumulative) VWAP tick-by-tick throughout the day:

`running_vwap = SUM(price * size) OVER (...) / SUM(size) OVER (...)`

**Critical constraint:** The VWAP must reset every session — cumulation starts fresh at 09:30 ET each trading day. A tick at 10:00 on trade_date X must NEVER include volume from trade_date X-1. Use `PARTITION BY trade_date` in the window frame to enforce this.

Filter to RTH only (`ts_event AT TIME ZONE 'America/New_York'` between 09:30 and 16:00), exclude `side = 'N'`.

To keep the result set manageable, aggregate into 30-minute buckets first (one row per trade_date × 30-min window), computing the running VWAP as of the end of each bucket — not tick-by-tick for every row.

**Expected output columns:**  
`trade_date, bucket_start, running_vwap`

`running_vwap` rounded to 2 decimals. Order by `trade_date`, `bucket_start`.


Your instructions were contradicting each other - first you suggested the window function, which would be a very typical, yet costly approach, then you suggested to aggregate it at the end of each bucket instead, which made me obviously pick the simpler, less expensive option in terms of memory & resources

WITH ticks_dates_rth AS (
SELECT 
    *,
    ts_event AT TIME ZONE 'America/New_York' AS time_et,
    (ts_event AT TIME ZONE 'America/New_York')::DATE AS trade_date,
    DATE_TRUNC('Hour', ts_event AT TIME ZONE 'America/New_York') 
        + (EXTRACT(MINUTE FROM ts_event AT TIME ZONE 'America/New_York')::int / 30 * INTERVAL '30 Minutes') AS bucket_start
FROM nq_data.ticks t
WHERE (ts_event AT TIME ZONE 'America/New_York')::time >= '09:30'
  AND (ts_event AT TIME ZONE 'America/New_York')::time <= '16:00'
  AND t.side != 'N'
),
ticks_window_start_buckets AS (
SELECT 
	*,
	TO_CHAR(bucket_start, 'HH:MI') AS window_start
FROM ticks_dates_rth
),
buckets_usd_sizes AS (
SELECT 
	trade_date,
	window_start,
	min(price) AS min_price,
	MAX(price) AS max_price,
	SUM(price * size) AS total_usd_volume,
	sUM(size) AS total_volume,
	ROUND(SUM(price * size) / sUM(size), 2) AS window_vwap
FROM ticks_window_start_buckets
GROUP BY trade_date, window_start
)
SELECT * FROM buckets_usd_sizes


---

## Submission Instructions

Paste your queries below each task.
