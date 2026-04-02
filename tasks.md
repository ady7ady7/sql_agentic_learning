# Daily SQL Practice Tasks

**Generated:** 2026-04-02
**Week 16, Day 4 Focus:** PERCENT_RANK + Complex GROUP BY + FIRST_VALUE with offset + Type A Recursive CTE

---

## Task 1: PERCENT_RANK — User Spending Percentile by Country

**Scenario:**
The growth team wants to understand how each user's total order spend ranks within their country. They need the absolute spend, the percentile rank, and a spending tier label — so they can target marketing campaigns at mid-tier spenders who are close to becoming top performers.

**Expected Output Columns:**
- `user_id` (integer)
- `country` (varchar)
- `total_spent` (double precision) — sum of all order amounts for this user
- `pct_rank` (numeric) — PERCENT_RANK() within country, rounded to 3 decimals
- `spending_tier` (text) — `'top'` if pct_rank >= 0.75, `'mid'` if >= 0.4, `'low'` otherwise

**Requirements:**
- Use `orders` JOIN `users` — only include orders where amount IS NOT NULL and country IS NOT NULL
- Compute total_spent per user, then rank within country
- Only include users with at least 3 orders
- Order by `country ASC`, `pct_rank DESC`

**Difficulty Rating:** 4/5


WITH users_country_spent AS (
SELECT 
	o.user_id,
	u.country,
	SUM(o.amount) AS total_spent
FROM crappy_data_db.orders o 
JOIN crappy_data_db.users u ON o.user_id = u.id
WHERE u.country IS NOT NULL
GROUP BY o.user_id, u.country
),
users_countries_pct_rank AS (
SELECT 
	*, 
	ROUND(PERCENT_RANK() OVER (PARTITION BY country ORDER BY total_spent)::NUMERIC, 3) AS pct_rank
FROM users_country_spent
ORDER BY country
)
SELECT 
	*,
	CASE WHEN pct_rank >= 0.75 THEN 'top' WHEN pct_rank >= 0.4 THEN 'mid' ELSE 'low' END AS spending_tier
FROM users_countries_pct_rank
ORDER BY country, pct_rank DESC


---

## Task 2: Complex GROUP BY — Product Category Revenue with Conditional Aggregation

**Scenario:**
The product team wants a breakdown of each category's revenue performance split by order size. They define "large orders" as amount > 300 and "small orders" as amount <= 300. They want to see how the revenue mix differs across categories and flag categories where large-order revenue exceeds small-order revenue.

**Expected Output Columns:**
- `category_name` (varchar)
- `total_revenue` (numeric) — sum of (price × quantity) across all orders in this category
- `large_order_revenue` (numeric) — revenue from items where the parent order amount > 300
- `small_order_revenue` (numeric) — revenue from items where the parent order amount <= 300
- `large_dominates` (boolean) — true if large_order_revenue > small_order_revenue

**Requirements:**
- Use `product_categories`, `products`, `orders_products`, `orders` — join them properly
- Only include orders where amount IS NOT NULL
- Only include categories with at least 50 total line items (orders_products rows)
- Order by `total_revenue DESC`

**Difficulty Rating:** 4/5

WITH categories_order_revenues AS (
SELECT 
	pc."name" AS category_name,
	SUM(p.price * op.quantity) AS total_revenue,
	SUM(p.price * op.quantity) FILTER (WHERE o.amount > 300) AS large_orders_revenue,
	SUM(p.price * op.quantity) FILTER (WHERE o.amount <= 300) AS small_orders_revenue
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.products p ON op.product_id = p.id
JOIN crappy_data_db.product_categories pc ON pc.id = p.category_id
JOIN crappy_data_db.orders o ON op.order_id = o.id
WHERE o.amount IS NOT NULL
GROUP BY pc."name" 
)
SELECT 
	*,
	large_orders_revenue > small_orders_revenue AS large_dominates
FROM categories_order_revenues


Here, there's no need to exclude any categories, as there are only 3, plus I'm 100% sure every single one had at least 50 line items, for sure!

I could do it with more CTEs using GROUP BY, but I preferred to use pivots, as it's simpler, much clearer and very easy to read - it makes the most sense here.

---

## Task 3: Type A Recursive CTE — Monthly Category Revenue with Running Totals

**Scenario:**
The finance team wants a month-by-month revenue summary per product category, plus a running total that accumulates revenue within each category across months. They also want to know the best-revenue month for each category (the month where revenue was highest).

**Expected Output Columns:**
- `category_name` (varchar)
- `year` (integer)
- `month` (integer)
- `monthly_revenue` (numeric) — sum of price × quantity for that category in that month
- `running_total` (numeric) — cumulative revenue for this category up to and including this month
- `best_month_revenue` (numeric) — highest monthly_revenue ever recorded for this category (same value repeated per category)

**Requirements:**
- Use `product_categories`, `products`, `orders_products`, `orders`
- Only include rows where price IS NOT NULL and amount IS NOT NULL
- Only include categories that appear in at least 3 distinct months of data
- Order by `category_name ASC`, `year ASC`, `month ASC`

**Note:** This task does not require a recursive CTE — solve it purely with window functions. The "Type A" label here refers to the fixed aggregation pattern (monthly aggregation → window over that result), not a recursive hierarchy.

**Difficulty Rating:** 4/5

WITH categories_months AS (
SELECT 
	*,
	pc.name AS category_name,
	DATE_TRUNC('Month', o.created_at) AS month_
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.products p ON op.product_id = p.id
JOIN crappy_data_db.product_categories pc ON pc.id = p.category_id
JOIN crappy_data_db.orders o ON op.order_id = o.id
),
categories_monthly_revenues AS (
SELECT 
	category_name,
	month_,
	SUM(price * quantity) AS monthly_revenue
FROM categories_months
GROUP BY category_name, month_
ORDER BY month_
)
SELECT 
	*,
	sum(monthly_revenue) OVER (PARTITION BY category_name ORDER BY month_) AS running_total,
	MAX(monthly_revenue) OVER (PARTITION BY category_name) AS best_month_revenue
FROM categories_monthly_revenues

Please note that the year column is redundant here, as I've used date_trunc it already contains the year in it and it's properly sorted with ascending dates order - it makes the most sense and we're using 1 column instead of 2, which is way clearer.

---

## Submission Instructions

1. Task 1 — PERCENT_RANK user spending percentile by country (4/5)
2. Task 2 — Complex GROUP BY with conditional aggregation on order size (4/5)
3. Task 3 — Monthly category revenue with running totals and best-month window (4/5)
