# SQL Tasks — Week 32 Day 4

**Generated:** 2026-07-30
**Dataset:** crappy_data
**Focus:** Rolling window z ramką czasową, date spine z COALESCE, Top-N per grupa z RANK

---

## Task 1: Rolling 7-Day Transaction Sum
**Difficulty: 3/5**

**Business question:**
For each transaction, compute the rolling sum of `amount` for that user over the preceding 7 days (including the current transaction's day). Every transaction row should appear in the output with its rolling total at that point in time.

The window should be time-based — cover the current day and the 6 days before it — not a fixed number of rows.

**Expected output columns:**
`user_id, created_at, type, amount, rolling_7d_sum`

**Difficulty: 3/5**

SELECT 
	user_id,
	created_at,
	TYPE,
	amount,
	SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at ROWS BETWEEN 1 PRECEDING AND CURRENT ROW) AS rolling_7d_sum
FROM crappy_data_db.transactions t

EASY


---

## Task 2: Monthly Active Users — Including Empty Months
**Difficulty: 4/5**

**Business question:**
For each calendar month in the `dates` table (from the earliest to the latest order date), show how many distinct users placed at least one order that month. Months with zero orders should appear with a count of 0 — do not drop them.

**Expected output columns:**
`month, active_users`

**Difficulty: 4/5**


WITH months_ AS (
SELECT 
	DISTINCT(date_trunc('Month', date)) AS month_
FROM crappy_data_db.dates d
),
orders_months AS (
SELECT 
	*,
	date_trunc('Month', o.created_at) AS month_
FROM crappy_data_db.orders o
)
SELECT 
	m.month_,
	COALESCE(COUNT(DISTINCT(om.user_id)), 0) AS active_users 
FROM months_ m
LEFT JOIN orders_months om ON om.month_ = m.month_
GROUP BY m.month_
ORDER BY m.month_


---

## Task 3: Top 3 Products by Avg Quantity per Category
**Difficulty: 5/5**

**Business question:**
For each product category, find the top 3 products ranked by their average quantity per order line (`orders_products.quantity`). If there are ties at rank 3, include all tied products. Categories with fewer than 3 products are fine — just show what's there.

**Expected output columns:**
`category_name, product_name, avg_quantity, rank`

**Difficulty: 5/5**




WITH categories_products_avg_qties AS (
SELECT 
	p.category_id,
	pc."name" AS category_name,
	p.id AS product_id,
	ROUND(AVG(op.quantity), 2) AS avg_quantity_per_order
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.products p ON op.product_id = p.id
JOIN crappy_data_db.product_categories pc ON p.category_id = pc.id
GROUP BY p.category_id, pc."name", p.id
),
categories_ranks AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY category_name ORDER BY avg_quantity_per_order DESC) AS category_rank
FROM categories_products_avg_qties
)
SELECT * FROM categories_ranks 
WHERE category_rank <= 3


Easy for me.

---

## Submission Instructions

Paste your queries below each task.
