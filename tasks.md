# SQL Tasks — 2026-09-09 (Week 37, Day 3)

**Dataset:** transactions  
**Focus:** Pivot with conditional aggregation · RANK (global, no partition)

---

## Task 1 — Transaction Count by Type per Month (Pivot)

**Difficulty: 3/5**

**Business question:**  
For each month, show the count of transactions broken down by `type` — one column per type. Use conditional aggregation (`FILTER` or `CASE WHEN`).

**Expected output columns:**  
`month, deposit_count, withdrawal_count, transfer_count, payment_count, purchase_count`

Order by `month`.


WITH t_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
FROM crappy_data_db.transactions t
)
SELECT 
	month_,
	COUNT(*) FILTER (WHERE TYPE = 'deposit') AS deposit_count,
	COUNT(*) FILTER (WHERE TYPE = 'withdrawal') AS withdrawal_count,
	COUNT(*) FILTER (WHERE TYPE = 'payment') AS payment_count,
	COUNT(*) FILTER (WHERE TYPE = 'transfer') AS transfer_count,
	COUNT(*) FILTER (WHERE TYPE = 'purchase') AS purchase_count
FROM t_months
GROUP BY month_
ORDER BY month_

---

## Task 2 — Top 5 Largest Transactions per Month (Global RANK)

**Difficulty: 3/5**

**Business question:**  
For each month, find the 5 largest transactions by `amount` — ranked globally within that month (not per user). Use `RANK()`.

If there's a tie at position 5, include all tied transactions.

**Expected output columns:**  
`month, id, user_id, amount, rank`

Order by `month`, `rank`.


WITH t_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
FROM crappy_data_db.transactions t
),
monthly_ranks AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY month_ ORDER BY amount DESC) AS rank
FROM t_months
)
SELECT 
	month_,
	id,
	user_id,
	amount,
	rank
FROM monthly_ranks
WHERE RANK <= 5
ORDER BY month_, rank

---

## Submission Instructions

Paste your queries below each task.
