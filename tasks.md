# SQL Tasks — 2026-09-14 (Week 38, Day 1)

**Dataset:** nq_data.ticks · crappy_data_db  
**Focus:** True cumulative VWAP (third attempt) · Self-join on dominant type · LAG (no offset)

---

## Task 1 — Intraday Cumulative VWAP (Third Attempt: Weighted Cumulation)

**Difficulty: 5/5**

**Business question:**  
Same goal: running VWAP from 09:30 ET, resetting per session.

**The rule that was missing last time:** never cumulate or average an already-divided ratio (like a per-bucket VWAP) when the groups behind it have different weights (volume). Always cumulate the raw numerator and denominator separately, and divide only once, at the very end.

**Correct shape:**
```sql
bucket_sums AS (
    SELECT trade_date, bucket_start,
        SUM(price * size) AS bucket_usd,
        SUM(size) AS bucket_size
    FROM ticks_buckets_rth
    GROUP BY trade_date, bucket_start
),
running AS (
    SELECT *,
        SUM(bucket_usd) OVER (PARTITION BY trade_date ORDER BY bucket_start) AS cum_usd,
        SUM(bucket_size) OVER (PARTITION BY trade_date ORDER BY bucket_start) AS cum_size
    FROM bucket_sums
)
SELECT trade_date, bucket_start,
    ROUND(cum_usd / cum_size, 2) AS running_vwap
FROM running
ORDER BY trade_date, bucket_start
```

Note there is no `bucket_vwap` (divided value) anywhere in the window functions — only raw sums get cumulated. The division happens exactly once, in the final SELECT.

**Expected output columns:**  
`trade_date, bucket_start, running_vwap`

`running_vwap` rounded to 2 decimals. Order by `trade_date`, `bucket_start`.

**Sanity check:** for one trade_date, the running_vwap at the last bucket of the day should exactly equal a flat `SUM(price*size)/SUM(size)` over all RTH ticks of that day (no window function — direct aggregate as ground truth).

WITH rth_ticks_dates AS (
SELECT 
	*,
	DATE_TRUNC('Hour', ts_event AT TIME ZONE 'America/New_York') + (EXTRACT('Minute' FROM ts_event AT TIME ZONE 'America/New_York')::int/30 * INTERVAL '30 Minutes') AS bucket_start,
	(ts_event AT TIME ZONE 'America/New_York')::time AS et_time,
	(ts_event AT TIME ZONE 'America/New_York')::date AS trade_date
FROM nq_data.ticks t
WHERE (ts_event AT TIME ZONE 'America/New_York')::time >= '9:30' AND (ts_event AT TIME ZONE 'America/New_York')::time <= '16:00'
LIMIT 150000
),
bucket_windows AS (
SELECT 
	*,
	TO_CHAR(bucket_start, 'HH24:MI') AS bucket_window
FROM rth_ticks_dates
),
buckets_cum_size_usd AS (
SELECT 
	trade_date,
	bucket_start,
	bucket_window,
	SUM(size) AS bucket_size,
	SUM(price * size) AS bucket_total_usd
FROM bucket_windows
GROUP BY trade_date, bucket_start, bucket_window
),
running_sums AS (
SELECT 
	*,
	sum(bucket_total_usd) OVER (PARTITION BY trade_date ORDER BY bucket_start) AS running_usd,
	sum(bucket_size) OVER (PARTITION BY trade_date ORDER BY bucket_start) AS running_volume
FROM buckets_cum_size_usd
),
running_vwaps AS (
SELECT 
	*,
	ROUND(running_usd / running_volume, 2) AS running_vwap
FROM running_sums
)
SELECT 
	* 
FROM running_vwaps


I've done it correctly this time and I don't think we need the sanity check.




---

## Task 2 — City Pairs Sharing the Same Dominant Transaction Type (Self-Join)

**Difficulty: 3/5**

**Business question:**  
For each city, determine its dominant transaction type (the type with the highest transaction count among users from that city). Then find pairs of cities that share the same dominant type.

Exclude NULL cities. Avoid duplicate pairs (`city_a < city_b`).

**Expected output columns:**  
`city_a, city_b, dominant_type`

Order by `dominant_type`, `city_a`.



WITH cities_types_amounts AS (
SELECT
	u.city,
	t.TYPE,
	COUNT(t.amount) AS transactions_cnt
FROM crappy_data_db.users u
JOIN crappy_data_db.transactions t ON u.id = t.user_id 
WHERE u.city IS NOT NULL
GROUP BY u.city, t.TYPE
),
cities_type_ranks AS (
SELECT 
	*,
	Row_number() OVER (PARTITION BY city ORDER BY transactions_cnt DESC) AS city_type_rank
FROM cities_types_amounts
)
SELECT 
	c1.city AS city_a,
	c2.city AS city_b,
	c1.TYPE AS dominant_type
FROM cities_type_ranks c1
JOIN cities_type_ranks c2 ON c1.TYPE = c2.TYPE AND c1.city > c2.city
WHERE c1.city_type_rank = 1 AND c2.city_type_rank = 1



---

## Task 3 — Amount Change Between Consecutive Transactions (LAG)

**Difficulty: 3/5**

**Business question:**  
For each user, show every transaction alongside the amount of their previous transaction and the difference (`amount - previous_amount`).

**Expected output columns:**  
`user_id, id, created_at, amount, prev_amount, amount_diff`

Order by `user_id`, `created_at`.


SELECT 
	user_id,
	id,
	created_at,
	amount,
	lag(AMOUNT) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_amount,
	amount - lag(AMOUNT) OVER (PARTITION BY user_id ORDER BY created_at) AS amount_diff
FROM crappy_data_db.transactions t


Super easy.


---

## Submission Instructions

Paste your queries below each task.
