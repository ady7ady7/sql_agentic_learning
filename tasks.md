# Daily SQL Practice Tasks

**Generated:** 2026-04-09
**Week 17, Day 3 Focus:** Easy-medium session — LAG for change detection + self-join affinity + window running total

---

## Task 1: LAG — Detect Users Whose Order Value Dropped

**Scenario:**
The retention team wants to find users whose most recent order was lower in value than their previous order — a potential sign of disengagement.

**Expected Output Columns:**
- `user_id` (integer)
- `prev_amount` (numeric) — amount of the second-to-last order, rounded to 2 decimals
- `last_amount` (numeric) — amount of the most recent order, rounded to 2 decimals
- `drop` (numeric) — prev_amount minus last_amount, rounded to 2 decimals

**Requirements:**
- Use `orders`, exclude NULL amounts
- Use LAG to get the previous order amount per user (ordered by created_at ASC)
- Only return users where the last order amount is lower than the previous one
- Only include users with at least 2 orders
- Order by `drop DESC`

**Difficulty Rating:** 3/5

WITH users_orders AS (
SELECT 
	*,
	LAG(o.amount) OVER (PARTITION BY user_id ORDER BY created_at DESC) AS prev_amount,
	rank() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
FROM crappy_data_db.orders o
ORDER BY user_id, created_at DESC
)
SELECT 
	user_id,
	prev_amount,
	amount AS last_amount,
	prev_amount - amount AS drop
FROM users_orders
WHERE rn = 2
ORDER BY DROP desc

I used slightly different, but clearer logic.


---

## Task 2: Self-Join — Products Frequently Bought Together

**Scenario:**
The recommendations team wants to find pairs of products that appear together in the same order at least 3 times — to power a "frequently bought together" feature.

**Expected Output Columns:**
- `product_a` (integer) — lower product_id of the pair
- `product_b` (integer) — higher product_id of the pair
- `times_bought_together` (integer)

**Requirements:**
- Use `orders_products`
- Self-join on `order_id`, with `op1.product_id < op2.product_id` to avoid duplicates
- Only include pairs appearing together in at least 3 distinct orders
- Order by `times_bought_together DESC`

**Difficulty Rating:** 3/5

SELECT
	op1.product_id AS product_a,
	op2.product_id AS product_b,
	COUNT(*) AS times_bought_together
FROM crappy_data_db.orders_products op1
JOIN crappy_data_db.orders_products op2 ON op1.order_id = op2.order_id
WHERE op1.product_id > op2.product_id
GROUP BY op1.product_id, op2.product_id
HAVING COUNT(*) >= 3
ORDER BY times_bought_together DESC


---

## Task 3: Running Total — Cumulative Revenue per User Over Time

**Scenario:**
The finance team wants to see how each user's cumulative order spend grows over time — one row per order, with the running total up to and including that order.

**Expected Output Columns:**
- `user_id` (integer)
- `order_id` (integer)
- `created_at` (timestamp)
- `amount` (numeric)
- `cumulative_spent` (numeric) — running total of amount for this user up to this order, rounded to 2 decimals

**Requirements:**
- Use `orders`, exclude NULL amounts
- SUM OVER partitioned by user_id, ordered by created_at ASC
- Only include users with at least 3 orders
- Order by `user_id ASC`, `created_at ASC`

**Difficulty Rating:** 3/5

SELECT 
	o.user_id,
	o.id AS order_id,
	o.created_at,
	o.amount,
	SUM(o.amount) OVER (PARTITION BY o.user_id ORDER BY o.created_at) AS cumulative_spent
FROM crappy_data_db.orders o


Very easy.


---

## Submission Instructions

1. Task 1 — LAG to detect order value drop per user (3/5)
2. Task 2 — Self-join: products bought together at least 3 times (3/5)
3. Task 3 — Running cumulative spend per user with SUM OVER (3/5)
