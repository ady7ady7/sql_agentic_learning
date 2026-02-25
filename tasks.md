# Daily SQL Practice Tasks

**Generated:** 2026-02-25
**Week 11, Day 3 Focus:** HackerRank Hard — Multi-CTE Combinations + EPOCH Practice + Hierarchy

---

## Task 1: 3-Level Hierarchy — Users by Registration Month and Country

**Scenario:**
Build a 3-level hierarchy over user registration data:
- Level 1: `'All Users'`
- Level 2: Distinct registration months (format: `YYYY-MM`, pulled dynamically from `users.created_at`)
- Level 3: For each month, the 3 countries with the most registrations (show country name, exclude NULLs)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — month string at Level 2, country at Level 3
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Pre-aggregate distinct months and top-3-countries-per-month before the recursive CTE
- Format months as text using `TO_CHAR(created_at, 'YYYY-MM')`
- Exclude NULL countries at Level 3
- Termination condition required

**Difficulty Rating:** 4/5

WITH RECURSIVE distinct_registration_months AS (
SELECT 
	DISTINCT date_trunc('Month', created_at) AS registration_month
FROM users
ORDER BY registration_month
),
countries_registrations AS (
SELECT 
	date_trunc('Month', created_at) AS registration_month,
	country,
	COUNT(*) AS registered_users_cnt
FROM users
WHERE country IS NOT NULL
GROUP BY date_trunc('Month', created_at), country
ORDER BY registration_month
),
monthly_registration_rank AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY registration_month ORDER BY registered_users_cnt DESC) AS registration_rank
FROM countries_registrations
),
top_three_countries_registration_rank AS (
SELECT * FROM monthly_registration_rank
WHERE registration_rank <= 3
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Users'::TEXT AS name,
	NULL::TEXT AS parent_name,
	'All Users' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(TO_CHAR(drm.registration_month, 'YYYY-MM'), ttcr.country::TEXT),
	h.name,
	h.PATH || ' > ' || COALESCE(TO_CHAR(drm.registration_month, 'YYYY-MM'), ttcr.country::TEXT)
FROM HIERARCHY h
LEFT JOIN distinct_registration_months drm ON h.LEVEL = 1
LEFT JOIN top_three_countries_registration_rank ttcr ON h.LEVEL = 2 AND h.name = TO_CHAR(ttcr.registration_month, 'YYYY-MM')
WHERE h.LEVEL < 3
)
SELECT * FROM hierarchy


---

## Task 2: Ticket Resolution Time Analysis (EPOCH Practice)

**Scenario:**
The support team wants to measure how long it takes to resolve tickets. For each resolved ticket (status = `'resolved'`), calculate the time between creation and last update (as a proxy for resolution time).

Then segment tickets by resolution speed:
- `fast`: resolved in under 1 hour
- `medium`: 1 to 24 hours
- `slow`: more than 24 hours

**Expected Output Columns:**
- `ticket_id` (bigint)
- `priority` (text)
- `resolution_hours` (numeric) — hours between created_at and updated_at, rounded to 2 decimals
- `resolution_segment` (text)

**Requirements:**
- Use `chat_tickets` table
- Only include tickets where `status = 'resolved'`
- Use `EXTRACT(EPOCH FROM ...)` to calculate the interval in seconds, then convert to hours
- Order by `resolution_hours ASC`

**Difficulty Rating:** 3/5

WITH resolved_tickets_resolUtion_times AS (
SELECT 
  cm.ticket_id,
  ct.created_at AS ticket_creation_time,
  cm.created_at AS ticket_resolve_time,
  EXTRACT('Epoch' FROM cm.created_at - ct.created_at) / 60 AS resolution_minutes
FROM chat_messages cm
JOIN chat_tickets ct ON cm.ticket_id = ct.id
WHERE cm.status = 'resolved'
)
SELECT 
	*,
	CASE 
		WHEN resolution_minutes <= 5 THEN 'fast'
		WHEN resolution_minutes <= 10 THEN 'medium'
		WHEN resolution_minutes > 10 THEN 'slow'
	END AS resolution_segment
FROM resolved_tickets_resolUtion_times
ORDER BY resolution_minutes

Adapted this to the acutal data, where 26 MINUTES was the maximum amount, so I adapted the resolution to minutes and used relevant values as filters for fast/medium/slow.


---

## Task 3: Product Affinity — Frequently Co-Purchased Products

**Scenario:**
The recommendations team wants to find product pairs that are frequently bought together in the same order. Find all pairs of products that appear together in at least 2 orders.

**Expected Output Columns:**
- `product_a_id` (integer)
- `product_b_id` (integer)
- `product_a_name` (text)
- `product_b_name` (text)
- `co_purchase_count` (bigint) — number of orders containing both products
- `co_purchase_rank` (bigint) — rank by co_purchase_count DESC

**Requirements:**
- Use `orders_products` and `products` tables
- A pair is defined as `product_a_id < product_b_id` to avoid duplicates
- Only include pairs appearing in >= 2 orders
- Order by `co_purchase_count DESC`, `product_a_id ASC`

**Difficulty Rating:** 5/5

There's a lot of co purchase count 3 and 2, so I used row number to actually rank them based on co_purchase_count and product_a_id ASC, as otherwise it wouldn't make sense :))

WITH orders_product_pairs AS (
SELECT 
	op1.product_id AS product_a_id,
	op2.product_id AS product_b_id,
	p1."name" AS product_a_name,
	p2.name AS product_b_name,
	COUNT(op1.order_id) AS co_purchase_count
FROM orders_products op1
JOIN orders_products op2 ON op1.order_id = op2.order_id
JOIN products p1 ON op1.product_id = p1.id
JOIN products p2 ON op2.product_id = p2.id
WHERE op1.product_id > op2.product_id
GROUP BY op1.product_id, op2.product_id, p1."name", p2."name"
ORDER BY co_purchase_count DESC, product_a_id
)
SELECT 
	*,
	ROW_NUMBER() OVER (ORDER BY co_purchase_count DESC)
FROM orders_product_pairs
WHERE co_purchase_count >= 2

---

## Submission Instructions

1. Task 1 — Registration month/country hierarchy (4/5)
2. Task 2 — Ticket resolution time with EPOCH (3/5)
3. Task 3 — Product affinity pairs (5/5)
