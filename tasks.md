# Daily SQL Practice Tasks

**Generated:** 2026-04-14
**Week 18, Day 2 Focus:** Light session — window functions review + basic aggregation

---

## Task 1: Top Product per Category by Revenue

**Scenario:**
The product team wants to know the single best-selling product (by revenue) within each category.

**Expected Output Columns:**
- `category_name` (varchar)
- `product_name` (varchar)
- `revenue` (numeric) — sum of price × quantity, rounded to 2 decimals

**Requirements:**
- Use `product_categories`, `products`, `orders_products`
- Only include rows where price IS NOT NULL
- Use RANK() or ROW_NUMBER() partitioned by category, then filter to rank = 1
- Order by `revenue DESC`

**Difficulty Rating:** 2/5

WITH products_sales AS (
SELECT 
	pc."name" AS category_name,
	p.name AS product_name,
	SUM(p.price * op.quantity) AS revenue
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.products p ON p.id = op.product_id
JOIN crappy_data_db.product_categories pc ON p.category_id = pc.id
WHERE price IS NOT NULL
GROUP BY pc.name, p.name
),
categories_rank AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY category_name ORDER BY revenue DESC) AS sales_rank
FROM products_sales
)
SELECT * FROM categories_rank WHERE sales_rank = 1
ORDER BY revenue DESC



---

## Task 2: Transaction Count and Amount by Type and Month

**Scenario:**
The finance team wants a monthly breakdown of transaction volume and total amount per transaction type.

**Expected Output Columns:**
- `year` (integer)
- `month` (integer)
- `type` (text)
- `transaction_count` (integer)
- `total_amount` (numeric) — rounded to 2 decimals

**Requirements:**
- Use `transactions`, exclude NULL types and NULL amounts
- Extract year and month from created_at
- Order by `year ASC`, `month ASC`, `type ASC`

**Difficulty Rating:** 2/5


WITH transactions_y_m AS (
SELECT 
	*,
	date_trunc('Year', created_at) AS YEAR,
	date_trunc('Month', created_at) AS month
FROM crappy_data_db.transactions t
)
SELECT 
	YEAR,
	MONTH,
	TYPE,
	COUNT(*) AS transaction_count,
	SUM(amount) AS total_amount
FROM transactions_y_m
GROUP BY YEAR, MONTH, TYPE
ORDER BY YEAR, month, type


---

## Task 3: Users with Above-Average Order Spend

**Scenario:**
Find users whose total order spend is above the overall average total spend across all users.

**Expected Output Columns:**
- `user_id` (integer)
- `total_spent` (numeric) — rounded to 2 decimals
- `overall_avg` (numeric) — the overall average, same value on every row, rounded to 2 decimals

**Requirements:**
- Use `orders`, exclude NULL amounts
- Compute total_spent per user, then compare against the average of those totals
- Include `overall_avg` as a window or scalar subquery so it's visible in output
- Order by `total_spent DESC`

**Difficulty Rating:** 3/5

WITH users_spents AS (
SELECT 
	user_id,
	ROUND(SUM(o.amount)::numeric, 2) AS total_spent
FROM crappy_data_db.orders o
GROUP BY user_id
)
SELECT 
	user_id,
	total_spent,
	ROUND((SELECT AVG(total_spent) FROM users_spents), 2) AS overall_avg
FROM users_spents
ORDER BY total_spent DESC



---

## Submission Instructions

1. Task 1 — Top product per category by revenue using RANK (2/5)
2. Task 2 — Monthly transaction breakdown by type (2/5)
3. Task 3 — Users with above-average total spend (3/5)
