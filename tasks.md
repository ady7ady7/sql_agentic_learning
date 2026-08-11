# SQL Tasks — 2026-08-11 (Week 34, Day 2)

**Dataset:** orders / transactions / users / products / product_categories / orders_products  
**Focus:** TOP N per group · Anti-join with date filter

---

## Task 1 — Top 3 Products per Category by Revenue
**Difficulty: 3/5**

**Business question:**  
For each product category, find the top 3 products by total revenue (quantity × price). If there's a tie at position 3, include all tied products.

**Expected output columns:**  
`category_name, product_name, total_revenue, rank`

Only include rows where `rank <= 3`. Order by `category_name`, `rank`.

**Difficulty: 3/5**


WITH products_revs AS (
SELECT 
	PC.NAME AS product_category,
	p.id AS product_id,
	p.name AS product_name,
	SUM(p.price * op.quantity) AS total_revenue
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.products p ON op.product_id = p.id
JOIN crappy_data_db.product_categories pc ON p.category_id = pc.id
GROUP BY PC.NAME, p.id, p.name
),
categories_revenue_ranks AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY product_category ORDER BY total_revenue DESC) AS product_rank
FROM products_revs
)
SELECT * FROM categories_revenue_ranks
WHERE product_rank <= 3



---

## Task 2 — Inactive Users (No Transactions in Last 90 Days)
**Difficulty: 3/5**

**Business question:**  
Find all users who have never had a transaction in the last 90 days relative to the most recent transaction in the dataset. Include users who have never transacted at all.

Show their `user_id` and their most recent transaction date (`last_tx_date`) — NULL if they never transacted.

**Expected output columns:**  
`user_id, last_tx_date`

Order by `last_tx_date` DESC NULLS LAST.

**Difficulty: 3/5**


WITH last_tx_date AS (
SELECT 
	max(created_at) AS last_tx
FROM crappy_data_db.transactions t
)
SELECT 
	u.id AS user_id,
	MAX(t2.created_at) AS last_tx_date
FROM crappy_data_db.users u
JOIN crappy_data_db.transactions t2 ON u.id = t2.user_id
JOIN last_tx_date l ON l.last_tx >= t2.created_at
WHERE NOT EXISTS (
SELECT
	t.user_id
FROM crappy_data_db.transactions t
WHERE t.created_at >= l.last_tx - INTERVAL '90 Days'
AND t.user_id = u.id
)
GROUP BY u.id
ORDER BY last_tx_date DESC NULLS LAST


---

## Submission Instructions

Paste your queries below each task.