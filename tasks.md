# SQL Tasks — 2026-09-18 (Week 38, Day 5)

**Dataset:** crappy_data_db  
**Focus:** PERCENTILE_CONT (median) · DISTINCT ON (multi-column)

---

## Task 1 — Median Spending per City (PERCENTILE_CONT)

**Difficulty: 4/5**

**Business question:**  
For each city, calculate both the average and the median total transaction amount across its users (one total per user, same shape as the STDDEV task — aggregate per user first, then compute city-level stats across those per-user totals).

Use `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY ...)` for the median.

Only include cities with at least 5 users who have transactions.

**Expected output columns:**  
`city, user_count, avg_total_amount, median_total_amount`

Both numeric columns rounded to 2 decimals. Order by `city`.

WITH user_amts AS (
SELECT 
	user_id,
	city,
	SUM(amount) AS total_amount
FROM crappy_data_db.transactions t
JOIN crappy_data_db.users u ON t.user_id = u.id
GROUP BY user_id, city
)
SELECT 
	city,
	COUNT(*) AS user_count,
	ROUND(AVG(total_amount), 2) AS avg_total_amount,
	ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY total_amount)::NUMERIC, 2) AS median_total_amount
FROM user_amts
WHERE city IS NOT NULL
GROUP BY city
HAVING COUNT(*) >= 5
ORDER BY city

---

## Task 2 — First Transaction of Each Type per User (DISTINCT ON)

**Difficulty: 4/5**

**Business question:**  
For each user, find their first-ever transaction of EACH type (not just their single first transaction overall) — e.g. their first deposit, their first withdrawal, etc. Use `DISTINCT ON (user_id, type)`.

**Expected output columns:**  
`user_id, type, id, created_at, amount`

Order by `user_id`, `type`.


SELECT DISTINCT ON (user_id, type)
	user_id,
	TYPE,
	id,
	created_at,
	amount
FROM crappy_data_db.transactions t
ORDER BY user_id, TYPE, created_at


Niby takie proste, a serio musiałem chwilę się zastanowić - wcale nieoczywiste... TRUDNE!


---

## Submission Instructions

Paste your queries below each task.
