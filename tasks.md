# SQL Tasks — 2026-09-17 (Week 38, Day 4)

**Dataset:** crappy_data_db  
**Focus:** Cohort retention rework · Window frame EXCLUDE CURRENT ROW · INTERSECT / EXCEPT

---

## Task 1 — Cohort Retention Rework: Correct Cohort Definition

**Difficulty: 4/5**

**Business question:**  
Same as before: for each user, find their cohort month — this time defined correctly as the month of their FIRST ORDER (`MIN(orders.created_at)` per user), not their registration date. Then check, per cohort:
- how many users returned with at least one order in month +1
- how many users returned with at least one order in month +2

`cohort_size` must count only users who have at least one order (i.e. users present in the first-order CTE), not all rows in `users`.

**Expected output columns:**  
`cohort_month, cohort_size, returned_month1, retention_month1_pct, returned_month2, retention_month2_pct`

Both percentage columns rounded to 2 decimals. Order by `cohort_month`.


WITH users_cohorts AS (
SELECT 
	o.user_id,
	date_trunc('Month', MIN(o.created_at)) AS cohort_month
FROM crappy_data_db.orders o
GROUP BY user_id
)
SELECT 
	u.cohort_month,
	COUNT(DISTINCT(u.user_id)) AS cohort_size,
	COUNT(DISTINCT(u.user_id)) FILTER (WHERE o1.created_at IS NOT NULL) AS returned_month1,
	ROUND(COUNT(DISTINCT(o1.user_id)) FILTER (WHERE o1.created_at IS NOT NULL) / COUNT(DISTINCT(u.user_id))::NUMERIC * 100, 2) AS retention_month1_pct,
	COUNT(DISTINCT(u.user_id)) FILTER (WHERE o2.created_at IS NOT NULL) AS returned_month2,
	ROUND(COUNT(DISTINCT(o2.user_id)) FILTER (WHERE o2.created_at IS NOT NULL) / COUNT(DISTINCT(u.user_id))::NUMERIC * 100, 2) AS retention_month1_pct
FROM users_cohorts u
LEFT JOIN crappy_data_db.orders o1 ON u.user_id = o1.user_id AND DATE_TRUNC('Month', o1.created_at) >= DATE_TRUNC('Month', u.cohort_month) + INTERVAL '1 Month' AND DATE_TRUNC('Month', o1.created_at) < DATE_TRUNC('Month', u.cohort_month) + INTERVAL '2 Month'
LEFT JOIN crappy_data_db.orders o2 ON u.user_id = o2.user_id AND DATE_TRUNC('Month', o2.created_at) >= DATE_TRUNC('Month', u.cohort_month) + INTERVAL '2 Month' AND DATE_TRUNC('Month', o2.created_at) < DATE_TRUNC('Month', u.cohort_month) + INTERVAL '3 Month'
GROUP BY U.cohort_month
ORDER BY cohort_month





---

## Window Frame EXCLUDE — Introduction

By default, a window frame like `ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING` includes the current row in the sum/average. `EXCLUDE CURRENT ROW` removes just the current row from that frame — useful for "compare this row to its neighbors, without itself pulling the average toward itself."

```sql
-- Sum of the 2 rows before and 2 rows after, NOT including the current row's own amount:
SUM(amount) OVER (
    PARTITION BY user_id
    ORDER BY created_at
    ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING
    EXCLUDE CURRENT ROW
) AS neighbors_sum
```

Other options: `EXCLUDE GROUP` (excludes peers with the same ORDER BY value), `EXCLUDE TIES` (keeps the current row but excludes other peers), `EXCLUDE NO OTHERS` (the default — excludes nothing).

---

## Task 2 — Local Neighbor Average (EXCLUDE CURRENT ROW)

**Difficulty: 5/5**

**Business question:**  
For each user's transactions (ordered by `created_at`), calculate the average `amount` of the 2 transactions before and 2 transactions after the current one — excluding the current transaction's own amount. Then flag whether the current transaction's amount is above or below that local neighbor average.

Only include users with at least 5 transactions.

**Expected output columns:**  
`user_id, id, created_at, amount, neighbors_avg, above_neighbors_avg`

`neighbors_avg` rounded to 2 decimals. `above_neighbors_avg` is boolean (`amount > neighbors_avg`). Order by `user_id`, `created_at`.



WITH transactions_nbors AS (
SELECT 
	*,
	ROUND(avg(amount) OVER (
	PARTITION BY user_id 
	ORDER BY created_at 
	ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING
	EXCLUDE CURRENT ROW
	), 2) AS neighbors_avg
FROM crappy_data_db.transactions t
)
SELECT 
	*,
	amount > neighbors_avg AS above_neighbors_avg
FROM transactions_nbors


---

## Task 3 — Users With Both vs Users With Transactions Only (INTERSECT / EXCEPT)

**Difficulty: 4/5**

**Business question:**  
Using set operators (not JOIN or NOT EXISTS):
- Find `user_id`s that appear in BOTH `transactions` and `orders` — use `INTERSECT`.
- Find `user_id`s that appear in `transactions` but NOT in `orders` — use `EXCEPT`.

Return both result sets stacked together with a label column so they're distinguishable in one result.

**Expected output columns:**  
`user_id, category` (`category` = `'has_both'` or `'transactions_only'`)

Order by `category`, `user_id`.


SELECT DISTINCT(t.user_id)  FROM crappy_data_db.transactions t 
INTERSECT
SELECT DISTINCT(o.user_id)  FROM crappy_data_db.orders o


WITH users_both AS (
SELECT DISTINCT(t.user_id)  FROM crappy_data_db.transactions t 
INTERSECT
SELECT DISTINCT(o.user_id)  FROM crappy_data_db.orders o
),
users_transactions_only AS (
SELECT DISTINCT(t.user_id)  FROM crappy_data_db.transactions t 
EXCEPT
SELECT DISTINCT(o.user_id) FROM crappy_data_db.orders o
),
users_unionized AS (
SELECT 
	*
FROM users_both
UNION ALL
SELECT
	*
FROM users_transactions_only
)
SELECT 
	u.user_id,
	CASE WHEN u1.user_id IS NOT NULL THEN 'has_both' ELSE 'transactions_only' END AS category
FROM users_unionized u
LEFT JOIN users_both u1 ON u.user_id = u1.user_id
LEFT JOIN users_transactions_only u2 ON u.user_id = u2.user_id
ORDER BY category, user_id


It wasn't taht obvious at first, but I managed to get it :))


---

## Submission Instructions

Paste your queries below each task.
