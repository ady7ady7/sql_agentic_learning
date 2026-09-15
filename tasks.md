# SQL Tasks — 2026-09-15 (Week 38, Day 2)

**Dataset:** nq_data.ticks · crappy_data_db  
**Focus:** Anti-join (NOT EXISTS) · STDDEV per city

---

## Task 1 — Trading Days with No Activity in the Last 15 Minutes (Anti-Join)

**Difficulty: 4/5**

**Business question:**  
Find all RTH trading days (trade_date, using `ts_event AT TIME ZONE 'America/New_York'`) that have at least one tick in the session, but have ZERO ticks in the last 15 minutes of RTH (15:45–16:00 ET).

Use `NOT EXISTS`.

**Approach:**
1. Build a CTE of all distinct RTH trade_dates (any tick between 09:30–16:00 ET).
2. Use `NOT EXISTS` to exclude any trade_date that has at least one tick between 15:45–16:00 ET.

**Expected output columns:**  
`trade_date`

Order by `trade_date`.

WITH ticks_dates AS (
SELECT 
	*,
	(ts_event AT TIME ZONE 'America/New_York')::date AS trade_date
FROM nq_data.ticks t
LIMIT 100
)
SELECT 
	t.trade_date
FROM ticks_dates t
WHERE NOT EXISTS (
	SELECT * FROM ticks_dates t2
	WHERE t2.trade_date = t.trade_date 
	AND (t2.ts_event AT TIME ZONE 'America/New_York')::time >= '15:45' AND (t2.ts_event AT TIME ZONE 'America/New_York')::time <= '16:00'
)

There are no such days, it's empty xD.



---

## Task 2 — Spending Variability by City (STDDEV)

**Difficulty: 4/5**

**Business question:**  
For each city, calculate the standard deviation of total transaction amount across its users (one total per user, then STDDEV across those per-user totals — not STDDEV across individual transactions).

Only include cities with at least 5 users who have transactions.

**Expected output columns:**  
`city, user_count, avg_total_amount, stddev_total_amount`

Both numeric columns rounded to 2 decimals. Order by `stddev_total_amount DESC`.


WITH users_cities_t_totals AS (
SELECT 
	u.city,
	t.user_id,
	SUM(t.amount) AS total_amount
FROM crappy_data_db.users u
JOIN crappy_data_db.transactions t ON u.id = t.user_id
WHERE u.city IS NOT null
GROUP BY u.city, t.user_id
)
SELECT 
	city,
	COUNT(*) AS user_count,
	round(AVG(total_amount), 2) AS avg_total_amount,
	ROUND(stddev(TOTAL_AMOUNT), 2) AS stddev_total_amount
FROM users_cities_t_totals
GROUP BY city
HAVING COUNT(*) >= 5
ORDER BY stddev_total_amount desc


---

## Submission Instructions

Paste your queries below each task.
