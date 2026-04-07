# Daily SQL Practice Tasks

**Generated:** 2026-04-07
**Week 17, Day 1 Focus:** Light review — JOINs, GROUP BY, HAVING, basic window functions

---

## Task 1: Multi-Table JOIN — Order Revenue by Country and Gender

**Scenario:**
The marketing team wants a simple breakdown of total order revenue and order count, grouped by country and gender. They want to see which country + gender segments are the most valuable.

**Expected Output Columns:**
- `country` (varchar)
- `gender` (varchar)
- `order_count` (integer)
- `total_revenue` (numeric) — sum of order amounts, rounded to 2 decimals

**Requirements:**
- Use `orders` JOIN `users`
- Exclude rows where country IS NULL, gender IS NULL, or amount IS NULL
- Only include segments with at least 10 orders (HAVING)
- Order by `total_revenue DESC`

**Difficulty Rating:** 2/5

SELECT 
	u.country,
	u.gender,
	COUNT(o.id) AS order_count,
	SUM(o.amount) AS total_revenue
FROM crappy_data_db.orders o
JOIN crappy_data_db.users u ON o.user_id = u.id
WHERE u.country IS NOT NULL AND u.gender IS NOT NULL AND o.amount IS NOT NULL
GROUP BY u.country, u.gender
ORDER BY total_revenue DESC

Values were already rounded as intended

---

## Task 2: HAVING with Multiple Conditions — Active High-Value Users

**Scenario:**
The retention team wants to find users who are both frequent and high-spending: at least 5 orders AND average order value above 300. They also want to know the date of their most recent order.

**Expected Output Columns:**
- `user_id` (integer)
- `order_count` (integer)
- `avg_order_value` (numeric) — rounded to 2 decimals
- `total_spent` (numeric) — rounded to 2 decimals
- `last_order_date` (timestamp)

**Requirements:**
- Use `orders` only
- Exclude NULL amounts
- HAVING: order_count >= 5 AND avg_order_value > 300
- Order by `total_spent DESC`

**Difficulty Rating:** 2/5

WITH user_orders_metrics AS (
SELECT 
	user_id,
	COUNT(*) AS order_cnt,
	round(AVG(amount)::NUMERIC, 2) AS avg_order_amt,
	SUM(amount) AS total_spent,
	MAX(created_at) AS last_order_date
FROM crappy_data_db.orders o
WHERE amount IS NOT NULL
GROUP BY user_id
)
SELECT * FROM user_orders_metrics
WHERE order_cnt >= 5 AND avg_order_amt > 300
ORDER BY total_spent DESC

---

## Task 3: Window Function Review — Rank Users by Sessions Within City

**Scenario:**
The engagement team wants to rank users by their total session count, partitioned by city — so they can see who the most active user is within each city.

**Expected Output Columns:**
- `user_id` (integer)
- `city` (varchar)
- `total_sessions` (integer) — sum of count_sessions across all dates
- `city_rank` (integer) — RANK() within city, 1 = most active

**Requirements:**
- Use `user_sessions_daily` JOIN `users`
- Exclude rows where city IS NULL
- Only show users where `city_rank = 1` (top user per city)
- Order by `city ASC`

**Difficulty Rating:** 3/5

WITH users_cities_session_cnts AS (
SELECT 
	usd.user_id,
	u.city,
	SUM(usd.count_sessions) AS total_sessions
FROM crappy_data_db.user_sessions_daily usd
JOIN crappy_data_db.users u ON usd.user_id = u.id
WHERE U.CITY IS NOT null
GROUP BY USD.user_id, u.city
),
sessions_city_ranks AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY city ORDER BY total_sessions DESC) AS city_rank
FROM users_cities_session_cnts
)
SELECT * FROM sessions_city_ranks
WHERE city_rank = 1
ORDER BY city


---

## Submission Instructions

1. Task 1 — JOIN + GROUP BY + HAVING: order revenue by country and gender (2/5)
2. Task 2 — HAVING with multiple conditions: active high-value users (2/5)
3. Task 3 — RANK() partitioned by city, filter to top user per city (3/5)
