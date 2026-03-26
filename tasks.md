# Daily SQL Practice Tasks

**Generated:** 2026-03-26
**Week 15, Day 4 Focus:** Time-Proximity Variant + PIVOT Multi-Dimension + Anti-Join Complex

---

## Task 1: Time-Proximity — Order Bursts per User with Variable Threshold

**Scenario:**
Group each user's orders into bursts where consecutive orders are placed within **3 days** of each other. This tests the pattern at day granularity rather than minutes.

Use the `orders` table from the real database.

**Expected Output Columns:**
- `user_id` (integer)
- `burst_id` (bigint) — sequential per user (1, 2, 3...)
- `burst_start` (date)
- `burst_end` (date)
- `order_count` (bigint)
- `total_revenue` (numeric) — rounded to 2 decimals
- `duration_days` (integer) — days between burst_start and burst_end

**Requirements:**
- Use `DATE(created_at)` to work at day granularity
- LAG → is_new_burst flag (gap > 3 days) → SUM() OVER → GROUP BY
- Only show bursts with at least 2 orders
- Order by `user_id ASC`, `burst_start ASC`

**Difficulty Rating:** 4/5

WITH orders_users AS (
SELECT 
	*,
	LAG(o.created_at) OVER (PARTITION BY user_id ORDER BY o.created_at) AS prev_order_time
FROM crappy_data_db.orders o
),
orders_users_is_streak AS (
SELECT 
	*,
	CASE WHEN prev_order_time IS NULL OR created_at - prev_order_time > INTERVAL '3 Days' THEN 1 ELSE 0 END AS is_new_streak
FROM orders_users
),
orders_users_streaks_id AS (
SELECT 
	*,
	SUM(is_new_streak) OVER (PARTITION BY user_id ORDER BY created_at) AS streak_id
FROM orders_users_is_streak
),
users_order_streaks AS (
SELECT 
	user_id,
	streak_id,
	MIN(created_at) AS streak_start,
	MAX(created_at) AS streak_end,
	COUNT(*) AS order_count,
	SUM(amount) AS total_revenue,
	ROUND(EXTRACT(EPOCH FROM MAX(created_at) - MIN(created_at)) / 86400, 2) AS duration_days
FROM orders_users_streaks_id
GROUP BY user_id, streak_id
)
SELECT * FROM users_order_streaks
WHERE order_count >= 2
ORDER BY user_id, streak_start


Lovely pattern, feels like I've mastered it already more or less.
Also I changed the column names a bit, as streak sounds a bit more natural imo than burst in this context.

---

## Task 2: PIVOT — User Age Group × Transaction Type Revenue Matrix

**Scenario:**
The finance team wants a full matrix showing total transaction revenue broken down by both age group (rows) and transaction type (columns).

**Expected Output Columns:**
- `age_group` (text) — `'under_30'`, `'30_to_50'`, `'over_50'`
- `deposit` (numeric) — rounded to 2 decimals
- `withdrawal` (numeric) — rounded to 2 decimals
- `payment` (numeric) — rounded to 2 decimals
- `transfer` (numeric) — rounded to 2 decimals
- `purchase` (numeric) — rounded to 2 decimals
- `total` (numeric) — rounded to 2 decimals

**Requirements:**
- Use `transactions` and `users` tables
- Join on `user_id`, exclude NULL ages and NULL amounts
- Age groups: `age < 30` → `'under_30'`, `30-50` → `'30_to_50'`, `> 50` → `'over_50'`
- Use `SUM(amount) FILTER (WHERE type = '...')` per column
- Order by `age_group ASC`

**Difficulty Rating:** 4/5

WITH users_age_groups AS (
SELECT 
	id AS users_user_id,
	CASE WHEN age < 30 THEN 'under_30'
	WHEN age >= 30 AND age <= 50 THEN '30_to_50' ELSE 'over_50' END AS age_group
FROM crappy_data_db.users u
WHERE age IS NOT NULL
)
SELECT 
	age_group,
	sum(t.amount) FILTER (WHERE TYPE = 'deposit') AS deposit_sum,
	sum(t.amount) FILTER (WHERE TYPE = 'withdrawal') AS withdrawal_sum,
	sum(t.amount) FILTER (WHERE TYPE = 'payment') AS payment_sum,
	sum(t.amount) FILTER (WHERE TYPE = 'transfer') AS transfer_sum,
	sum(t.amount) FILTER (WHERE TYPE = 'purchase') AS purchase_sum,
	sum(t.amount) AS total
FROM crappy_data_db.transactions t
JOIN users_age_groups uag ON t.user_id = uag.users_user_id
WHERE t.amount IS NOT NULL
GROUP BY age_group


Works perfectly imo, no needed to round anything, as the values are already rounded



---

## Task 3: Anti-Join — Users Who Ordered But Never Sent a Support Ticket

**Scenario:**
The CX team wants to identify "silent" customers — users who have placed at least one order but have never opened a support ticket. These users may have had issues they never reported.

**Expected Output Columns:**
- `user_id` (integer)
- `total_orders` (bigint)
- `total_order_revenue` (numeric) — rounded to 2 decimals
- `first_order_date` (date)
- `last_order_date` (date)

**Requirements:**
- Use `orders` and `chat_tickets` tables
- Use `NOT EXISTS` to exclude users who have any ticket
- Only include users with at least 1 order
- Order by `total_orders DESC`, `total_order_revenue DESC`

**Difficulty Rating:** 4/5

WITH ticketless_users_with_orders AS (
SELECT
	DISTINCT o.user_id
FROM crappy_data_db.orders o
WHERE NOT EXISTS
(
SELECT 
	user_id 
FROM crappy_data_db.chat_tickets ct
WHERE ct.user_id = o.user_id
)
)
SELECT 
	tu.user_id,
	COUNT(o.id) AS total_orders,
	SUM(o.amount) AS total_order_revenue,
	MIN(o.created_at) AS first_order_date,
	MAX(o.created_at) AS last_order_date
FROM ticketless_users_with_orders tu
JOIN crappy_data_db.orders o ON tu.user_id = o.user_id
GROUP BY tu.user_id
ORDER BY total_orders DESC, total_order_revenue DESC


---

## Submission Instructions

1. Task 1 — Order bursts at day granularity (4/5)
2. Task 2 — Age group × transaction type revenue matrix (4/5)
3. Task 3 — Anti-join: ordered but never ticketed users (4/5)
