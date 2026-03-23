# Daily SQL Practice Tasks

**Generated:** 2026-03-23
**Week 15, Day 1 Focus:** Time-Proximity Variant + PIVOT Complex + Anti-Join with Subquery Twist

---

## Task 1: Time-Proximity Gaps-and-Islands — Order Bursts per User

**Scenario:**
The operations team wants to identify "ordering bursts" — periods where a user places multiple orders in quick succession. Define a burst as a sequence of orders where each consecutive order arrives within **2 hours** of the previous one (per user).

For each burst show:

**Expected Output Columns:**
- `user_id` (integer)
- `burst_id` (bigint) — sequential per user (1, 2, 3...)
- `burst_start` (timestamp)
- `burst_end` (timestamp)
- `order_count` (bigint)
- `burst_revenue` (numeric) — total order amount in the burst, rounded to 2 decimals

**Requirements:**
- Use `orders` table, exclude NULL amounts
- Gap threshold: 2 hours between consecutive orders per user
- LAG → is_new_burst flag → SUM() OVER burst_key → GROUP BY pattern
- Only include bursts with at least 2 orders
- Order by `user_id ASC`, `burst_start ASC`

**Difficulty Rating:** 4/5

WITH users_orders AS (
SELECT 
	*,
	LAG(o.created_at) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_order_time
FROM crappy_data_db.orders o
),
users_streak_beginnings AS (
SELECT 
	*,
	CASE WHEN prev_order_time IS NULL OR created_at - prev_order_time > INTERVAL '2 HOURS' THEN 1 ELSE 0 END AS is_new_streak
FROM users_orders
),
users_streak_keys AS (
SELECT 
	*,
	SUM(is_new_streak) OVER (PARTITION BY user_id ORDER BY created_at) AS streak_key
FROM users_streak_beginnings
),
users_order_streaks AS (
SELECT 
	user_id,
	streak_key AS burst_id,
	MIN(created_at) AS burst_start,
	MAX(created_at) AS burst_end,
	COUNT(*) AS order_count,
	SUM(amount) AS burst_revenue
FROM users_streak_keys
GROUP BY user_id, streak_key
)
SELECT 
*
FROM users_order_streaks
WHERE order_count > 1
ORDER BY user_id, burst_start


Very useful task - this is definitely a pattern I want to practice.
Here there were only 12 such bursts in total, but still - very useful pattern to practice in more and more context and more and more advanced tasks with different data.


---

## Task 2: PIVOT — User Age Group × Order Frequency Matrix

**Scenario:**
The analytics team wants a cross-tab matrix showing how many users fall into each combination of age group and order frequency bucket.

Age groups:
- `under_30`: age < 30
- `30_to_50`: age between 30 and 50
- `over_50`: age > 50

Order frequency buckets (total orders per user):
- `one_time`: exactly 1 order
- `occasional`: 2–4 orders
- `regular`: 5+ orders

**Expected Output Columns:**
- `age_group` (text)
- `one_time` (bigint)
- `occasional` (bigint)
- `regular` (bigint)
- `total_users` (bigint)

**Requirements:**
- Use `users` and `orders` tables
- Exclude users with NULL age
- Users with 0 orders are NOT included (only users who appear in orders)
- Use conditional aggregation for the pivot columns
- Order by `age_group ASC`

**Difficulty Rating:** 4/5

WITH users_orders_cnt AS (
SELECT 
	user_id,
	COUNT(*) AS orders_cnt,
	CASE WHEN COUNT(*) = 1 THEN 'one_time' WHEN COUNT (*) BETWEEN 2 AND 4 THEN 'occasional' ELSE 'regular' END AS frequency_bucket
FROM crappy_data_db.orders o
GROUP BY o.user_id
),
users_orders_age AS (
SELECT 
	*,
	CASE WHEN u.age < 30 THEN 'under_30' WHEN age BETWEEN 30 AND 50 THEN '30_to_50' ELSE 'over_50' END AS age_group
FROM users_orders_cnt uo
JOIN crappy_data_db.users u ON uo.user_id = u.id
)
SELECT 
	age_group,
	COUNT(*) FILTER (WHERE frequency_bucket = 'one_time') AS one_time,
	COUNT(*) FILTER (WHERE frequency_bucket = 'occasional') AS occasional,
	COUNT(*) FILTER (WHERE frequency_bucket = 'regular') AS regular,
	COUNT(*) AS total_users
FROM users_orders_age
GROUP BY age_group
ORDER BY age_group

I really struggled with that today - this matrix did feel unintuitive.

---

## Task 3: Anti-Join — Users Who Ordered but Never Had a Delivered Order

**Scenario:**
The customer success team wants to find users who have placed at least one order, but none of their orders have ever been successfully delivered (no delivery record with `status = 'delivered'`).

Solve this using **NOT EXISTS** only — this is the safest pattern when the subquery involves NULLable joins.

**Expected Output Columns:**
- `user_id` (integer)
- `total_orders` (bigint)
- `first_order_date` (date)

**Requirements:**
- Use `users`, `orders`, `deliveries` tables
- A "delivered order" = an order that has at least one delivery record with `status = 'delivered'`
- Only include users who have at least 1 order
- Order by `total_orders DESC`, `first_order_date ASC`

**Difficulty Rating:** 4/5


WITH delivered_users AS (
SELECT
DISTINCT o2.user_id
FROM crappy_data_db.orders o1
JOIN crappy_data_db.deliveries d
ON d.status = 'delivered' AND d.order_id = o1.id
JOIN crappy_data_db.orders o2 ON o1.user_id = o2.user_id
),
non_delivered_users AS (
SELECT 
	o.user_id
FROM crappy_data_db.orders o
WHERE NOT EXISTS
(SELECT * FROM delivered_users du WHERE du.user_id = o.user_id)
)
SELECT 
	ndu.user_id,
	COUNT(*) AS total_orders,
	MIN(o.created_at) AS first_order_date
FROM non_delivered_users ndu
JOIN crappy_data_db.orders o ON ndu.user_id = o.user_id
GROUP BY ndu.user_id


---

## Submission Instructions

1. Task 1 — Order burst clustering (4/5)
2. Task 2 — Age group × order frequency PIVOT matrix (4/5)
3. Task 3 — Anti-join: users with no delivered orders (4/5)
