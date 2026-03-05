# Daily SQL Practice Tasks

**Generated:** 2026-03-05
**Week 12, Day 4 Focus:** HackerRank Hard — Final Exam Prep

---

## Task 1: 3-Level Hierarchy — User Segments by Country and Registration Year

**Scenario:**
Build a 3-level hierarchy over user registration data:
- Level 1: `'All Users'`
- Level 2: Distinct countries (exclude NULLs, dynamic from `users`)
- Level 3: For each country, the 3 registration years with the most users (show as `'YYYY (N users)'`)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — country at Level 2, formatted string at Level 3
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Pre-aggregate user counts per country+year before the recursive CTE
- Format Level 3 name as: `EXTRACT(YEAR FROM created_at)::text || ' (' || count::text || ' users)'`
- Exclude NULL countries
- Termination condition required

**Difficulty Rating:** 4/5

#Swapped to top three months instead of years, as there were only 2 years of data, so it wouldn't make sense!

WITH RECURSIVE users_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
FROM users
),
countries_months_registrations AS (
SELECT 
	month_,
	country,
	COUNT(id) AS registered_users_count
FROM users_months
WHERE country IS NOT NULL
GROUP BY month_, country
),
countries_monthly_reg_ranks AS (
SELECT 
	*,
	ROW_NUMBER() OVER (PARTITION BY country ORDER BY registered_users_count DESC, month_) AS country_registration_rank
FROM countries_months_registrations
),
countries_top_three_reg_rank_months AS (
SELECT 
* 
FROM countries_monthly_reg_ranks
WHERE country_registration_rank <= 3
),
distinct_countries AS (
SELECT DISTINCT country FROM users
WHERE country IS NOT NULL
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Users' AS name,
	NULL::TEXT AS parent_name,
	'All Users' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(dc.country::text, ctt.month_::TEXT || ' (' || ctt.registered_users_count::TEXT || ' user(s))'),
	h.name,
	h.PATH || ' > ' || COALESCE(dc.country::text, ctt.month_::TEXT || ' (' || ctt.registered_users_count::TEXT || ' user(s))')
FROM HIERARCHY h
LEFT JOIN distinct_countries dc ON h.LEVEL = 1
LEFT JOIN countries_top_three_reg_rank_months ctt ON h.LEVEL = 2 AND ctt.country = h.name
WHERE h.LEVEL < 3
)
SELECT * FROM HIERARCHY


Definitely a very long query, but I managed to handle it, as well as the formatting, which was quite tricky here. Feels good!




---

## Task 2: Running Total of Orders with Threshold Flags

**Scenario:**
The finance team wants to track each user's cumulative order spending over time, and flag the exact order where they crossed key spending milestones.

For each order, show the user's cumulative total spend up to and including that order, and flag whether it's the order that first pushed them over 500, 1000, or 2000 in total spend.

**Expected Output Columns:**
- `order_id` (integer)
- `user_id` (integer)
- `order_amount` (numeric) — rounded to 2 decimals
- `cumulative_spend` (numeric) — running total per user ordered by created_at, rounded to 2 decimals
- `milestone` (text) — `'500'`, `'1000'`, `'2000'` for the first order crossing each threshold, NULL otherwise

**Requirements:**
- Use `orders` table, exclude NULL amounts
- Use a window SUM for cumulative spend
- For milestone: an order crosses a threshold if cumulative_spend >= threshold AND the previous cumulative_spend < threshold
- Only one milestone per order (use the highest threshold crossed if multiple)
- Order by `user_id ASC`, `created_at ASC`

**Difficulty Rating:** 5/5

WITH users_cumulative_spend AS (
SELECT 
	id AS order_id,
	created_at,
	user_id,
	amount,
	SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS user_cumulative_spend
FROM orders
),
users_prev_spend AS (
SELECT 
	*,
	LAG(user_cumulative_spend) OVER (PARTITION BY user_id ORDER BY user_cumulative_spend) AS previous_cumulative_spend
FROM users_cumulative_spend
)
SELECT 
	order_id,
	user_id,
	amount AS order_amount,
	user_cumulative_spend AS cumulative_spend,
	CASE 
		WHEN user_cumulative_spend > 2000 AND previous_cumulative_spend < 2000 THEN '2000'
		WHEN user_cumulative_spend > 1000 AND previous_cumulative_spend < 1000 THEN '1000'
		WHEN user_cumulative_spend > 500 AND previous_cumulative_spend < 500 THEN '500' ELSE NULL
	END AS milestone
FROM users_prev_spend


---

## Task 3: Product Category Affinity — Categories Bought Together

**Scenario:**
The recommendations team wants to know which product categories are most frequently purchased together in the same order.

Find all pairs of distinct categories that appear together in at least 3 orders, ranked by co-occurrence frequency.

**Expected Output Columns:**
- `category_a` (text)
- `category_b` (text)
- `co_occurrence_count` (bigint)
- `co_occurrence_rank` (bigint)

**Requirements:**
- Use `orders_products`, `products`, `product_categories`
- A pair is `category_a < category_b` (alphabetically) to avoid duplicates
- Count distinct orders containing both categories
- Only pairs appearing in >= 3 orders
- Order by `co_occurrence_count DESC`, `category_a ASC`

**Difficulty Rating:** 5/5

WITH orders_categories_deduplicated AS (
SELECT 
	op1.order_id AS order_id1,
	pc1."name" AS category_a,
	pc2."name" AS category_b
FROM orders_products op1
JOIN products p1 ON op1.product_id = p1.id
JOIN product_categories pc1 ON p1.category_id = pc1.id
JOIN orders_products op2 ON op1.order_id = op2.order_id
JOIN products p2 ON op2.product_id = p2.id
JOIN product_categories pc2 ON p2.category_id = pc2.id
WHERE pc1.id < pc2.id
),
categories_cooccurences AS (
SELECT
	category_a,
	category_b,
	COUNT(DISTINCT(order_id1)) AS co_occurence_count
FROM orders_categories_deduplicated
GROUP BY category_a, category_b
)
SELECT 
	*,
	RANK() OVER (ORDER BY co_occurence_count DESC) AS co_occurence_rank
FROM categories_cooccurences
WHERE co_occurence_count >= 3

Handled the task with grace.
I had to add DISTINCT to filter out situations where two different products, even with the same name but different id from the same category were bought together.

---

## Submission Instructions

1. Task 1 — Country/year hierarchy with formatted Level 3 (4/5)
2. Task 2 — Running total with milestone flags (5/5)
3. Task 3 — Category co-occurrence pairs (5/5)
