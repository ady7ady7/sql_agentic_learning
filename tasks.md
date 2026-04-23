# Daily SQL Practice Tasks

**Generated:** 2026-04-23
**Week 19, Day 4 Focus:** LAG/LEAD + self-join affinity + cohort basics

---

## Task 1: LAG — Days Between Consecutive Orders per User

**Scenario:**
The retention team wants to understand how long users typically wait between orders.

For each order, show:
- `order_id`
- `user_id`
- `created_at`
- `prev_order_date` — date of the previous order for that user (NULL if first order)
- `days_since_prev` — number of days since the previous order (NULL if first order)

**Tables:** `orders`

**Requirements:**
- Use `LAG` to get the previous order date per user
- Exclude NULL amounts
- Order by `user_id ASC`, `created_at ASC`

**Difficulty Rating:** 3/5

WITH orders_prev AS (
SELECT 
	*,
	LAG(o.created_at) OVER (PARTITION BY user_id ORDER BY created_at DESC) AS prev_order_date,
	ABS(EXTRACT('Day' FROM created_at - LAG(o.created_at) OVER (PARTITION BY user_id ORDER BY created_at DESC))) AS days_since_prev
FROM crappy_data_db.orders o
)
SELECT * FROM orders_prev
WHERE amount IS NOT null



---

## Task 2: Self-Join — Users Who Ordered in Both January and February

**Scenario:**
The marketing team wants to find users who placed at least one order in **both** January 2025 and February 2025 — to identify early-year repeat buyers.

**Expected Output Columns:**
- `user_id`

**Tables:** `orders`

**Requirements:**
- Use a self-join (join `orders` to itself)
- No subqueries or CTEs required — a clean self-join with the right conditions is enough
- Order by `user_id ASC`

**Difficulty Rating:** 3/5

SELECT 
	DISTINCT(o1.user_id)
FROM crappy_data_db.orders o1
JOIN crappy_data_db.orders o2 ON o1.user_id = o2.user_id 
WHERE DATE_TRUNC('Month', o1.created_at) = '2025-01-01 00:00:00.000'
AND DATE_TRUNC('Month', o2.created_at) = '2025-02-01 00:00:00.000'

No ordering needed, it's already ordered for whatever reason


---

## Task 3: Cohort Retention — Did Users Order Again After First Month?

**Scenario:**
The growth team wants a simple cohort retention check: for each user, find their first order month, then check whether they placed any order in the **following month** (exactly one month later).

**Expected Output Columns:**
- `user_id`
- `first_order_month` (date — DATE_TRUNC to month)
- `returned_next_month` (boolean — true if they placed an order in the month immediately after their first)

**Tables:** `orders`

**Requirements:**
- Use a CTE to find each user's first order month
- Use a LEFT JOIN back to orders to check for activity in `first_order_month + INTERVAL '1 month'`
- Order by `user_id ASC`

**Difficulty Rating:** 4/5

WITH users_orders AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
FROM crappy_data_db.orders o
),
users_first_orders AS (
SELECT 
	*,
	FIRST_VALUE(month_) OVER (PARTITION BY user_id ORDER BY created_at)
FROM users_orders
),
users_months AS (
SELECT 
	DISTINCT user_id, MONTH_
FROM users_first_orders
ORDER BY user_id, month_
),
users_next_month AS (
SELECT 
	*,
	row_number() OVER (PARTITION BY user_id ORDER BY month_) AS rn_,
	LEAD(month_) OVER (PARTITION BY user_id) AS next_month
FROM users_months
),
users_rn AS (
SELECT * FROM users_next_month
WHERE rn_ = 1
)
SELECT 
	*,
	next_month - month_ AS diff,
	CASE WHEN next_month - month_ <= INTERVAL '31 days' THEN TRUE ELSE FALSE END AS returned_next_month
FROM users_rn

It works.

---

## Submission Instructions

1. Task 1 — LAG days between consecutive orders (3/5)
2. Task 2 — Self-join users active in both Jan and Feb 2025 (3/5)
3. Task 3 — Cohort: did user return the following month (4/5)
