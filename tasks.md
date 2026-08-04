# SQL Tasks — 2026-08-04 (Week 33, Day 2)

**Dataset:** orders / transactions / users  
**Focus:** STDDEV anomaly detection · FIRST_VALUE baseline · Conditional aggregation FILTER pivot

---

## Task 1 — Spending Outliers
**Difficulty: 3/5**

**Business question:**  
Find users whose **total transaction amount** is a statistical outlier — more than 2 standard deviations above the mean across all users.

Show each outlier user, their total spending, the dataset mean, the stddev, and how many standard deviations above the mean they are.

**Expected output columns:**  
`user_id, total_amount, mean_amount, stddev_amount, deviations_above_mean`

Order by `deviations_above_mean DESC`.

**Difficulty: 3/5**


WITH users_totals AS (
SELECT 
	user_id,
	SUM(amount) AS total_amount
FROM crappy_data_db.transactions t
GROUP BY user_id
),
dataset_std_means AS (
SELECT 
	*,
	(SELECT avg(total_amount) FROM users_totals) AS mean_amount_dataset,
	(SELECT stddev(total_amount) FROM users_totals) AS stddev_amount_dataset
FROM users_totals
),
users_agg AS (
SELECT 
	user_id,
	total_amount,
	mean_amount_dataset,
	stddev_amount_dataset,
	ROUND(total_amount / stddev_amount_dataset, 2) AS deviations_above_mean
FROM dataset_std_means
ORDER BY deviations_above_mean DESC
)
SELECT * FROM users_agg
WHERE deviations_above_mean >= 2


---

## Task 2 — Growth vs First Transaction
**Difficulty: 3/5**

**Business question:**  
For each user, treat their **first transaction** (by `created_at`) as the baseline. For every subsequent transaction, show the percentage change in amount relative to that first transaction.

Only include users who have more than 1 transaction. Only include rows after the first transaction (exclude the baseline row itself).

**Expected output columns:**  
`user_id, created_at, amount, first_amount, pct_change`

Where `pct_change = ROUND((amount - first_amount) / first_amount * 100, 2)`.

Order by `user_id`, `created_at`.

**Difficulty: 3/5**

WITH users_first_tx AS (
SELECT 
	*,
	FIRST_VALUE(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS first_amount 
FROM crappy_data_db.transactions t
)
SELECT
	user_id,
	created_at,
	amount,
	first_amount,
	ROUND((amount - first_amount) / first_amount::NUMERIC * 100, 2) AS pct_change
FROM users_first_tx
WHERE amount != first_amount


The results are already ordered by the window func, so there's no need to use ORDER BY

---

## Task 3 — Transaction Type Pivot Per User
**Difficulty: 5/5**

**Business question:**  
For each user, produce a single summary row showing:
- Total number of transactions
- Count and total amount for each of the 5 transaction types: `deposit`, `withdrawal`, `payment`, `transfer`, `purchase`

Use conditional aggregation with `FILTER` (not CASE WHEN). No subqueries, no JOINs — one pass over `transactions`.

Then filter the result to only users who have **at least 3 different transaction types** (i.e. at least 3 of the 5 type counts are > 0).

**Expected output columns:**  
`user_id, total_tx, deposit_cnt, deposit_sum, withdrawal_cnt, withdrawal_sum, payment_cnt, payment_sum, transfer_cnt, transfer_sum, purchase_cnt, purchase_sum`

Order by `total_tx DESC`.

**Difficulty: 5/5**

SELECT 
	user_id,
	COUNT(*) AS total_tx,
	COUNT(DISTINCT(type)) AS types_cnt,
	COALESCE(COUNT(*) FILTER (WHERE TYPE = 'deposit'), 0) AS deposit_cnt,
	COALESCE(SUM(amount) FILTER (WHERE TYPE = 'deposit'), 0) AS deposit_sum,
	COALESCE(COUNT(*) FILTER (WHERE TYPE = 'withdrawal'), 0) AS withdrawal_cnt,
	COALESCE(SUM(amount) FILTER (WHERE TYPE = 'withdrawal'), 0) AS withdrawal_sum,
	COALESCE(COUNT(*) FILTER (WHERE TYPE = 'payment'), 0) AS payment_cnt,
	COALESCE(SUM(amount) FILTER (WHERE TYPE = 'payment'), 0) AS payment_sum,
	COALESCE(COUNT(*) FILTER (WHERE TYPE = 'transfer'), 0) AS transfer_cnt,
	COALESCE(SUM(amount) FILTER (WHERE TYPE = 'transfer'), 0) AS transfer_sum,
	COALESCE(COUNT(*) FILTER (WHERE TYPE = 'purchase'), 0) AS purchase_cnt,
	COALESCE(SUM(amount) FILTER (WHERE TYPE = 'purchase'), 0) AS purchase_sum
FROM crappy_data_db.transactions t
GROUP BY user_id
HAVING COUNT(DISTINCT(type)) >= 3
ORDER BY total_tx DESC



SELECT 
	user_id,
	COUNT(*) AS total_tx,
	COALESCE(COUNT(*) FILTER (WHERE TYPE = 'deposit'), 0) AS deposit_cnt,
	COALESCE(SUM(amount) FILTER (WHERE TYPE = 'deposit'), 0) AS deposit_sum,
	COALESCE(COUNT(*) FILTER (WHERE TYPE = 'withdrawal'), 0) AS withdrawal_cnt,
	COALESCE(SUM(amount) FILTER (WHERE TYPE = 'withdrawal'), 0) AS withdrawal_sum,
	COALESCE(COUNT(*) FILTER (WHERE TYPE = 'payment'), 0) AS payment_cnt,
	COALESCE(SUM(amount) FILTER (WHERE TYPE = 'payment'), 0) AS payment_sum,
	COALESCE(COUNT(*) FILTER (WHERE TYPE = 'transfer'), 0) AS transfer_cnt,
	COALESCE(SUM(amount) FILTER (WHERE TYPE = 'transfer'), 0) AS transfer_sum,
	COALESCE(COUNT(*) FILTER (WHERE TYPE = 'purchase'), 0) AS purchase_cnt,
	COALESCE(SUM(amount) FILTER (WHERE TYPE = 'purchase'), 0) AS purchase_sum
FROM crappy_data_db.transactions t
GROUP BY user_id
ORDER BY total_tx DESC

---

## Submission Instructions

Paste your queries below each task.
