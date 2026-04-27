# Daily SQL Practice Tasks

**Generated:** 2026-04-27
**Week 20, Day 1 Focus:** Type A CTE drill + NTILE + EXPLAIN ANALYZE query optimization

---

## Task 1: Type A CTE — Revenue by Category and Product

**Scenario:**
The product team wants a 2-level revenue breakdown: total revenue per category (level 1), and total revenue per product (level 2). Both in one result set.

**The rule for this task:** Two independent aggregation CTEs, then a plain `UNION ALL`. No `WITH RECURSIVE`, no `hierarchy` CTE, no self-joins. If you catch yourself writing `FROM cte JOIN cte`, stop and rethink.

**Expected Output Columns:**
- `level` (integer — 1 or 2)
- `label` (text — category name for level 1, product name for level 2)
- `revenue` (numeric, rounded to 2 decimals)

**Order by:** `level ASC`, `revenue DESC`

**Tables:** `orders_products`, `products`, `product_categories`

**Difficulty Rating:** 3/5

WITH product_revenues AS (
SELECT 
	p.name AS product_name,
	SUM(p.price * op.quantity) AS product_revenue
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.products p ON op.product_id = p.id
JOIN crappy_data_db.product_categories pc ON p.category_id = pc.id
GROUP BY p.name
),
categories_revenues AS (
SELECT 
	pc.name AS category_name,
	SUM(p.price * op.quantity) AS category_revenue
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.products p ON op.product_id = p.id
JOIN crappy_data_db.product_categories pc ON p.category_id = pc.id
GROUP BY pc.name
)
SELECT 
	1 AS LEVEL,
	category_name AS LABEL, 
	category_revenue AS revenue
FROM categories_revenues
UNION ALL
SELECT 
	2,
	product_name,
	product_revenue
FROM product_revenues
ORDER BY LEVEL, revenue DESC


No need to round anything. It felt pretty natural as well.



---

## Task 2: NTILE — User Spend Quartiles

**Scenario:**
The marketing team wants to segment users into 4 equal spend buckets — from lowest to highest total order spend — so they can target each segment differently.

For each user who has placed at least one order, show:
- `user_id`
- `total_spend` (numeric, rounded to 2 decimals)
- `quartile` — 1 (lowest) to 4 (highest), using `NTILE(4)`
- `segment` — label based on quartile:
  - `'platinum'` for quartile 4
  - `'gold'` for quartile 3
  - `'silver'` for quartile 2
  - `'bronze'` for quartile 1

**Tables:** `orders`

**Requirements:**
- Exclude NULL amounts
- Use a CTE to aggregate total spend per user, then apply `NTILE(4)` in the final SELECT
- Order by `total_spend DESC`

**Difficulty Rating:** 4/5

WITH users_spend AS (
SELECT 
	user_id,
	SUM(o.amount) AS total_spend
FROM crappy_data_db.orders o
GROUP BY user_id
),
users_quartiles AS (
SELECT
	*,
	NTILE(4) OVER (ORDER BY total_spend) AS quartile 
FROM users_spend
)
SELECT 
	user_id,
	total_spend,
	quartile,
	CASE WHEN quartile = 4 THEN 'platinum' WHEN quartile = 3 THEN 'gold' WHEN quartile = 2 THEN 'silver' ELSE 'bronze' END AS segment
FROM users_quartiles
ORDER BY total_spend DESC

Few things - it's already rounded to 2 decimals, no nulls here, everything works as expected. That was a bit easy, not gonna lie.

---

## Task 3: Query Optimization — Rewrite and Compare

**Scenario:**
The following query is logically correct but inefficient — it uses a correlated subquery in the WHERE clause that re-executes for every user row.

**Original slow query:**
```sql
SELECT u.id AS user_id, u.country
FROM crappy_data_db.users u
WHERE (
    SELECT AVG(t.amount)
    FROM crappy_data_db.transactions t
    WHERE t.user_id = u.id
) > 500;
```

**Your tasks:**
1. Rewrite this using a CTE + JOIN (or HAVING) to eliminate the correlated subquery
2. Run `EXPLAIN ANALYZE` on both versions and paste the key timing lines
3. In a comment, explain: what makes the original slow, and what makes yours faster?

**Expected Output Columns:** `user_id`, `country`

**Tables:** `users`, `transactions`

**Difficulty Rating:** 5/5

Original speed: 
Planning Time: 0.083 ms
Execution Time: 5.944 ms

The issue seems to be that we're doing a weird aggregation in where clause, which runs as the first data filter. It means we're trying to filter out the users BEFORE even calculating there collective price averages. Instead of doing one aggregated filtering AFTER the prices are calculated, we're executing a weird filtering command, then calculating the AVG to filter it out, and it's very ineffective. That's my call from my knowledge, but you could further explain it, if I'm wrong or missing the point.

However, my query is a lot better, as first it calculates averages for every user (the aggergation level is each user, country is also there but it doesn't affect the aggregation level since every user_id/id is unique). We easily calculate the averages, and only then exclude the users that are not meeting the required value, but it's done on already calculated values, so it's a lot faster, no weird recalculations there. And we have a simple INNER JOIN as well.

SELECT
	user_id,
	u.country,
	round(AVG(amount), 2) AS avg_transaction
FROM crappy_data_db.transactions t
JOIN crappy_data_db.users u ON t.user_id = u.id
GROUP BY user_id, country
HAVING round(AVG(amount), 2) > 500

Easy as hell,

Planning:
  Buffers: shared hit=2
Planning Time: 0.127 ms
Execution Time: 0.383 ms



---

## Submission Instructions

1. Task 1 — Type A 2-level UNION ALL, no recursion (3/5)
2. Task 2 — NTILE quartile segmentation (4/5)
3. Task 3 — Correlated subquery rewrite + EXPLAIN ANALYZE comparison (5/5)
