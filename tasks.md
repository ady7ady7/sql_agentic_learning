# SQL Tasks — 2026-08-27 (Week 35, Day 3)

**Dataset:** transactions / users / job_db  
**Focus:** LEAD with offset · Anti-join · job_db seniority trend

---

## Task 1 — Days to Next Transactions (LEAD with offset)

**Difficulty: 4/5**

**Business question:**  
For each user, show every transaction with the number of days until their next transaction (offset 1) and the number of days until the transaction after that (offset 2).

Only include users who have at least 4 transactions (so offset 2 always has a value).

`days_to_next` and `days_to_next2` should be NULL when there is no subsequent transaction — do not filter those rows out.

Use `DATE_PART('epoch', ...)` or `EXTRACT(epoch FROM ...)` for day calculations.

**Expected output columns:**  
`user_id, id, created_at, days_to_next, days_to_next2`

Order by `user_id`, `created_at`.

WITH users_transactions AS (
SELECT 
	*,
	LEAD(t.created_at) OVER (PARTITION BY user_id ORDER BY created_at) AS next_transaction_date,
	LEAD(t.created_at, 2) OVER (PARTITION BY user_id ORDER BY created_at) AS next_2nd_transaction_date
FROM crappy_data_db.transactions t
),
transactions_filter AS (
SELECT 
	user_id,
	COUNT(*) AS transaction_cnt
FROM users_transactions 
GROUP BY user_id
HAVING COUNT(*) >= 4
)
SELECT 
	u.user_id,
	u.created_at,
	ROUND(EXTRACT(EPOCH FROM u.next_transaction_date - u.created_at) / 60, 1) AS minutes_to_next,
	ROUND(EXTRACT(EPOCH FROM u.next_2nd_transaction_date - u.created_at) / 60, 1) AS minutes_to_next2
FROM users_transactions u
JOIN transactions_filter t ON u.user_id = t.user_id
ORDER BY user_id, created_at

Not a big deal, but I've changed the days to minutes, because the gaps are veeery small, using days would result in getting a mass of 0s, no useful observations AT ALL.

Don't even dare to take way points for that.


---

## Task 2 — Users with Transactions but No Orders (Anti-join)

**Difficulty: 4/5**

**Business question:**  
Find all users who have at least one transaction in `crappy_data_db.transactions` but have never placed any order in `crappy_data_db.orders`.

Write the query using `NOT EXISTS`. Then write it again using `LEFT JOIN ... WHERE IS NULL`.

Both queries should return the same result: `user_id`, ordered ascending.

**Expected output columns:**  
`user_id`

EASY.
AND once again, I refuse to do two queries, it's chaotic and useless as fuck to mix these every time. In every time in a similar context I'll simply use NOT EXISTS:

SELECT 
	DISTINCT(t.user_id)
FROM crappy_data_db.transactions t
WHERE NOT EXISTS
	(SELECT o.user_id FROM crappy_data_db.orders o WHERE o.user_id = t.user_id)


Don't you dare to take away pts for that.
---

## Task 3 — Dominant Seniority per Month (job_db)

**Difficulty: 5/5**

**Business question:**  
For each month in the dataset, find which seniority level had the most job offers. Show the month, seniority name, offer count for that seniority, and its rank within that month.

Only include months where there are at least 3 distinct seniority levels with offers.

If two seniority levels tie for rank 1 in a month, include both.

Only include rows where `data_wystawienia` IS NOT NULL and `seniority_id` IS NOT NULL.

**Expected output columns:**  
`month, seniority_name, offer_count, rank`

Order by `month`, `rank`, `seniority_name`.


WITH offers_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', data_wystawienia) AS month_
FROM job_db.oferty o
WHERE o.data_wystawienia IS NOT NULL AND o.seniority_id IS NOT NULL
),
seniorities_offers_months AS (
SELECT 
	om.month_,
	om.seniority_id,
	s.nazwa AS seniority_name,
	COUNT(*) AS offer_count
FROM offers_months om
JOIN job_db.seniority s ON om.seniority_id = s.id
GROUP BY om.month_, om.seniority_id, s.nazwa
),
seniorities_monthly_ranks AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY month_ ORDER BY offer_count DESC) AS RANK
FROM seniorities_offers_months
),
month_filter AS (
SELECT 
	month_,
	COUNT(*) AS seniority_count
FROM seniorities_monthly_ranks
GROUP BY month_
HAVING count(*) >= 3
)
SELECT 
	s.month_,
	s.seniority_name,
	s.offer_count,
	s.rank
FROM seniorities_monthly_ranks s
JOIN month_filter m ON s.month_ = m.month_
WHERE s.RANK = 1
ORDER BY month_


Not a big deal, nice task :)).
I've moved ordering to the last CTE, so it's cheaper in terms of memory usage.



---

## Submission Instructions

Paste your queries below each task.
