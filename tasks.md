# Daily SQL Practice Tasks

**Generated:** 2026-05-12
**Week 22, Day 2 Focus:** EPOCH ticket resolution + cohort LEFT JOIN + STDDEV cross-table analysis

---

## Task 1: Ticket Resolution Time per Priority

**Scenario:**
The support team wants to know how long tickets typically stay open before being updated, broken down by priority.

For each priority show:
- `priority`
- `ticket_count`
- `avg_hours_open` — average time in hours between `created_at` and `updated_at`, rounded to 1 decimal

**Tables:** `crappy_data_db.chat_tickets`

**Requirements:**
- Use `EXTRACT(EPOCH FROM (updated_at - created_at)) / 3600` for hours
- Order by `avg_hours_open DESC`

**Difficulty Rating:** 3/5

SELECT 
	priority,
	COUNT(*) AS ticket_count,
	ROUND(EXTRACT(EPOCH FROM AVG(updated_at - created_at)/ 3600), 1) AS avg_hours_open
FROM crappy_data_db.chat_tickets ct 
GROUP BY priority
ORDER BY avg_hours_open DESC

---

## Task 2: Cohort Retention — First Order in 2025, Returned Within 3 Months

**Scenario:**
The growth team wants to know which users placed their first ever order in 2025 and then came back to order again within 3 months.

For each user show:
- `user_id`
- `first_order_date` — their first ever order date (not truncated to month, exact date)
- `returned_within_3_months` — boolean, true if they placed any order where `created_at >= first_order_date + INTERVAL '1 month'` AND `created_at < first_order_date + INTERVAL '4 months'`

Only include users whose first order was in 2025.

**Tables:** `crappy_data_db.orders`

**Requirements:**
- CTE 1: `MIN(created_at)` per user for first order date
- Final SELECT: LEFT JOIN back to `orders` on the 3-month window condition
- Derive boolean from whether the joined order IS NOT NULL
- Use `DISTINCT` on user_id in the final SELECT to avoid duplicates
- Order by `user_id ASC`

**Difficulty Rating:** 4/5

WITH users_first_order_dates AS (
SELECT 
	user_id,
	MIN(created_at) AS first_order_date
FROM crappy_data_db.orders o
GROUP BY user_id
),
first_orders_joined AS (
SELECT 
	ufod.user_id,
	ufod.first_order_date,
	o.created_at
FROM users_first_order_dates ufod
LEFT JOIN crappy_data_db.orders o ON o.user_id = ufod.user_id AND o.created_at >= ufod.first_order_date + INTERVAL '1 Month' AND o.created_at <= ufod.first_order_date + INTERVAL '3 Month'
WHERE EXTRACT('YEAR' FROM ufod.first_order_date) = 2025
GROUP BY ufod.user_id, ufod.first_order_date, o.created_at
)
SELECT 
	DISTINCT user_id,
	first_order_date,
	CASE WHEN created_at IS NULL THEN FALSE ELSE TRUE END AS returned_within_3_months
FROM first_orders_joined
ORDER BY user_id



---

## Task 3: STDDEV + Cross-Table — High-Volatility Transaction Users vs Order Spend

**Scenario:**
The risk team hypothesises that users with highly volatile transaction amounts also tend to spend more on orders. Test this by computing transaction volatility per user, then joining to their total order spend.

For each user who has both transactions and orders show:
- `user_id`
- `stddev_amount` — standard deviation of transaction amounts, rounded to 2 decimals
- `volatility` — `'high'` if stddev > 300, `'medium'` if > 150, `'low'` otherwise
- `total_order_spend` — sum of order amounts, rounded to 2 decimals

Order by `stddev_amount DESC`.

**Tables:** `crappy_data_db.transactions`, `crappy_data_db.orders`

**Requirements:**
- CTE 1: STDDEV per user from transactions (exclude NULL amounts)
- CTE 2: total order spend per user from orders (exclude NULL amounts)
- JOIN the two CTEs on user_id

**Difficulty Rating:** 5/5


WITH users_stds AS (
SELECT 
	user_id,
	round(STDDEV(amount) OVER (PARTITION BY user_id)::NUMERIC, 2) AS stddev_amount 
FROM crappy_data_db.orders
WHERE amount IS NOT NULL
GROUP BY user_id, amount
),
users_totals AS (
SELECT 
	user_id,
	sum(amount)
FROM crappy_data_db.orders o
GROUP BY user_id
)
SELECT 
	DISTINCT ut.user_id, 
	ut.sum AS total_order_spend, 
	us.stddev_amount,
	CASE WHEN us.stddev_amount > 300 THEN 'high' WHEN us.stddev_amount > 150 THEN 'medium' ELSE 'low' END AS total_order_spend
FROM users_TOTALS ut
LEFT JOIN users_stds US ON UT.user_id = US.user_id
WHERE US.stddev_amount IS NOT NULL
ORDER BY stddev_amount DESC


---

## Submission Instructions

1. Task 1 — EPOCH ticket resolution time per priority (3/5)
2. Task 2 — Cohort LEFT JOIN: first 2025 order + 3-month return (4/5)
3. Task 3 — STDDEV transaction volatility joined to order spend (5/5)
