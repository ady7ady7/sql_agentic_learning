# SQL Tasks — 2026-09-01 (Week 36, Day 1)

**Dataset:** transactions / users  
**Focus:** Gaps-and-islands (session-based) · Z-score outliers · ROWS vs RANGE

---

## Task 1 — Transaction Sessions per User (Gaps-and-Islands, Time-Based)

**Difficulty: 5/5**

**Business question:**  
Group each user's transactions into "sessions". A new session starts whenever the gap between two consecutive transactions (for the same user) exceeds 60 minutes.

For each transaction, show the user, the transaction id, created_at, and the session number (1, 2, 3, ...) within that user.

**Hint:** Use LAG to get the previous transaction timestamp, compute the gap in minutes, flag where a new session starts (gap > 60 or no previous = new session), then use a cumulative SUM of that flag as the session number.

Only include users with at least 3 transactions.

**Expected output columns:**  
`user_id, id, created_at, session_number`

Order by `user_id`, `created_at`.


WITH users_transactions AS (
SELECT 
	*,
	LAG(created_at) OVER (PARTITION BY user_id ORDER BY created_at) prev_transaction 
FROM crappy_data_db.transactions t
),
prev_transaction_diff AS (
SELECT 
	*,
	ROUND(EXTRACT(EPOCH FROM created_at - prev_transaction) / 60, 0) AS mins_from_prev_transaction
FROM users_transactions sers_transactions
WHERE prev_transaction IS NOT NULL
),
transactions_streak_keys AS (
SELECT 
	*,
	COUNT(id) OVER (PARTITION BY user_id) AS transaction_cnt,
	CASE WHEN mins_from_prev_transaction <= 60 THEN 0 ELSE 1 END AS streak_key
FROM prev_transaction_diff
)
SELECT
	user_Id,
	id,
	created_at,
	SUM(streak_key) OVER (PARTITION BY user_id ORDER BY created_at) AS session_number
FROM transactions_streak_keys
WHERE transaction_cnt >= 3




---

## Task 2 — Z-Score Outlier Transactions per User

**Difficulty: 4/5**

**Business question:**  
For each user, calculate the mean and standard deviation of their transaction amounts. Then for each transaction, compute the z-score: `(amount - mean) / stddev`.

Return only transactions where the absolute z-score exceeds 2.0 — these are statistical outliers.

Exclude users where `stddev = 0` or `stddev IS NULL` (users with identical amounts or fewer than 2 transactions).

**Expected output columns:**  
`user_id, id, amount, mean_amount, stddev_amount, z_score`

All float columns rounded to 2 decimal places. Order by `ABS(z_score) DESC`.

WITH users_transactions AS (
SELECT 
	user_id,
	id,
	amount,
	AVG(amount) OVER (PARTITION BY t.user_id) AS mean_amount,
	STDDEV(amount) OVER (PARTITION BY t.user_id) AS stddev_amount
FROM crappy_data_db.transactions t
),
transactions_zscores AS (
SELECT 
	*,
	ROUND((amount - mean_amount) / stddev_amount, 2) AS z_score
FROM users_transactions
)
SELECT * FROM transactions_zscores
WHERE z_score >= 2.0
ORDER BY z_score DESC 


---

## Task 3 — ROWS vs RANGE: Cumulative Daily Amount

**Difficulty: 4/5**

**Business question:**  
For each transaction in `crappy_data_db.transactions`, compute two running totals of `amount` ordered by `created_at`, partitioned by `user_id`:

- `running_rows`: uses `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`
- `running_range`: uses `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`

Show both side by side. Find at least one user where the two columns differ — this happens when multiple transactions share the exact same `created_at` timestamp.

**Expected output columns:**  
`user_id, id, created_at, amount, running_rows, running_range`

Order by `user_id`, `created_at`, `id`.

**Note:** After writing the query, add a short comment (2–3 sentences) explaining WHY the two columns differ for ties.



WITH users_transactions AS (
SELECT 
	*,
	SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_rows_amount,
	SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_range_amount
FROM crappy_data_db.transactions t
)
SELECT * FROM users_transactions
ORDER BY user_id, created_at, id

Z czapy to query.
Ja ogólnie kumam różnicę między RANGE/ROWS - szczególnie w kontekście danego interwału czasu vs danej liczby operacji ma sense używanie jednego lub drugiego. Obecnie w tym zadaniu daje to NIC, bo nie zauważyłem ani jednej transakcji, która by się różniła.

Inaczej mówiąc zadanie jest na siłę zrobione, a nie tak żeby faktycnzie był sens tego używać. 



---

## Submission Instructions

Paste your queries below each task.
