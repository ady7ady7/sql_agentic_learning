# Daily SQL Practice Tasks

**Generated:** 2026-03-06
**Week 12, Day 5 — FINAL SESSION: Full HackerRank Hard Exam Simulation**

---

## Task 1: 3-Level Hierarchy — Support Ticket Statuses, Priorities, and Top Users

**Scenario:**
Build a 3-level hierarchy over support data:
- Level 1: `'All Tickets'`
- Level 2: Distinct ticket statuses (dynamic, from `chat_tickets`)
- Level 3: For each status, the 3 priorities with the highest average resolution time in minutes (show as `'priority (avg_minutes avg min)'`, rounded to 1 decimal)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — status at Level 2, formatted string at Level 3
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Join `chat_tickets` to calculate avg resolution time per status+priority using `EXTRACT(EPOCH FROM updated_at - created_at) / 60`
- Pre-aggregate avg resolution time per status+priority, then rank before the recursive CTE
- Format Level 3 name as: `priority || ' (' || avg_minutes::text || ' avg min)'`
- Termination condition required

**Difficulty Rating:** 5/5

WITH RECURSIVE tickets_res_times AS (
SELECT 
	ct.id AS ticket_id,
	ct.status,
	ct.priority,
	ct.created_at AS ticket_creation_time,
	EXTRACT(EPOCH FROM ct.updated_at - ct.created_at)/60 AS ticket_resolution_minutes
FROM chat_tickets ct
),
distinct_statuses AS (
SELECT DISTINCT status FROM chat_tickets
),
tickets_statuses_priorities_avg_resolution AS (
SELECT 
	status,
	priority,
	ROUND(AVG(ticket_resolution_minutes), 1) AS avg_resolution_minutes
FROM tickets_res_times
GROUP BY status, priority
),
statuses_ranks AS (
SELECT 
	*,
	rank() OVER (PARTITION BY status ORDER BY avg_resolution_minutes) AS status_resolution_rank
FROM tickets_statuses_priorities_avg_resolution
),
top_three_status_resolution_ranks AS (
SELECT * FROM statuses_ranks
WHERE status_resolution_rank <= 3
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Tickets' AS name,
	NULL::TEXT AS parent_name,
	'All Tickets' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(ds.status, tts.priority || ' (' || tts.avg_resolution_minutes::TEXT || ' avg min)'),
	h.name,
	h.PATH || ' > ' || COALESCE(ds.status, tts.priority || ' (' || tts.avg_resolution_minutes::TEXT || ' avg min)')
FROM HIERARCHY h
LEFT JOIN distinct_statuses ds ON h.LEVEL = 1
LEFT JOIN top_three_status_resolution_ranks tts ON h.LEVEL = 2 AND tts.status = h.name
WHERE h.LEVEL < 3
)
SELECT * FROM hierarchy


---

## Task 2: User Lifetime Value Segments with Gaps-and-Islands

**Scenario:**
The growth team wants to identify high-value users who have maintained consistent monthly ordering habits. For each user, calculate their total lifetime order revenue and their longest consecutive monthly ordering streak. Then segment them:

- `champion`: lifetime_revenue >= 1000 AND longest_streak >= 3 months
- `loyal`: lifetime_revenue >= 500 AND longest_streak >= 2 months
- `at_risk`: lifetime_revenue >= 200 AND longest_streak = 1
- `other`: everything else

**Expected Output Columns:**
- `user_id` (integer)
- `lifetime_revenue` (numeric) — rounded to 2 decimals
- `longest_streak` (bigint) — in months
- `segment` (text)
- `segment_rank` (bigint) — ranked by lifetime_revenue DESC within each segment

**Requirements:**
- Use `orders` table
- Lifetime revenue = SUM of all order amounts per user
- Longest streak = gaps-and-islands on monthly order activity (DATE_TRUNC to month, ROW_NUMBER subtraction pattern)
- Order by `segment ASC`, `segment_rank ASC`

**Difficulty Rating:** 5/5

WITH users_revenues AS (
SELECT 
	user_id,
	SUM(amount) AS lifetime_revenue
FROM orders
GROUP BY user_id
),
orders_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
FROM orders
),
users_order_months AS (
SELECT 
	user_id,
	month_
FROM orders_months
GROUP BY user_id, month_
ORDER BY user_id
),
users_months_rn AS (
SELECT 
	*,
	ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY month_) AS rn
