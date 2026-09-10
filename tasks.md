# SQL Tasks — 2026-09-10 (Week 37, Day 4)

**Dataset:** transactions / users · nq_data.ticks  
**Focus:** LATERAL top-N-per-group (scaffolded) · True cumulative VWAP · Cohort retention

---

## Task 1 — Top 3 Spenders per City (LATERAL, Scaffolded)

**Difficulty: 4/5**

**Business question:**  
For each city, find the top 3 users by total transaction amount.

**Why LATERAL here, specifically:** You need, for each city, to run a full aggregation (GROUP BY user + SUM + ORDER BY + LIMIT) that's correlated to that city. A plain JOIN can't do this — it can only match rows, not "run this mini-query per outer row." LATERAL is the tool for exactly this shape: "for each X, compute a small ranked/limited result set that depends on X."

**Step-by-step scaffold:**

**Step A — build the filter first, standalone.** Write a CTE `eligible_cities` that lists only cities with >= 3 distinct users who have transactions. This has nothing to do with LATERAL — it's a plain JOIN + GROUP BY + HAVING. Get this right and verify it in isolation before moving on.

**Step B — write the LATERAL subquery standalone, for ONE hardcoded city first.** Before correlating it to anything, write and test:
```sql
SELECT t.user_id, SUM(t.amount) AS total_amount
FROM crappy_data_db.users u
JOIN crappy_data_db.transactions t ON t.user_id = u.id
WHERE u.city = 'SomeRealCityFromYourData'
GROUP BY t.user_id
ORDER BY total_amount DESC
LIMIT 3
```
Run this with a real city name. Confirm it returns 3 rows with sensible totals.

**Step C — correlate it.** Replace the hardcoded city with a reference to the outer city (`eligible_cities.city`), and wrap it as `CROSS JOIN LATERAL (...) AS top`. The subquery body from Step B barely changes — only the `WHERE u.city = ...` line changes from a literal to a correlated reference.

**Step D — assemble.** `FROM eligible_cities CROSS JOIN LATERAL (...) top`. No extra JOINs inside the LATERAL subquery back to `eligible_cities` — the correlation IS the join, you don't need another one.

**Expected output columns:**  
`city, user_id, total_amount`

Order by `city`, `total_amount DESC`.


WITH eligible_cities AS (
SELECT
	city,
	COUNT(DISTINCT(t.user_id)) AS user_cnt
FROM crappy_data_db.users u
JOIN crappy_data_db.transactions t ON u.id = t.user_id
GROUP BY city
HAVING COUNT(DISTINCT(t.user_id)) >= 3
)
SELECT 
	e.city,
	top.*
FROM eligible_cities e
CROSS JOIN LATERAL (
	SELECT 
	t.user_id,
	SUM(t.amount) AS total_amount
	FROM crappy_data_db.users u 
	JOIN crappy_data_db.transactions t ON u.id = t.user_id
	WHERE u.city = e.city
	GROUP BY t.user_id
	ORDER BY total_amount DESC
	LIMIT 3
) AS top


I've done it and I've skipped step 2 as it's useless frankly.
I've used your instructions and it helped today, still not feeling confident about this, but it was a tiny bit better.


Some data as example:

Gdańsk	14	7413.84
Gdańsk	53	5732.65
Gdańsk	81	4668.82
Gliwice	82	8240.92
Gliwice	60	7809.81
Gliwice	49	7052.53
Haga	83	6687.47
Haga	43	4653.65
Haga	10	4620.41
Katowice	84	8134.55

---

## Task 2 — Intraday Cumulative VWAP (Session-Reset) — True Running Version

**Difficulty: 5/5**

**Business question:**  
Continuing from the per-bucket VWAP you already built: now make it a true **running** VWAP that accumulates from 09:30 ET onward within each session, resetting at the start of every new trading day.

**Two-layer approach:**
1. You already have (or can rebuild) per-bucket sums: `bucket_usd_volume = SUM(price*size)` and `bucket_volume = SUM(size)`, one row per `(trade_date, bucket_start)`.
2. On top of that small aggregated result, compute a **cumulative sum** of those two columns using a window function: `SUM(bucket_usd_volume) OVER (PARTITION BY trade_date ORDER BY bucket_start)` and the same for volume. `PARTITION BY trade_date` is what makes it reset every session — this is not optional.
3. Divide the two cumulative sums to get `running_vwap` as of the end of each bucket.

**Expected output columns:**  
`trade_date, bucket_start, running_vwap`

