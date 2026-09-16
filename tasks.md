# SQL Tasks — 2026-09-16 (Week 38, Day 3)

**Dataset:** crappy_data_db · nq_data.ticks  
**Focus:** Cohort retention (two steps) · First 5-min range vs full-day range

---

## Task 1 — Cohort Retention: Month +1 and Month +2

**Difficulty: 4/5**

**Business question:**  
For each user, find their cohort month (the month of their first-ever order). Then check, per cohort:
- how many users returned with at least one order in month +1
- how many users returned with at least one order in month +2

Show both retention steps side by side per cohort month.

**Expected output columns:**  
`cohort_month, cohort_size, returned_month1, retention_month1_pct, returned_month2, retention_month2_pct`

Both percentage columns rounded to 2 decimals. Order by `cohort_month`.



WITH users_cohorts AS (
SELECT 
	*,
	o1.user_id AS uid1,
	o2.user_id AS uid2,
	DATE_TRUNC('Month', o1.created_at) AS month1,
	DATE_TRUNC('Month', o2.created_at) AS month2,
	DATE_TRUNC('Month', u.created_at) AS cohort_month
FROM crappy_data_db.users u
LEFT JOIN crappy_data_db.orders o1 ON u.id = o1.user_id AND DATE_TRUNC('Month', o1.created_at) >= DATE_TRUNC('Month', u.created_at) + INTERVAL '1 Month' AND DATE_TRUNC('Month', o1.created_at) < DATE_TRUNC('Month', u.created_at) + INTERVAL '2 Month'
LEFT JOIN crappy_data_db.orders o2 ON u.id = o2.user_id AND DATE_TRUNC('Month', o2.created_at) >= DATE_TRUNC('Month', u.created_at) + INTERVAL '2 Month' AND DATE_TRUNC('Month', o2.created_at) < DATE_TRUNC('Month', u.created_at) + INTERVAL '3 Month'
)
SELECT
	cohort_month,
	COUNT(DISTINCT(email)) AS cohort_size,
	COUNT(DISTINCT(uid1)) AS returned_month1,
	ROUND(COUNT(DISTINCT(uid1)) / COUNT(DISTINCT(email))::NUMERIC * 100, 2) AS retention_month1_pct,
	COUNT(DISTINCT(uid2)) AS returned_month2,
	ROUND(COUNT(DISTINCT(uid2)) / COUNT(DISTINCT(email))::NUMERIC * 100, 2) AS retention_month2_pct
FROM users_cohorts
GROUP BY cohort_month


---

## Task 2 — First 5 Minutes Range vs Full RTH Day Range

**Difficulty: 4/5**

**Business question:**  
For each RTH session, calculate:
- the price range (high - low) during the first 5 minutes (09:30–09:35 ET)
- the full RTH day's price range (09:30–16:00 ET)
- what percentage of the full day's range was already captured in those first 5 minutes

Filter to RTH only, exclude `side = 'N'`.

**Expected output columns:**  
`trade_date, first5min_range, full_day_range, pct_captured`

`pct_captured` rounded to 2 decimals. Order by `trade_date`.


WITH ticks_dates AS (
SELECT 
	*,
	(ts_event AT TIME ZONE 'America/New_York')::time AS et_time,
	(ts_event AT TIME ZONE 'America/New_York')::date AS trade_date
FROM nq_data.ticks t
WHERE (ts_event AT TIME ZONE 'America/New_York')::time >= '9:30' AND (ts_event AT TIME ZONE 'America/New_York')::time <= '16:00'
),
agg_5min AS (
SELECT 
	trade_date,
	MIN(price) AS low_5min,
	MAX(price) AS high_5min,
	MAX(price) - MIN(price) AS first5min_range
FROM ticks_dates
WHERE (ts_event AT TIME ZONE 'America/New_York')::time >= '9:30' AND (ts_event AT TIME ZONE 'America/New_York')::time <= '9:35'
GROUP BY trade_date
),
agg_daily AS (
SELECT
	trade_date,
	MIN(price) AS daily_low,
	MAX(price) AS daily_high,
	MAX(price) - MIN(price) AS daily_range
FROM ticks_dates
GROUP BY trade_date
)
SELECT 
	a.trade_date,
	a2.first5min_range,
	a.daily_range,
	ROUND(a2.first5min_range / a.daily_range::NUMERIC * 100, 2) AS pct_captured
FROM agg_daily a
JOIN agg_5min a2 ON a.trade_date = a2.trade_date
GROUP BY a.trade_date, a2.first5min_range, A.daily_range 
ORDER BY a.trade_date


Zrobione

---

## Submission Instructions

Paste your queries below each task.
