# SQL Tasks — 2026-08-25 (Week 35, Day 1)

**Dataset:** transactions / orders / users  
**Focus:** Rozgrzewka — running total · TOP N cities by revenue

---

## Task 1 — Running Total per User
**Difficulty: 3/5**

**Business question:**  
For each user, show every transaction with a running cumulative total of their spending, ordered by `created_at`. Include `user_id`, `id`, `created_at`, `amount`, and `running_total`.

Only include users who have at least 3 transactions.

**Expected output columns:**  
`user_id, id, created_at, amount, running_total`

Order by `user_id`, `created_at`, `id`.

**Difficulty: 3/5**

SELECT
	user_id,
	id,
	created_at,
	amount,
	SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS running_total
FROM crappy_data_db.transactions t
ORDER BY user_id, created_at, id


Super easy, took me 1 minute literally, no thinker.


---

## Task 2 — Top 3 Cities by Order Revenue
**Difficulty: 3/5**

**Business question:**  
Find the top 3 cities by total order revenue. JOIN `users` to `orders`, group by `city`, sum `orders.amount`. Exclude NULL cities.

If there's a tie at position 3, include all tied cities.

**Expected output columns:**  
`city, total_revenue, rank`

Order by `rank`, `city`.

**Difficulty: 3/5**

WITH cities_revs AS (
SELECT
	u.city,
	SUM(o.amount) AS total_revenue
FROM crappy_data_db.users u
LEFT JOIN crappy_data_db.orders o ON u.id = o.user_id
WHERE u.city IS NOT NULL AND O.amount IS NOT null
GROUP BY u.city
),
cities_ranks AS (
SELECT 
	*,
	RANK() OVER (ORDER BY total_revenue DESC) AS rank
FROM cities_revs
)
SELECT * FROM cities_ranks
WHERE RANK <= 3
ORDER BY RANK, city

Same, too easy - but maybe a perfect way to come back from a holiday


---

## Submission Instructions

Paste your queries below each task.
