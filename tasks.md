# Daily SQL Practice Tasks

**Generated:** 2026-03-19
**Week 14, Day 4 Focus:** Time-Proximity Sessions (New Context) + PIVOT Reinforcement + Anti-Join Complex

---

## Task 1: Time-Proximity Gaps-and-Islands — Trade Clustering

**Scenario:**
A trading desk wants to group trades into "bursts" — clusters of consecutive trades where each trade occurs within 30 minutes of the previous one for the same user. If the gap exceeds 30 minutes, it's a new burst.

Use this inline data:

```sql
WITH trades (user_id, trade_time, amount) AS (
    VALUES
    (1, '2024-01-15 09:02'::timestamp, 1500.00),
    (1, '2024-01-15 09:18'::timestamp, 2300.00),
    (1, '2024-01-15 09:45'::timestamp, 800.00),
    (1, '2024-01-15 10:20'::timestamp, 4100.00),
    (1, '2024-01-15 10:48'::timestamp, 950.00),
    (1, '2024-01-15 11:30'::timestamp, 3200.00),
    (2, '2024-01-15 08:55'::timestamp, 600.00),
    (2, '2024-01-15 09:10'::timestamp, 1200.00),
    (2, '2024-01-15 09:38'::timestamp, 900.00),
    (2, '2024-01-15 11:05'::timestamp, 2800.00)
)
```

**Expected Output Columns:**
- `user_id` (integer)
- `burst_id` (bigint) — sequential burst number per user (1, 2, 3...)
- `burst_start` (timestamp)
- `burst_end` (timestamp)
- `trade_count` (bigint)
- `total_amount` (numeric) — sum of amounts in the burst

**Requirements:**
- Gap threshold: 30 minutes
- Use LAG → is_new_burst flag → SUM() OVER session key pattern
- `burst_id` should be sequential per user (1, 2, 3...) — use RANK() or DENSE_RANK() over session_key
- Order by `user_id ASC`, `burst_start ASC`

**Difficulty Rating:** 4/5

WITH trades (user_id, trade_time, amount) AS (
    VALUES
    (1, '2024-01-15 09:02'::timestamp, 1500.00),
    (1, '2024-01-15 09:18'::timestamp, 2300.00),
    (1, '2024-01-15 09:45'::timestamp, 800.00),
    (1, '2024-01-15 10:20'::timestamp, 4100.00),
    (1, '2024-01-15 10:48'::timestamp, 950.00),
    (1, '2024-01-15 11:30'::timestamp, 3200.00),
    (2, '2024-01-15 08:55'::timestamp, 600.00),
    (2, '2024-01-15 09:10'::timestamp, 1200.00),
    (2, '2024-01-15 09:38'::timestamp, 900.00),
    (2, '2024-01-15 11:05'::timestamp, 2800.00)
),
users_trades AS (
SELECT 
	*,
	LAG(trade_time) OVER (PARTITION BY user_id ORDER BY trade_time) AS prev_trade_time
FROM trades
),
users_streak_starts AS (
SELECT 
	*,
	CASE WHEN prev_trade_time IS NULL OR trade_time - prev_trade_time > INTERVAL '30 MINUTES' THEN 1 ELSE 0 END AS is_start 
FROM users_trades
),
users_streak_keys AS (
SELECT 
	*,
	SUM(is_start) OVER (PARTITION BY user_id ORDER BY trade_time) AS burst_id
FROM users_streak_starts
)
SELECT
	user_id,
	burst_id,
	MIN(trade_time) AS burst_start,
	MAX(trade_time) AS burst_end,
	COUNT(*) AS trade_count,
	SUM(amount) AS total_amount
FROM users_streak_keys _keys
GROUP BY user_id, burst_id
ORDER BY user_id, burst_start


Works again, but oh man, that pattern feels so unintuitive now. I really needed to check the yesterday's progress to use it again. We need to practice it more, as it's golden

---

## Task 2: PIVOT — User Age Group Revenue Breakdown

**Scenario:**
The marketing team wants a monthly revenue breakdown by user age group. Classify users into age buckets and pivot them as columns.

Age groups:
- `under_30`: age < 30
- `age_30_to_50`: age BETWEEN 30 AND 50
- `over_50`: age > 50
- `unknown`: age IS NULL

**Expected Output Columns:**
- `month` (date) — truncated to month
- `under_30` (numeric) — total order revenue, rounded to 2 decimals
- `age_30_to_50` (numeric)
- `over_50` (numeric)
- `unknown` (numeric)
- `total_revenue` (numeric)

**Requirements:**
- Use `users` and `orders` tables
- Join on `user_id`
- Use `SUM(amount) FILTER (WHERE ...)` pattern with CASE for age bucketing
- Order by `month ASC`

**Difficulty Rating:** 4/5

WITH users_orders AS (
SELECT 
	*,
	DATE_TRUNC('Month', o.created_at) AS month_
FROM crappy_data_db.users u
JOIN crappy_data_db.orders o ON u.id = o.user_id
)
SELECT 
	month_,
	COALESCE(SUM(amount) FILTER (WHERE age < 30), 0) AS under_30,
	COALESCE(SUM(amount) FILTER (WHERE age >= 30 AND age <= 50), 0) AS age_30_to_50,
	COALESCE(SUM(amount) FILTER (WHERE age > 50), 0) AS over_50,
	COALESCE(SUM(amount) FILTER (WHERE age IS NULL), 0) AS unknown,
	SUM(amount) AS total_revenue
FROM users_orders
GROUP BY month_
ORDER BY month_

This pattern is super handy and much more efficient than classic CASE WHEN FILTERING and then grouping - love it. Definitely gotta practice it and use in a lot of differnet contexts and advanced scenarios.

This talk felt easy.

---

## Task 3: Complex Anti-Join — Users Who Ordered but Never Had a Delivery

**Scenario:**
The operations team wants to find users who placed at least one order but none of their orders ever received a delivery record. These are potential fulfilment failures.

**Expected Output Columns:**
- `user_id` (integer)
- `order_count` (bigint) — total orders placed by this user
- `total_order_value` (numeric) — rounded to 2 decimals

**Requirements:**
- Use `users`, `orders`, and `deliveries` tables
- A user qualifies if they have orders but NONE of those orders appear in `deliveries`
- Solve using `NOT EXISTS` (preferred) — no need for all three approaches this time
- Order by `total_order_value DESC`

**Difficulty Rating:** 4/5\\

SELECT * 
FROM crappy_data_db.orders o
WHERE NOT EXISTS
(SELECT * FROM crappy_data_db.deliveries d
 WHERE d.order_id = o.id)

 That's the first and most logical step.
 Such orders DO NOT EXIST, SO THERE'S ABSOLUTELY NO reason to proceed further.

 IF THERE WERE such orders/users, it would be as simple as finding their total order count and order value.

---

## Submission Instructions

1. Task 1 — Trade burst clustering (4/5)
2. Task 2 — Age group revenue PIVOT (4/5)
3. Task 3 — Users with orders but no deliveries (4/5)