`running_vwap` rounded to 2 decimals. Order by `trade_date`, `bucket_start`.

**Sanity check:** the last bucket of each day's `running_vwap` should equal the full-day VWAP you calculated in the earlier session-close task.


This time I simply used proper window functions to calculate running vwap for each day instead of what I did last time. This could be way simpler but I wanted to do the sanity check you've asked for. 


WITH ticks_dates_rth AS (
SELECT
	*,
	(ts_event AT TIME ZONE 'America/New_York')::date AS trade_date,
	(ts_event AT TIME ZONE 'America/New_York')::time AS et_time,
	DATE_TRUNC('Hour', ts_event AT TIME ZONE 'America/New_York') + (EXTRACT(MINUTE FROM ts_event AT TIME ZONE 'America/New_York')::int / 15 * INTERVAL '15 Minutes') AS bucket_start
FROM nq_data.ticks t
WHERE (ts_event AT TIME ZONE 'America/New_York')::time >= '9:30' AND (ts_event AT TIME ZONE 'America/New_York')::time <= '16:00'
LIMIT 10000
),
running_vwap_calc AS (
SELECT 
	*,
	TO_CHAR(bucket_start, 'HH24:MI') AS bucket_window,
	ROUND(SUM(price * size) OVER (PARTITION BY trade_date ORDER BY ts_event) / SUM(size) OVER (PARTITION BY trade_date ORDER BY ts_event), 2) AS running_vwap
FROM ticks_dates_rth
),
sanity_check_vwaps AS (
SELECT
	bucket_start,
	ROUND(SUM(price * size) / SUM(size), 2) AS sanity_check_final_vwap
FROM running_vwap_calc
GROUP BY bucket_start
),
last_timestamp_running_vwaps AS (
SELECT 
	r.bucket_start,
	MAX(ts_event) AS last_timestamp
FROM running_vwap_calc r
GROUP BY r.bucket_start
)
SELECT 
	l.bucket_start,
	r.running_vwap,
	s.sanity_check_final_vwap
FROM last_timestamp_running_vwaps l
JOIN running_vwap_calc r ON l.last_timestamp = r.ts_event
JOIN sanity_check_vwaps s ON r.bucket_start = s.bucket_start


I could obviously pick the format you wanted, not a big deal, but I ended up with the sanity check :)). There are]some differences between the two (running vwap vs sanity_check_final_vwap which I'm not sure about...)


bucket_start	running_vwap	sanity_check_final_vwap
2025-09-30 10:00:00.000	24,761.37	24,761.37
2025-09-30 10:15:00.000	24,773.95	24,778.71
2025-09-30 10:30:00.000	24,777.34	24,781.23
2025-09-30 10:45:00.000	24,794.95	24,827.85
2025-09-30 10:45:00.000	24,794.95	24,827.85
2025-09-30 11:00:00.000	24,806.5	24,848.21




---

## Task 3 — Simple Cohort Retention (Month +1)

**Difficulty: 3/5**

**Business question:**  
For each user, find their cohort month (the month of their first-ever order). Then check: did that user place at least one more order in the month immediately following their cohort month?

Show, per cohort month: the number of users in that cohort, and the number/percentage who returned in month +1.

**Expected output columns:**  
`cohort_month, cohort_size, returned_next_month, retention_pct`

`retention_pct` rounded to 2 decimals. Order by `cohort_month`.


WITH users_first_orders AS (
SELECT 
	user_id,
	MIN(created_at) AS first_order
FROM crappy_data_db.orders o
GROUP BY user_id
),
fo_cohorts AS (
SELECT 
	*,
	date_trunc('Month', first_order) AS cohort_month_
FROM users_first_orders
),
cohorts_sizes_returned AS (
SELECT 
	f.cohort_month_,
	COUNT(DISTINCT(f.user_id)) AS cohort_size,
	COUNT(DISTINCT(f.user_id)) FILTER (WHERE o.id IS NOT NULL) AS returned_next_month
FROM fo_cohorts f
LEFT JOIN crappy_data_db.orders o 
ON o.user_id = f.user_id
AND date_trunc('Month', o.created_at) = f.cohort_month_ + INTERVAL '1 Month'
GROUP BY f.cohort_month_
)
SELECT 
	*,
	ROUND(returned_next_month / cohort_size::NUMERIC * 100, 2) AS retention_pct
FROM cohorts_sizes_returned
ORDER BY cohort_month_


Not a big deal, but also a nice task that already feels like a solid Mid+ comprehension elvel

---

## Submission Instructions

Paste your queries below each task.
