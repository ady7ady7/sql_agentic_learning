# SQL Tasks — 2026-08-14 (Week 34, Day 5)

**Dataset:** transactions / users / orders  
**Focus:** Pivot z FILTER · FIRST_VALUE + LAST_VALUE per window

---

## Task 1 — Monthly Transaction Pivot per User
**Difficulty: 3/5**

**Business question:**  
For each user, show their total transaction amount broken down by month as separate columns — one column per month from the data. Use only the 4 most recent distinct months in the dataset.

Show users who had at least one transaction in any of those 4 months. Use conditional aggregation with `FILTER`.

**Expected output columns:**  
`user_id, month_1, month_2, month_3, month_4`

Where `month_1` is the oldest of the 4 and `month_4` is the most recent. Column names can reflect the actual month (e.g. `m_2025_06`). NULL if no transactions that month.

Order by `user_id`.

**Difficulty: 3/5**


WITH transactions_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month
FROM crappy_data_db.transactions t
),
users_months AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY USER_ID ORDER BY month DESC) AS user_month
FROM transactions_months
)
SELECT
	user_id,
	SUM(amount) FILTER (WHERE user_month = 1) AS month_1,
	SUM(amount) FILTER (WHERE user_month = 2) AS month_2,
	SUM(amount) FILTER (WHERE user_month = 3) AS month_3,
	SUM(amount) FILTER (WHERE user_month = 4) AS month_4
FROM users_months
GROUP BY user_id
ORDER BY user_id




---

## Task 2 — First and Last Transaction per User per Month
**Difficulty: 4/5**

**Business question:**  
For each user, for each month they were active, show:
- Their first transaction amount that month (`first_amount`)
- Their last transaction amount that month (`last_amount`)
- The difference between last and first (`change`)
- How many transactions they had that month (`tx_count`)

Only include user-months with at least 2 transactions.

**Expected output columns:**  
`user_id, tx_month, first_amount, last_amount, change, tx_count`

Order by `user_id`, `tx_month`.

**Hint:** Use `FIRST_VALUE(amount ORDER BY created_at)` and `FIRST_VALUE(amount ORDER BY created_at DESC)` — both within `PARTITION BY user_id, tx_month`. Avoid LAST_VALUE without an explicit frame.

**Difficulty: 4/5**


WITH users_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month
FROM crappy_data_db.transactions t
),
users_months_summary AS (
SELECT
	user_id,
	MONTH,
	COUNT(*) AS tx_month
FROM users_months
GROUP BY user_id, MONTH
ORDER BY user_id
),
first_last_tx AS (
SELECT 
	*,
	FIRST_VALUE(amount) OVER (PARTITION BY user_id, MONTH ORDER BY created_at) AS first_tx,
	FIRST_VALUE(amount) OVER (PARTITION BY user_id, MONTH ORDER BY created_at DESC) AS last_tx
FROM users_months
)
SELECT DISTINCT ON (u.user_id, u.month)
	u.user_id,
	u.MONTH AS tx_month,
	f.first_tx AS first_amount,
	f.last_tx AS last_amount,
	f.last_tx - f.first_tx AS diff,
	u.tx_month AS tx_count
FROM users_months_summary u
LEFT JOIN first_last_tx f ON u.user_id = f.user_id AND u."month" = f."month"

---

## Submission Instructions

Paste your queries below each task.