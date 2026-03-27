# Daily SQL Practice Tasks

**Generated:** 2026-03-27
**Week 15, Day 5 Focus:** Review Day — Gaps & Islands + Window Functions + Recursive CTE (Type A)

---

## Task 1: Gaps & Islands — Monthly Active User Streaks

**Scenario:**
Find each user's consecutive months of activity (at least 1 order placed). Report streaks of 2+ consecutive months.

**Expected Output Columns:**
- `user_id` (integer)
- `streak_start` (text) — first month in streak, formatted as `'YYYY-MM'`
- `streak_end` (text) — last month in streak, formatted as `'YYYY-MM'`
- `streak_length` (bigint) — number of consecutive months

**Requirements:**
- Use `orders` table
- Derive active months using `DATE_TRUNC('month', created_at)`
- Use the classic gaps-and-islands pattern: `ROW_NUMBER()` subtraction to group consecutive months
- Only include streaks of 2+ months
- Order by `streak_length DESC`, `user_id ASC`

**Difficulty Rating:** 3/5


WITH orders_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
FROM crappy_data_db.orders
),
users_months_orders AS (
SELECT 
	user_id,
	month_,
	COUNT(*) AS orders_cnt
FROM orders_months
GROUP BY user_id, month_
ORDER BY user_id
),
orders_months_rn AS (
SELECT 
	*,
	ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY month_) AS rn
FROM users_months_orders
),
users_streak_ids AS (
SELECT 
	*,
	month_ - rn * INTERVAL '1 Month' AS streak_id
FROM orders_months_rn
),
users_streaks AS (
SELECT 
	user_id,
	MIN(month_) AS streak_start,
	MAX(month_) AS streak_end,
	COUNT(*) AS streak_length
	FROM users_streak_ids
GROUP BY user_id, streak_id
ORDER BY streak_length DESC, user_id
)
SELECT * FROM users_streaks WHERE streak_length >= 2


---

## Task 2: Window Functions — Top Spender Per Product Category

**Scenario:**
For each product category, find the top 3 users by total spend. Include their rank and total spend.

**Expected Output Columns:**
- `category_name` (text)
- `user_id` (integer)
- `rank` (bigint)
- `total_spend` (numeric) — rounded to 2 decimals

**Requirements:**
- Use `orders`, `orders_products`, `products`, `product_categories`
- Revenue = `quantity × price`
- Use `RANK()` window function partitioned by category
- Only return ranks 1–3
- Order by `category_name ASC`, `rank ASC`

**Difficulty Rating:** 3/5


WITH users_categories_spendings AS (
SELECT 
	pc.name AS category_name,
	o.user_id,
	SUM(p.price * op.quantity) AS category_spent
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.products p ON OP.product_id = p.id
JOIN crappy_data_db.product_categories pc ON pc.id = p.category_id
JOIN crappy_data_db.orders o ON o.id = op.order_id
GROUP BY pc.name, o.user_id
),
users_spending_rank AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY category_name ORDER BY category_spent DESC) AS rank
FROM users_categories_spendings
)
SELECT * FROM users_spending_rank 
WHERE RANK <= 3
ORDER BY category_name, rank

I thought category_spent IS WAY MORE logical, as total_spent indicates that these are total user's spendings. Anyway, it's good

---


## Task 3: Recursive CTE (Type A) — 3-Level Category Hierarchy Summary

**Scenario:**
The product team wants a summary of the category hierarchy: for each top-level category, show its direct subcategories and the total number of products at leaf level.

**Expected Output Columns:**
- `top_level` (text) — name of root category (parent_id IS NULL)
- `sub_level` (text) — name of direct child category
- `product_count` (bigint) — total products belonging to the sub_level category

**Requirements:**
- Use `product_categories` and `products` tables
- Fixed 3-level structure: root → subcategory → products
- Build with CTEs: root categories, subcategories, then join to products
- Order by `top_level ASC`, `product_count DESC`

**Difficulty Rating:** 3/5

Stupid question, THERE IS NO SUCH STRUCTURE IN MY DATABASE.
AVOIDING IT, DON'T PUNISH ME FOR IT!

---

## Submission Instructions

1. Task 1 — Monthly active user streaks (3/5)
2. Task 2 — Top 3 spenders per category (3/5)
3. Task 3 — 3-level category hierarchy summary (3/5)
