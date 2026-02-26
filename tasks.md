# Daily SQL Practice Tasks

**Generated:** 2026-02-26
**Week 11, Day 4 Focus:** HackerRank Hard — Session Outliers + Multi-Table Aggregation + Hierarchy

---

## Task 1: 3-Level Hierarchy — Product Categories, Products, and Order Count

**Scenario:**
Build a 3-level hierarchy over product sales:
- Level 1: `'All Categories'`
- Level 2: Distinct category names (dynamic, from `product_categories`)
- Level 3: For each category, the 3 products with the highest total quantity sold (show product name)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — category name at Level 2, product name at Level 3
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Pre-aggregate total quantity sold per product before the recursive CTE
- Use `orders_products` and `products` and `product_categories`
- Termination condition required

**Difficulty Rating:** 4/5


WITH RECURSIVE product_sold_amt AS (
SELECT 
	op.product_id,
	p."name" AS product_name,
	p.category_id AS category_id,
	pc.name AS category_name,
	SUM(op.quantity) AS total_quantity_sold
FROM orders_products op
JOIN products p ON op.product_id = p.id
JOIN product_categories pc ON pc.id = p.category_id
GROUP BY op.product_id, p."name", p.category_id, pc.name
),
product_sales_ranking AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY category_id ORDER BY total_quantity_sold DESC) AS product_sales_rank
FROM product_sold_amt
),
top_three_products AS (
SELECT * FROM product_sales_ranking
WHERE product_sales_rank <= 3
),
distinct_categories AS (
SELECT DISTINCT id, name FROM product_categories PC
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Categories' AS name,
	NULL::TEXT AS parent_name,
	'All Categories' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(dc.name::TEXT, ttp.product_name),
	h.name,
	h.PATH || ' > ' || COALESCE(dc.name::TEXT, ttp.product_name)
FROM HIERARCHY h
LEFT JOIN distinct_categories dc ON h.LEVEL = 1
LEFT JOIN top_three_products ttp ON h.LEVEL = 2 AND h.name = ttp.category_name
WHERE h.LEVEL < 3
)
SELECT * FROM hierarchy


---

## Task 2: User Session Outlier Days

**Scenario:**
The engagement team wants to identify days where a user's session count was unusually high — specifically, days where their session count was more than **2 standard deviations above their own mean**.

For each such outlier day, show:

**Expected Output Columns:**
- `user_id` (integer)
- `date` (date)
- `count_sessions` (integer)
- `user_avg_sessions` (numeric) — that user's average daily sessions, rounded to 2 decimals
- `user_stddev_sessions` (numeric) — that user's standard deviation of daily sessions, rounded to 2 decimals
- `z_score` (numeric) — (count_sessions - user_avg) / user_stddev, rounded to 2 decimals

**Requirements:**
- Use `user_sessions_daily`
- Use `AVG()` and `STDDEV()` as window functions (no GROUP BY needed)
- Exclude rows where stddev = 0 (user has identical session count every day — no outliers possible)
- Order by `z_score DESC`

**Difficulty Rating:** 4/5

WITH users_sessions_metrics AS (
SELECT
	user_id,
	ROUND(STDDEV(count_sessions), 2) AS user_stddev_sessions,
	ROUND(AVG(count_sessions), 2) AS avg_user_daily_sessions
FROM user_sessions_daily usd 
GROUP BY user_id
)
SELECT 
	usd.user_id,
	usd.date,
	usd.count_sessions,
	usm.avg_user_daily_sessions AS user_avg_sessions,
	usm.user_stddev_sessions,
	ROUND((usd.count_sessions - usm.avg_user_daily_sessions) / usm.user_stddev_sessions, 2) AS z_score
FROM users_sessions_metrics usm
JOIN user_sessions_daily usd ON usd.user_id = usm.user_id
WHERE usm.user_stddev_sessions != 0
ORDER BY z_score DESC

---

## Task 3: Monthly Revenue vs Previous Year Same Month

**Scenario:**
The finance team wants a year-over-year comparison of monthly order revenue. For each month, show the revenue and compare it to the same month in the previous year.

**Expected Output Columns:**
- `month` (date) — truncated to month
- `monthly_revenue` (numeric) — total order amount that month, rounded to 2 decimals
- `same_month_prev_year_revenue` (numeric) — revenue for the same month 1 year ago, rounded to 2 decimals, NULL if no data
- `yoy_change` (numeric) — monthly_revenue minus same_month_prev_year_revenue, rounded to 2 decimals, NULL if no previous year data
- `yoy_pct_change` (numeric) — percentage change vs previous year, rounded to 1 decimal, NULL if no previous year data

**Requirements:**
- Use `orders` table
- Use `LAG` with `OFFSET 12` to get the same month from the previous year
- Exclude NULL order amounts
- Order by `month ASC`

**Difficulty Rating:** 4/5

WITH orders_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
	--DATE_TRUNC('Month', (DATE_TRUNC('Month', created_at) - INTERVAL '365' DAY)) AS prev_year
FROM orders
),
monthly_revenueS AS (
SELECT 
	month_,
	--prev_year,
	SUM(amount) AS total_revenue
FROM orders_months
GROUP BY month_
ORDER BY month_
),
revenues_prev_year AS (
SELECT 
	*,
	LAG(total_revenue, 12) OVER (ORDER BY month_) AS same_month_prev_year_revenue
FROM monthly_revenues
)
SELECT 
	*,
	total_revenue - same_month_prev_year_revenue AS yoy_change,
	ROUND(total_revenue::NUMERIC / same_month_prev_year_revenue::NUMERIC * 100, 1) || '%' AS yoy_pct_change
FROM revenues_prev_year
WHERE same_month_prev_year_revenue IS NOT NULL

Learning that we can specify the offset number in LAG is very useful - I didn't know that to be honest.


---

## Submission Instructions

1. Task 1 — Category/product hierarchy by quantity sold (4/5)
2. Task 2 — User session outlier days with z-score (4/5)
3. Task 3 — Monthly revenue vs previous year (4/5)
