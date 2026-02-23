# Daily SQL Practice Tasks

**Generated:** 2026-02-23
**Week 11, Day 1 Focus:** Warm-Up — Consolidation + Light Review

---

## Task 1: 3-Level Hierarchy — Transaction Types by Top Users

**Scenario:**
Build a 3-level hierarchy over transactions:
- Level 1: `'All Transactions'`
- Level 2: Distinct transaction types (dynamic, from `transactions`)
- Level 3: For each type, the 3 users with the highest total transaction amount (show user_id as text)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — type at Level 2, user_id at Level 3
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Pre-aggregate totals per user+type, then rank before the recursive CTE
- Termination condition required
- No hardcoded values

**Difficulty Rating:** 3/5


WITH RECURSIVE distinct_transaction_types AS (
SELECT DISTINCT TYPE FROM transactions
),
transaction_types_totals AS (
SELECT
	user_id,
	type,
	SUM(amount) AS total_transactions_amt
FROM transactions
GROUP BY user_id, TYPE
),
transaction_types_ranks AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY TYPE ORDER BY total_transactions_amt DESC) AS transaction_type_rank
FROM transaction_types_totals
),
transaction_types_top_three AS (
SELECT * FROM transaction_types_ranks
WHERE transaction_type_rank <= 3
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Transactions' AS name,
	NULL::TEXT AS parent_name,
	'All Transactions' AS PATH
UNION ALL
SELECT 
	H.LEVEL + 1,
	COALESCE(dtt.TYPE, ttt.user_id::TEXT),
	h.name,
	h.PATH || ' > ' || COALESCE(dtt.TYPE, ttt.user_id::TEXT)
FROM HIERARCHY h
LEFT JOIN distinct_transaction_types dtt ON h.LEVEL = 1
LEFT JOIN transaction_types_top_three ttt ON h.LEVEL = 2 AND ttt."type" = h."name"
WHERE h.LEVEL < 3
)
SELECT * FROM hierarchy


---

## Task 2: User Order Frequency Cohorts

**Scenario:**
The product team wants to understand how many orders users typically place. Group users into frequency buckets based on their total order count:

- `one_time`: exactly 1 order
- `occasional`: 2 to 4 orders
- `regular`: 5 to 9 orders
- `loyal`: 10 or more orders

For each bucket, show how many users fall in it and their average order value (avg of all order amounts for users in that bucket), rounded to 2 decimals.

**Expected Output Columns:**
- `frequency_bucket` (text)
- `user_count` (bigint)
- `avg_order_value` (numeric)

**Requirements:**
- Use `users` and `orders` tables
- Include users with 0 orders in `one_time`? No — only users who appear in `orders`
- Order by `user_count DESC`

**Difficulty Rating:** 3/5

WITH users_orders AS (
SELECT 
	user_id,
	COUNT(*) AS order_cnt
FROM orders
GROUP BY user_id
),
users_frequencies AS (
SELECT 
	*,
	CASE
		WHEN order_cnt = 1 THEN 'one_time'
		WHEN order_cnt > 1 AND order_cnt < 5 THEN 'occasional'
		WHEN order_cnt > 4 AND order_cnt < 10 THEN 'regular'
		WHEN order_cnt >= 10 THEN 'loyal'
	END AS frequency_bucket
FROM USERS_ORDERS
)
SELECT 
	uf.frequency_bucket,
	COUNT(uf.user_id) AS user_count,
	ROUND(AVG(o.amount)::numeric, 2) AS avg_order_value
FROM users_frequencies uf
JOIN orders o ON uf.user_id = o.user_id
GROUP BY uf.frequency_bucket
ORDER BY user_count DESC

There was literally no need to use users table, so I didn't do it and followed best practices.

---

## Task 3: Daily Session Trends with 3-Day Rolling Average

**Scenario:**
The analytics team wants a daily overview of platform engagement. For each day in the `user_sessions_daily` data, calculate:
- Total sessions across all users that day
- A 3-day rolling average of total sessions (current day + 2 preceding days), rounded to 1 decimal

**Expected Output Columns:**
- `date` (date)
- `total_daily_sessions` (bigint)
- `rolling_3d_avg` (numeric)

**Requirements:**
- Use `user_sessions_daily`
- Rolling average window: `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW`
- Order by `date ASC`

**Difficulty Rating:** 2/5

WITH dates_sessions AS (
SELECT 
	date,
	SUM(count_sessions) AS total_daily_sessions
FROM user_sessions_daily
GROUP BY date
)
SELECT 
	*,
	ROUND(AVG(total_daily_sessions) OVER (ORDER BY date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS rolling_avg_3d
FROM dates_sessions

Easy!

---

## Submission Instructions

1. Task 1 — Transaction type hierarchy (3/5)
2. Task 2 — Order frequency cohorts (3/5)
3. Task 3 — Daily session trends with rolling average (2/5)
