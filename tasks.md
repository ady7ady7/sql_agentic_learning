# Daily SQL Practice Tasks

**Generated:** 2026-05-01
**Week 20, Day 5 Focus:** PERCENT_RANK revisit + cohort retention with LEFT JOIN pattern

---

## Task 1: PERCENT_RANK — Transaction Amount Ranking per User

**Scenario:**
The risk team wants to see where each transaction sits within the user's personal transaction history — as a percentile.

For every transaction show:
- `transaction_id`
- `user_id`
- `amount`
- `pct_rank` — `PERCENT_RANK()` of this transaction within the user's transactions by amount, rounded to 2 decimals
- `is_top_10_pct` — boolean, true if `pct_rank >= 0.9`

**Tables:** `crappy_data_db.transactions`

**Requirements:**
- Exclude NULL amounts
- Order by `user_id ASC`, `amount DESC`

**Difficulty Rating:** 3/5

WITH users_pct_ranks AS (
SELECT 
	*,
	ROUND(PERCENT_RANK() OVER (PARTITION BY user_id ORDER BY amount)::numeric, 2) AS pct_rank
FROM crappy_data_db.transactions t
WHERE AMOUNT IS NOT null
)
SELECT 
	*,
	pct_rank >= 0.9 AS is_top_10_pct
FROM users_pct_ranks 







---

## Task 2: Cohort Retention — LEFT JOIN Pattern

**Scenario:**
The growth team wants to know what share of users who placed their first order in 2025 came back to order again within 3 months of their first order.

For each user show:
- `user_id`
- `first_order_month` — DATE_TRUNC to month of their first ever order
- `returned_within_3_months` — boolean, true if they placed any order between 1 and 3 months after `first_order_month`

Only include users whose first order was in 2025.

**Tables:** `crappy_data_db.orders`

**Requirements:**
- CTE 1: find each user's first order date using `MIN(created_at)`
- Final SELECT: LEFT JOIN back to orders to check for any order where `created_at >= first_order_date + INTERVAL '1 month'` AND `created_at < first_order_date + INTERVAL '4 months'`
- Use `IS NOT NULL` on the joined order to derive the boolean
- Order by `user_id ASC`

**Difficulty Rating:** 4/5

I've done it in a different way, and IMO it's very effective as well.
I simply took everyone's SECOND order - so it automatically excludes everyone who didn't make the second order, and checked it if it was in the given interval

WITH users_orders AS (
SELECT 
	*,
	DATE_TRUNC('Month', o.created_at) AS month_,
	FIRST_VALUE(DATE_TRUNC('Month', o.created_at)) OVER (PARTITION BY user_id ORDER BY created_at) AS first_order_month,
	row_number() OVER (PARTITION BY USER_ID ORDER BY created_at) AS rn
FROM crappy_data_db.orders o
)
SELECT 
	*,
	CASE WHEN MONTH_ < first_order_month + INTERVAL '1' MONTH THEN TRUE ELSE FALSE END AS returned_within_3_months
FROM users_orders
WHERE EXTRACT('YEAR' FROM first_order_month) = 2025 AND rn = 2

---

## Submission Instructions

1. Task 1 — PERCENT_RANK with top 10% flag (3/5)
2. Task 2 — Cohort 3-month return check via LEFT JOIN (4/5)