FROM users_order_months
),
users_streak_keys AS (
SELECT 
	*,
	month_ - rn * INTERVAL '1' MONTH AS streak_key,
	LAST_VALUE(month_) OVER (PARTITION BY user_id ORDER BY month_) AS prev_month
FROM users_months_rn 
),
users_streaks AS (
SELECT 
	user_id,
	streak_key,
	COUNT(*) AS streak_length
FROM users_streak_keys
GROUP BY user_id, streak_key
),
users_longest_streaks AS (
SELECT 
	user_id,
	MAX(streak_length) AS longest_streak
FROM users_streaks
GROUP BY user_id
),
users_segments AS (
SELECT 
	ur.user_id,
	ur.lifetime_revenue,
	uls.longest_streak,
	CASE 
		WHEN ur.lifetime_revenue >= 1000 AND uls.longest_streak >= 3 THEN 'champion'
		WHEN ur.lifetime_revenue >= 500 AND uls.longest_streak >= 2 THEN 'loyal'
		WHEN ur.lifetime_revenue >= 20 AND uls.longest_streak >= 1 THEN 'at_risk'
	END AS segment
	FROM users_revenues ur
JOIN users_longest_streaks uls ON ur.user_id = uls.user_id
)
SELECT 
	*,
	RANK() OVER (PARTITION BY segment ORDER BY lifetime_revenue DESC, longest_streak DESC) AS segment_rank
FROM users_segments

Definitely one of the longest queries we've ever done in this learning process, but I've managed to do it with full understanding from end to end. I'm also 100% sure that I've done the task perfectly, despite its complexity.

---

## Task 3: Complete Order Funnel Analysis

**Scenario:**
The analytics team wants a full funnel report showing how orders progress through the system. For each month, show:
- Total orders placed
- Orders that have a delivery record (fulfillment rate)
- Of those with deliveries, orders with status `'delivered'` (delivery success rate)
- Average hours from order creation to delivery creation (for orders that have a delivery), using EPOCH

**Expected Output Columns:**
- `month` (date) — truncated to month
- `total_orders` (bigint)
- `orders_with_delivery` (bigint)
- `delivered_orders` (bigint)
- `fulfillment_rate_pct` (numeric) — orders_with_delivery / total_orders * 100, rounded to 1 decimal
- `delivery_success_rate_pct` (numeric) — delivered_orders / orders_with_delivery * 100, rounded to 1 decimal
- `avg_hours_to_delivery` (numeric) — rounded to 2 decimals, NULL if no deliveries that month

**Requirements:**
- Use `orders` and `deliveries` tables
- Use LEFT JOIN to preserve orders without deliveries
- Use EPOCH for hours calculation
- Order by `month ASC`

**Difficulty Rating:** 4/5

WITH orders_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
FROM orders
),
monthly_total_orders AS (
SELECT 
	month_,
	COUNT(*) AS total_orders
FROM orders_months
GROUP BY month_
),
orders_with_delivery_status AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
FROM deliveries
),
orders_statuses_cnts AS (
SELECT 
	month_,
	status,
	COUNT(*) AS order_cnt
FROM orders_with_delivery_status
GROUP BY month_, status
),
orders_with_delivery_cnts AS (
SELECT 
	month_,
	SUM(order_cnt) AS orders_with_delivery
FROM orders_statuses_cnts
WHERE status = 'pending'
GROUP BY month_
),
orders_delivery_times AS (
SELECT 
	om.month_,
	om.created_at AS order_placed_at,
	d.created_at AS delivered_at,
	EXTRACT(EPOCH FROM d.created_at - om.created_at)/3600 AS order_delivery_time_in_hours
FROM orders_months om
JOIN deliveries d ON om.id = d.order_id
WHERE d.status = 'delivered'
),
orders_avg_delivery_times AS (
SELECT 
	month_,
	ROUND(AVG(order_delivery_time_in_hours), 2) AS avg_hours_to_delivery
FROM orders_delivery_times
GROUP BY month_
),
months_delivery_metrics AS (
SELECT 
	mto.month_,
	mto.total_orders,
	owd.orders_with_delivery,
	osc.order_cnt AS delivered_orders,
	ROUND(owd.orders_with_delivery / mto.total_orders * 100, 1) AS fulfillment_rate_pct,
	ROUND(osc.order_cnt / owd.orders_with_delivery * 100, 1) AS delivery_success_rate_pct,
	oad.avg_hours_to_delivery
FROM monthly_total_orders mto
LEFT JOIN orders_with_delivery_cnts owd ON mto.month_ = owd.month_
LEFT JOIN orders_statuses_cnts osc ON mto.month_ = osc.month_ AND osc.status = 'delivered'
LEFT JOIN orders_avg_delivery_times oad ON mto.month_ = oad.month_
)
SELECT * FROM months_delivery_metrics
ORDER BY month_

Another very long and complex query that needed multiple steps and thinking, but I handled it well.
---

## Submission Instructions

1. Task 1 — Ticket hierarchy with avg resolution time at Level 3 (5/5)
2. Task 2 — User lifetime value + monthly streak segmentation (5/5)
3. Task 3 — Complete order funnel analysis (4/5)

---

*This is the final session of the 12-week SQL Advanced program. Give it everything.*
