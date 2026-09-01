# SQL Tasks — 2026-09-02 (Week 36, Day 2)

**Dataset:** orders / transactions / users  
**Focus:** Funnel analysis · City pair affinity (self-join) · FIRST_VALUE per user

---

## Task 1 — Order → Transaction Funnel

**Difficulty: 4/5**

**Business question:**  
Calculate a simple conversion funnel:
- Step 1: total distinct users in the `users` table
- Step 2: distinct users who have placed at least one order
- Step 3: distinct users who have placed at least one order AND have at least one transaction

Show all three steps in one result set with: `step`, `user_count`, and `conversion_pct` (percentage of step 1 users who reached that step).

`conversion_pct` rounded to 2 decimal places.

**Expected output columns:**  
`step, user_count, conversion_pct`


WITH RECURSIVE users_cnt AS (
SELECT 
	COUNT(id) AS user_count 
FROM crappy_data_db.users u
),
orders_cnt AS (
SELECT 
	COUNT(DISTINCT(user_id)) AS orders_cnt
FROM crappy_data_db.orders o
),
combined_cnt AS (
SELECT 
	COUNT(DISTINCT(t.user_id)) AS t_ordered_cnt
FROM crappy_data_db.transactions t
JOIN crappy_data_db.orders o ON t.user_id = o.user_id 
),
recursive_cte AS (
SELECT
	1 AS step,
	u.user_count,
	100::NUMERIC AS conversion_pct
FROM users_cnt u
UNION ALL
SELECT
	r.step + 1,
	COALESCE(o.orders_cnt, c.t_ordered_cnt),
	COALESCE(ROUND(o.orders_cnt / r.user_count::numeric * 100, 2), ROUND(c.t_ordered_cnt / r.user_count::numeric * 100, 2))
FROM recursive_cte r
LEFT JOIN orders_cnt o ON r.step = 1
LEFT JOIN combined_cnt c ON r.step = 2
WHERE r.step <= 2
)
SELECT * FROM recursive_cte


It works :))





---

## Task 2 — City Pair Co-occurrence on Same Order Day (Self-Join)

**Difficulty: 5/5**

**Business question:**  
Find which pairs of cities most often appear together on the same calendar day — meaning: at least one user from city A and at least one user from city B both placed an order on the same date.

Join `users` to `orders` to get city + order date per user. Then self-join on `order_date` where `city_a < city_b` (to avoid duplicates). Count distinct dates where both cities appear.

Exclude NULL cities. Return the top 10 pairs by co-occurrence count.

**Expected output columns:**  
`city_a, city_b, cooccurrence_days`

Order by `cooccurrence_days DESC`, `city_a`.

WITH cities_days AS (
SELECT
	u1.city AS city_a,
	u1.created_at::DATE,
	u2.city AS city_b
FROM crappy_data_db.users u1
JOIN crappy_data_db.orders o1 ON u1.id = o1.user_id
JOIN crappy_data_db.orders o2 ON o1.created_at::DATE = o2.created_at::DATE
JOIN crappy_data_db.users u2 ON o2.user_id = u2.id
WHERE u2.city > u1.city
AND u1.city IS NOT NULL AND U2.CITY IS NOT NULL
)
SELECT 
	city_a,
	city_b,
	created_at,
	COUNT(*) AS cooccurence_days
FROM cities_days
GROUP BY city_a, city_b, created_at
ORDER BY cooccurence_days DESC, city_a

I struggled at first, but then I figured out if we use group by on dates and cities, the duplicates will take care of themselves on their own, so it's easy :).



---

## Task 3 — First vs Last Transaction Amount per User

**Difficulty: 4/5**

**Business question:**  
For each user, find the amount of their very first transaction and their very last transaction (by `created_at`). Calculate the difference: `last_amount - first_amount`.

Use `FIRST_VALUE` with appropriate ordering for both values.

Only include users with at least 2 transactions.

**Expected output columns:**  
`user_id, first_amount, last_amount, amount_diff`

Order by `ABS(amount_diff) DESC`.



WITH users_t_times AS (
SELECT 
	user_id,
	MIN(created_at) AS ft_time,
	MAX(created_at) AS lt_time
FROM crappy_data_db.transactions t
GROUP BY user_id
),
users_filter AS (
SELECT
	user_id,
	COUNT(*) AS t_cnt
FROM crappy_data_db.transactions t
GROUP BY user_id
HAVING COUNT(*) >= 2
)
SELECT 
	t.user_id,
	t2.amount AS first_amount,
	t3.amount AS last_amount,
	t3.amount - t2.amount AS amount_diff
FROM users_t_times t
JOIN users_filter u ON t.user_id = u.user_id
JOIN crappy_data_db.transactions t2
ON t.user_id = t2.user_id AND t2.created_at = t.ft_time
JOIN crappy_data_db.transactions t3
ON t.user_id = t3.user_id AND t3.created_at = t.lt_time
ORDER BY ABS(t3.amount - t2.amount) DESC


---

## Submission Instructions

Paste your queries below each task.
