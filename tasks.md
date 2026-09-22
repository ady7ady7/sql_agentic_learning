# SQL Tasks — 2026-09-22 (Week 39, Day 2)

**Dataset:** nq_data.ticks · crappy_data_db  
**Focus:** DISTINCT ON (new dataset) · NTH_VALUE

---

## Task 1 — Largest Single Print per RTH Session (DISTINCT ON)

**Difficulty: 4/5**

**Business question:**  
For each RTH trading day, find the single tick with the largest `size` (the biggest single print of the session). Use `DISTINCT ON`.

Filter to RTH only, exclude `side = 'N'`.

**Expected output columns:**  
`trade_date, ts_event, price, size, side`

Order by `trade_date`.


SELECT DISTINCT ON ((ts_event AT TIME ZONE 'America/New_York')::date)
	(ts_event AT TIME ZONE 'America/New_York')::date AS trade_date,
	ts_event AT TIME ZONE 'America/New_York' AS et_time,
	price,
	SIZE,
	side
FROM nq_data.ticks t
WHERE side != 'N'
ORDER BY trade_date, size DESC



---

## NTH_VALUE — Introduction

`NTH_VALUE(column, n) OVER (...)` returns the value of `column` at the N-th row of the window frame, according to the frame's `ORDER BY`. It's a generalization of `FIRST_VALUE` (which is just `NTH_VALUE(column, 1)`).

By default, the frame only extends up to the current row, so `NTH_VALUE` often needs an explicit frame (e.g. `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`) to "see" the whole partition, the same way `LAST_VALUE` does.

Use case: finding the 2nd-highest, 3rd-highest, etc. value within a group — without a separate RANK + filter step.




---

## Task 2 — Second-Largest Transaction per User (NTH_VALUE)

**Difficulty: 4/5**

**Business question:**  
For each user, find the amount of their second-largest transaction (by `amount`). Use `NTH_VALUE`, not `RANK`/`ROW_NUMBER`.

Only include users with at least 2 transactions.

**Expected output columns:**  
`user_id, second_largest_amount`

One row per user. Order by `user_id`.

WITH users_transactions AS (
SELECT 
	*,
	nth_value(amount, 2) OVER (PARTITION BY user_id ORDER BY amount DESC) AS second_largest_t
FROM crappy_data_db.transactions t
)
SELECT 
	user_id,
	second_largest_t
FROM users_transactions
WHERE second_largest_t IS NOT NULL
GROUP BY user_id, second_largest_t
ORDER BY user_id

The WHERE filter automatically deletes users with less than 2 transactions.



---

## Submission Instructions

Paste your queries below each task.
