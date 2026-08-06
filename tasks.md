# SQL Tasks — 2026-08-06 (Week 33, Day 4)

**Dataset:** orders / transactions / users  
**Focus:** ROWS vs RANGE (real case) · YoY comparison · Funnel analysis

---

## Task 1 — End-of-Day Running Total with RANGE
**Difficulty: 5/5**

**Business question:**  
For each user, for each transaction, show the cumulative sum of all transactions up to and including **the end of that calendar day** — meaning every transaction on the same day gets the same "end of day" running total, regardless of the order they appear within the day.

Do NOT pre-aggregate by day. Work directly from `transactions`, one row per transaction. Use `created_at::date` as the sort key in the window so that all transactions on the same day are treated as a peer group.

Then, in a comment above your query, explain in 2–3 sentences: what would happen if you used `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` instead — how would the output differ for a user with 3 transactions on the same day?

**Expected output columns:**  
`user_id, id, created_at, amount, end_of_day_running_total`

Order by `user_id`, `created_at::date`, `id`.

**Difficulty: 5/5**

SELECT 
	user_id,
	id,
	created_at,
	amount,
	SUM(amount) OVER (PARTITION BY user_id ORDER BY t.created_at::date) AS end_of_day_running_total
FROM crappy_data_db.transactions t
ORDER BY user_id, created_at::date, id

If I used ROWS BETWEEN I'd simply get the running total UP TO THE CURRENT transaction, not all the transactions from that given day, easy as that. For the needs of this exercise it would skew the results.




---

## Task 2 — Year-over-Year Revenue
**Difficulty: 4/5**

**Business question:**  
For each month, compute total order revenue and compare it to the same month one year prior. Show the absolute difference and the percentage change.

If there is no data for the prior year month, show NULLs for the comparison columns — do not exclude the row.

**Expected output columns:**  
`order_month, total_revenue, prev_year_revenue, yoy_diff, yoy_pct_change`

Where:
- `order_month` = DATE_TRUNC('month', created_at)
- `yoy_diff` = total_revenue − prev_year_revenue
- `yoy_pct_change` = ROUND((yoy_diff / prev_year_revenue) * 100, 2)

Order by `order_month`.

**Difficulty: 4/5**


WITH orders_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
FROM crappy_data_db.orders o
),
monthly_revs AS (
SELECT
	month_,
	SUM(amount) AS total_revenue
FROM orders_months
GROUP BY month_
ORDER BY month_
),
prev_year_revs AS (
SELECT 
	*,
	COALESCE(LAG(total_revenue, 12) OVER (ORDER BY month_), 0) AS prev_year_revenue
FROM monthly_revs
)
SELECT 
	*,
	total_revenue - prev_year_revenue AS yoy_diff,
	ROUND((total_revenue - prev_year_revenue)::numeric / total_revenue::NUMERIC * 100, 2) AS yoy_pct_change
FROM prev_year_revs
WHERE prev_year_revenue > 0



Everything done as you wanted.



---

## Task 3 — Conversion Funnel
**Difficulty: 4/5**

**Business question:**  
Measure the conversion funnel across three stages:
1. **All users** — total registered users
2. **Buyers** — users who placed at least one order
3. **Delivered** — users who have at least one order with a delivery record of any status

For each stage show the count and the percentage relative to the top of the funnel (all users).

Then add a fourth row showing the **drop-off between buyers and delivered** — what % of buyers have no delivery record at all.

**Expected output columns:**  
`stage, user_count, pct_of_total`

Stages in order: `all_users`, `buyers`, `delivered`, `buyers_no_delivery`.

**Difficulty: 4/5**


WITH users_orders_deliveries AS (
SELECT 
	u.id AS user_id,
	COUNT(o.id) AS ordered_,
	COUNT(d.id) FILTER (WHERE d.status = 'delivered') AS delivered_
FROM crappy_data_db.users u
LEFT JOIN crappy_data_db.orders o ON u.id = o.user_id
LEFT JOIN crappy_data_db.deliveries d ON d.order_id = o.id
GROUP BY u.id
)
SELECT
	'all_users' AS stage,
	COUNT(user_id) AS user_count,
	100.0 AS pct_of_total
FROM users_orders_deliveries
UNION ALL
SELECT
	'buyers',
	COUNT(*) FILTER (WHERE ordered_ > 0),
	ROUND(COUNT(*) FILTER (WHERE ordered_ > 0) / COUNT(user_id)::NUMERIC * 100, 2)
FROM users_orders_deliveries
UNION ALL
SELECT
	'delivered',
	COUNT(*) FILTER (WHERE delivered_ > 0),
	ROUND(COUNT(*) FILTER (WHERE delivered_ > 0) / COUNT(user_id)::NUMERIC * 100, 2)
FROM users_orders_deliveries

---

## Submission Instructions

Paste your queries below each task. For Task 1, include the comments — they're part of the answer.
