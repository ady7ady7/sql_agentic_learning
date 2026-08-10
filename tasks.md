# SQL Tasks — 2026-08-10 (Week 34, Day 1)

**Dataset:** orders / transactions / users  
**Focus:** NTILE quartile analysis · Self-join with multi-condition HAVING · PERCENT_RANK

---

## Task 1 — Spending Quartiles
**Difficulty: 3/5**

**Business question:**  
Divide users into 4 spending quartiles based on their total transaction amount. For each quartile show:
- The quartile number (1 = lowest, 4 = highest)
- Number of users in that quartile
- Average total spending per user
- Min and max total spending in that quartile

Then add a second part: for each user in the **top quartile (4)**, show their `user_id` and total spending, ordered by spending DESC.

**Expected output columns:**

Part A: `quartile, user_count, avg_spending, min_spending, max_spending`

Part B: `user_id, total_spending`

**Difficulty: 3/5**


WITH users_totals AS (
SELECT
	user_id,
	SUM(amount) AS total_amount
FROM crappy_data_db.transactions t
GROUP BY user_id
),
users_quartiles AS (
SELECT 
	*,
	NTILE(4) OVER (ORDER BY total_amount) AS quartile
FROM users_totals
),
quartiles AS (
SELECT 
	quartile,
	COUNT(*) AS user_count,
	AVG(total_amount) AS avg_spending,
	MIN(total_amount) AS min_spending,
	MAX(total_amount) AS max_spending
FROM users_quartiles
GROUP BY quartile
)
SELECT 
	user_id,
	total_amount
FROM users_quartiles
WHERE quartile = 4
ORDER BY total_amount DESC


---

## Task 2 — Same-City Same-Month Pairs
**Difficulty: 4/5**

**Business question:**  
Find pairs of users from the **same city** who both placed at least one order in the **same calendar month**. For each such pair, show the city, the month, and how many orders each user placed that month.

Only include pairs where both users placed at least 2 orders in that shared month. Each pair should appear once (use `u1.id < u2.id`). Exclude users with NULL city.

**Expected output columns:**  
`city, order_month, user_id_1, orders_u1, user_id_2, orders_u2`

Order by `city`, `order_month`.

**Difficulty: 4/5**


WITH uo_months AS (
SELECT 
	*,
	o.id AS order_id,
	DATE_TRUNC('Month', o.created_at) AS order_month
FROM crappy_data_db.users u
JOIN crappy_data_db.orders o ON u.id = o.user_id
),
users_order_months AS (
SELECT 
	user_id AS user_id,
	city,
	order_month,
	order_id
FROM uo_months
WHERE city IS NOT NULL
),
users_monthly_agg AS (
SELECT
	user_id,
	city,
	order_month,
	COUNT(*) AS user_orders_month
FROM users_order_months 
GROUP BY user_id, order_month, city
)
SELECT 
	u1.city,
	u1.order_month,
	u1.user_id AS user_id_1,
	u1.user_orders_month AS orders_u1,
	u2.user_id AS user_id_2,
	u2.user_orders_month AS orders_u2
FROM users_monthly_agg u1
JOIN users_monthly_agg u2 ON u1.city = u2.city AND u1.order_month = u2.order_month
WHERE u1.user_id > u2.user_id AND u1.user_orders_month >= 2 AND u2.user_orders_month >= 2
ORDER BY city, order_month

---

## Task 3 — Transaction Amount Percentile Rank
**Difficulty: 5/5**

**Business question:**  
For each transaction, compute its `PERCENT_RANK` among all transactions **of the same type** — i.e. how does this transaction's amount rank relative to all other transactions of that type?

Then filter to only show transactions in the **top 10%** of their type (percent_rank >= 0.90).

Finally, for each type, show how many transactions made it into the top 10% and what their average amount is.

Two-part output:

**Part A:** Per transaction  
`id, user_id, type, amount, percent_rank`  
(only top 10% per type, ordered by type, percent_rank DESC)

**Part B:** Summary per type  
`type, top10_count, avg_top10_amount`  
(ordered by avg_top10_amount DESC)

**Difficulty: 5/5**

WITH transactions_perc_ranks AS (
SELECT
	*,
	PERCENT_RANK() OVER (PARTITION BY TYPE ORDER BY amount) AS percent_rank
FROM crappy_data_db.transactions t
)
SELECT 
	TYPE,
	COUNT(*) AS top_10_count,
	ROUND(AVG(amount), 2) AS avg_top10_amount
FROM transactions_perc_ranks 
WHERE percent_rank >= 0.9
GROUP BY TYPE
ORDER BY avg_top10_amount DESC

---

## Submission Instructions

Paste your queries below each task.
