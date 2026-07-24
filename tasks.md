# SQL Tasks — Week 31 Day 5

**Generated:** 2026-07-24
**Dataset:** crappy_data
**Focus:** Filtered aggregation vs CASE WHEN, FULL OUTER JOIN

---

## Task 1: City-level Order vs Transaction Activity
**Difficulty: 3/5**

**Business question:**
Compare order activity and transaction activity by city. Some cities have users who ordered but never transacted, some transacted but never ordered, some both. Show the full picture — every city that appears in either source, with total order amount and total transaction amount. Where one side is missing, show 0.

Use a FULL OUTER JOIN. Do not use UNION.

**Expected output columns:**
`city, total_order_amount, total_transaction_amount`


WITH cities_orders_amounts AS (
SELECT 
	u.city,
	SUM(o.amount) AS total_order_amount
FROM crappy_data_db.users u
JOIN crappy_data_db.orders o ON u.id = o.user_id
WHERE u.city IS NOT NULL
GROUP BY u.city
),
cities_transactions_amounts AS (
SELECT 
	u.city,
	SUM(t.amount) AS total_transaction_amount
FROM crappy_data_db.users u
JOIN crappy_data_db.transactions t ON u.id = t.user_id
WHERE u.city IS NOT NULL
GROUP BY u.city
)
SELECT 
	c.city,
	COALESCE(c.total_order_amount, 0) AS total_order_amount,
	COALESCE(t.total_transaction_amount, 0) AS total_transaction_amount
FROM  cities_orders_amounts c
FULL JOIN cities_transactions_amounts t ON c.city = t.city
WHERE c.city IS NOT NULL

---

## Task 2: Conditional Aggregation — Two Approaches
**Difficulty: 4/5**

**Business question:**
For each transaction type, show: total amount, average amount, and — separately — the count and total amount of transactions above 500. 

Write this query **twice**:
- Version A: using `CASE WHEN` inside aggregation functions
- Version B: using `FILTER (WHERE ...)`

Then answer in a comment: is there any difference in how NULLs are handled between the two approaches?

**Expected output columns (both versions):**
`type, total_amount, avg_amount, count_above_500, total_above_500`


Version 1 - it's obviously longer, requires more preparation and isn't very convenient - I wouldn't use it with my current skillset


WITH transactions_filter AS (
SELECT 
	*,
	CASE WHEN amount > 500 THEN 1 ELSE 0 END AS above_500
FROM crappy_data_db.transactions t 
),
above_500_amts AS (
SELECT
TYPE,
SUM(amount) AS total_transactions_amount
FROM transactions_filter
WHERE above_500 = 1
GROUP BY TYPE
)
SELECT
	t.TYPE,
	COUNT(*) AS total_transactions,
	SUM(amount) AS total_amount,
	ROUND(AVG(amount), 2) AS avg_amount,
	SUM(above_500) AS count_above_500,
	a.total_transactions_amount AS total_above_500
FROM transactions_filter t 
JOIN above_500_amts a ON t.TYPE = a.type
GROUP BY t.TYPE, a.total_transactions_amount


------

Version 2 - smooth and easy


SELECT
	t.TYPE,
	COUNT(*) AS total_transactions,
	SUM(amount) AS total_amount,
	ROUND(AVG(amount), 2) AS avg_amount,
	COUNT(*) FILTER (WHERE amount > 500) AS count_above_500,
	SUM(amount) FILTER (WHERE amount > 500) AS total_above_500
FROM crappy_data_db.transactions t
GROUP BY t.type


Nulls are handled by SUM/AVG etc., but honestly I'm 100% sure there are no nulls here.


---

## Submission Instructions

Paste both queries and results below each task. For Task 2 include your NULL comment.
