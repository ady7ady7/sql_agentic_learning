# SQL Tasks — 2026-09-04 (Week 36, Day 5)

**Dataset:** transactions / users  
**Focus:** LATERAL joins · NTILE + conditional aggregation

---

## LATERAL — Introduction

A regular subquery in a JOIN cannot reference columns from the table on the left side of the join. `LATERAL` changes that — it lets the subquery reference the outer row's columns, effectively running "for each row on the left, execute this subquery."

**Classic use case: "top N per group".**

Without LATERAL (window function + filter):
```sql
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY amount DESC) AS rn
    FROM transactions
) t WHERE rn <= 3
```

With LATERAL (often clearer, sometimes faster since the DB can use an index and stop at LIMIT):
```sql
SELECT u.id AS user_id, t.*
FROM users u
CROSS JOIN LATERAL (
    SELECT id, amount, created_at
    FROM transactions
    WHERE transactions.user_id = u.id
    ORDER BY amount DESC
    LIMIT 3
) t
```

The key part: `WHERE transactions.user_id = u.id` inside the subquery references `u.id` from the outer FROM — impossible in a regular JOIN/subquery, only LATERAL allows it.

**Second use case: "find the nearest matching row" (e.g. first tick after a timestamp).**
```sql
CROSS JOIN LATERAL (
    SELECT ts_event, price
    FROM ticks t
    WHERE t.trade_date = m.trade_date AND t.ts_event >= m.some_timestamp
    ORDER BY t.ts_event
    LIMIT 1
) later
```

Use `LEFT JOIN LATERAL ... ON true` instead of `CROSS JOIN LATERAL` when you want to keep the outer row even if the subquery returns nothing.

---

## Task 1 — Top 3 Transactions per User (LATERAL)

**Difficulty: 4/5**

**Business question:**  
For each user, find their top 3 transactions by `amount` (descending). Use `CROSS JOIN LATERAL` — the subquery should reference the outer user's `id` directly (correlated), ordered and limited inside the LATERAL subquery.

Only include users who have at least 3 transactions.

**Expected output columns:**  
`user_id, id, amount, created_at`

Order by `user_id`, `amount DESC`.


SELECT 
	u.id AS user_id, t.* AS user_id
FROM crappy_data_db.users u
CROSS JOIN LATERAL (
	SELECT id, amount, created_at
	FROM crappy_data_db.transactions t 
	WHERE t.user_id = u.id
	ORDER BY amount DESC
	LIMIT 3
) t


---

## Task 2 — Spending Quartiles by City (NTILE + Conditional Aggregation)

**Difficulty: 4/5**

**Business question:**  
For each user, calculate their total transaction amount and assign them to a spending quartile using `NTILE(4)` (1 = lowest spenders, 4 = highest).

Then, for each city, count how many users fall into each quartile using conditional aggregation (`FILTER` or `CASE WHEN`).

Exclude users with NULL city.

**Expected output columns:**  
`city, quartile_1_count, quartile_2_count, quartile_3_count, quartile_4_count`

Order by `city`.

WITH user_spendings AS (
SELECT 
	user_id,
	SUM(amount) AS total_spent
FROM crappy_data_db.transactions t
GROUP BY user_id
),
spending_quartiles AS (
SELECT 
	*,
	ntile(4) OVER (ORDER BY total_spent) AS spending_quartile
FROM user_spendings
)
SELECT
	city,
	COUNT(*) FILTER (WHERE spending_quartile = 1) AS quartile_1_count,
	COUNT(*) FILTER (WHERE spending_quartile = 2) AS quartile_2_count,
	COUNT(*) FILTER (WHERE spending_quartile = 3) AS quartile_3_count,
	COUNT(*) FILTER (WHERE spending_quartile = 4) AS quartile_4_count
FROM spending_quartiles s
JOIN crappy_data_db.users u ON s.user_id = u.id
WHERE u.city IS NOT NULL
GROUP BY city


Ogarnięte

---

## Submission Instructions

Paste your queries below each task.
