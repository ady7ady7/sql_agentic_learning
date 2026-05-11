# Daily SQL Practice Tasks

**Generated:** 2026-05-11
**Week 22, Day 1 Focus:** crappy_data_db warm-up — GROUP BY + window RANK + NOT EXISTS

---

## Task 1: Order Stats per Country

**Scenario:**
The finance team wants a country-level summary of order activity.

For each country show:
- `country`
- `user_count` — number of distinct users from that country who placed at least one order
- `order_count` — total number of orders from users in that country
- `total_revenue` — sum of order amounts, rounded to 2 decimals
- `avg_order_value` — average order amount per order, rounded to 2 decimals

Exclude NULL countries and NULL amounts. Order by `total_revenue DESC`.

**Tables:** `crappy_data_db.orders`, `crappy_data_db.users`

**Difficulty Rating:** 3/5

WITH users_spending AS (
SELECT 
	u.country,
	SUM(o.amount) AS total_revenue,
	COUNT(DISTINCT(u.id)) AS user_count,
	COUNT(*) AS order_count,
	ROUND(AVG(o.amount)::numeric, 2) AS avg_order_value
FROM crappy_data_db.orders o
JOIN crappy_data_db.users u ON o.user_id = u.id
WHERE u.country IS NOT NULL AND O.amount IS NOT NULL
GROUP BY u.country
)
SELECT 
	*
FROM users_spending



---

## Task 2: RANK — Users by Spend Within Country

**Scenario:**
The marketing team wants to rank users by their total order spend within their country — to identify top spenders per region.

For each user show:
- `user_id`
- `country`
- `total_spend` — sum of order amounts, rounded to 2 decimals
- `country_rank` — RANK() of this user within their country by total_spend DESC

Exclude NULL countries and NULL amounts. Order by `country ASC`, `country_rank ASC`.

**Tables:** `crappy_data_db.orders`, `crappy_data_db.users`

**Difficulty Rating:** 4/5


WITH users_spending AS (
SELECT 
	o.user_id,
	u.country,
	SUM(o.amount) AS total_spend
FROM crappy_data_db.orders o
JOIN crappy_data_db.users u ON o.user_id = u.id
WHERE u.country IS NOT NULL
GROUP BY o.user_id, u.country
)
SELECT 
	*,
	RANK() OVER (PARTITION BY country ORDER BY total_spend DESC)
FROM users_spending
ORDER BY country, rank





---

## Task 3: NOT EXISTS — Users Who Never Opened a Support Ticket

**Scenario:**
The support team wants to find users who have never opened a chat ticket — to understand what share of the user base has never contacted support.

Use `NOT EXISTS`. Start from `crappy_data_db.users`, check against `crappy_data_db.chat_tickets`.

**Expected Output Columns:** `user_id`

**Tables:** `crappy_data_db.users`, `crappy_data_db.chat_tickets`

**Order by:** `user_id ASC`

**Difficulty Rating:** 3/5


SELECT 
	u.id AS user_id
FROM crappy_data_db.users u 
WHERE NOT EXISTS (
	SELECT 
		ct.user_id
	FROM crappy_data_db.chat_tickets ct
	WHERE ct.user_id = u.id
)



---

## Submission Instructions

1. Task 1 — Country-level order stats (3/5)
2. Task 2 — RANK users by spend within country (4/5)
3. Task 3 — NOT EXISTS: users with no support tickets (3/5)
