# SQL Tasks — 2026-09-07 (Week 37, Day 1)

**Dataset:** transactions / users  
**Focus:** LATERAL (repetition) · Pivot with conditional aggregation

---

## Task 1 — 2 Most Recent Transactions per User (LATERAL)

**Difficulty: 3/5**

**Business question:**  
For each user, find their 2 most recent transactions by `created_at` (descending). Use `CROSS JOIN LATERAL` with a correlated subquery.

Only include users who have at least 2 transactions.

**Expected output columns:**  
`user_id, id, amount, created_at`

Order by `user_id`, `created_at DESC`.


SELECT 
	u.id AS user_id,
	t.*
FROM crappy_data_db.users u
CROSS JOIN LATERAL 
(
SELECT 
	* 
FROM crappy_data_db.transactions t 
WHERE u.id = t.user_id
ORDER BY created_at DESC
LIMIT 2
) t;


---

## Task 2 — Transaction Count by Type per Month (Pivot)

**Difficulty: 3/5**

**Business question:**  
For each month, show the count of transactions broken down by `type` — one column per type. Use conditional aggregation (`FILTER` or `CASE WHEN`).

**Expected output columns:**  
`month, deposit_count, withdrawal_count, transfer_count, payment_count, purchase_count`

Order by `month`.


WITH transactions_months AS (
SELECT  
	*,
	DATE_TRUNC('Month', created_at) AS month_
FROM crappy_data_db.transactions t 
)
SELECT 
	month_,
	COUNT(*) FILTER (WHERE TYPE = 'deposit') AS deposit_count,
	COUNT(*) FILTER (WHERE TYPE = 'withdrawal') AS withdrawal_count,
	COUNT(*) FILTER (WHERE TYPE = 'transfer') AS transfer_count,
	COUNT(*) FILTER (WHERE TYPE = 'payment') AS payment_count,
	COUNT(*) FILTER (WHERE TYPE = 'purchase') AS purchase_count
FROM transactions_months
GROUP BY month_
ORDER BY month_



---

## Submission Instructions

Paste your queries below each task.
