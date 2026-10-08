# SQL Tasks — 2026-10-08 (Week 41, Day 4)

**Dataset:** crappy_data_db  
**Focus:** SUM with FILTER (conditional sums, not just conditional counts) · DISTINCT ON (new context)

---

## Task 1 — Conditional Sums by Transaction Type (SUM FILTER)

**Difficulty: 3/5**

**Business question:**  
For each user, show their total deposited amount and total withdrawn amount side by side, in one row — using `SUM(amount) FILTER (WHERE ...)`, not `COUNT(*) FILTER`.

Only include users who have at least one deposit AND at least one withdrawal.

**Expected output columns:**  
`user_id, total_deposited, total_withdrawn`

Order by `user_id`.


WITH users_with_depo AS (
SELECT
	user_id,
	COALESCE(SUM(amount) FILTER (WHERE TYPE = 'deposit'), 0) AS total_deposited,
	COALESCE(SUM(amount) FILTER (WHERE TYPE = 'withdrawal'), 0) AS total_withdrawn
FROM crappy_data_db.transactions
GROUP BY user_id
)
SELECT 
	* 
FROM users_with_depo 
WHERE total_deposited > 0 AND total_withdrawn > 0





---

## Task 2 — Highest-Value Order per City (DISTINCT ON)

**Difficulty: 4/5**

**Business question:**  
For each city, find the single highest-value order (by `amount`) placed by any user from that city — not the user's total, the order itself.

Only include orders where the user's `city` is not NULL.

**Expected output columns:**  
`city, order_id, user_id, amount`

Order by `city`.


SELECT DISTINCT ON (city)
	u.city,
	o.id AS order_id,
	o.user_id,
	o.amount
FROM crappy_data_db.orders o
JOIN crappy_data_db.users u ON o.user_id = u.id
WHERE u.city IS NOT NULL
ORDER BY city, amount DESC



---

## Submission Instructions

Paste your queries below each task.
