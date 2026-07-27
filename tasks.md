# SQL Tasks — Week 32 Day 1

**Generated:** 2026-07-27
**Dataset:** crappy_data
**Focus:** Anti-join patterns, window functions, retention cohort

---

## Task 1: Anti-join — Users Who Never Ordered
**Difficulty: 3/5**

**Business question:**
Find all users who have never placed an order. Write three separate versions of this query — NOT IN, NOT EXISTS, LEFT JOIN ... WHERE IS NULL — and add a short comment on which fails or behaves unexpectedly when NULLs are present in the joining column.

**Expected output columns:**
`user_id` (all three versions should return the same result)


SELECT 
	u.id AS user_id
FROM crappy_data_db.users u
WHERE NOT EXISTS (
SELECT
	o.user_id
FROM crappy_data_db.orders o
WHERE o.user_id = u.id
)


I'll just give you one version - no need to fuck around with three - literally just cluttering my memory with chaos this way.





---

## Task 2: Transaction Percentiles by Type
**Difficulty: 4/5**

**Business question:**
For each transaction, show its type, amount, and within its type: its percentile rank, which quartile it falls in, and the first (lowest) and last (highest) transaction amount in that type. No aggregation — keep one row per transaction.

**Expected output columns:**
`id, type, amount, percent_rank, quartile, min_in_type, max_in_type`

SELECT 
	id,
	TYPE,
	amount,
	percent_rank() OVER (PARTITION BY TYPE ORDER BY amount) AS percent_rank,
	ntile(4) OVER (PARTITION BY TYPE ORDER BY amount) AS type_quartile,
	FIRST_VALUE(amount) OVER (PARTITION BY TYPE ORDER BY amount) AS min_in_type,
	FIRST_VALUE(amount) OVER (PARTITION BY TYPE ORDER BY amount DESC) AS max_in_type
FROM crappy_data_db.transactions t

Easy.


---

## Task 3: User Retention Cohort — 30/60/90 Days
**Difficulty: 5/5**

**Business question:**
Group users by their registration month (cohort). For each cohort, show how many users placed their first order within 30, 60, and 90 days of registration — and what % of the cohort each threshold represents. Users who never ordered count as 0 across all thresholds.

**Expected output columns:**
`cohort_month, cohort_size, converted_30d, pct_30d, converted_60d, pct_60d, converted_90d, pct_90d`

WITH users_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
FROM crappy_data_db.users u
)
SELECT 
	um.month_,
	COUNT(DISTINCT(um.id)) AS cohort_size,
	COUNT(DISTINCT(um.id)) FILTER (WHERE o.created_at < um.month_ + INTERVAL '1' Month) AS converted_30d,
	ROUND(COUNT(DISTINCT(um.id)) FILTER (WHERE o.created_at < um.month_ + INTERVAL '1' Month) / COUNT(DISTINCT(um.id))::NUMERIC * 100, 2) AS pct_30d,
	COUNT(DISTINCT(um.id)) FILTER (WHERE o.created_at < um.month_ + INTERVAL '2' Month) AS converted_60d,
	ROUND(COUNT(DISTINCT(um.id)) FILTER (WHERE o.created_at < um.month_ + INTERVAL '2' Month) / COUNT(DISTINCT(um.id))::NUMERIC * 100, 2) AS pct_60d,
	COUNT(DISTINCT(um.id)) FILTER (WHERE o.created_at < um.month_ + INTERVAL '3' Month) AS converted_90d,
	ROUND(COUNT(DISTINCT(um.id)) FILTER (WHERE o.created_at < um.month_ + INTERVAL '3' Month) / COUNT(DISTINCT(um.id))::NUMERIC * 100, 2) AS pct_90d
FROM users_months um
LEFT JOIN crappy_data_db.orders o ON um.id = o.user_id
GROUP BY um.month_



---

## Submission Instructions

Paste your queries and results below each task.
