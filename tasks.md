# Daily SQL Practice Tasks

**Generated:** 2026-03-25
**Week 15, Day 3 Focus:** Time-Proximity Edge Cases + PIVOT Complex + Anti-Join Combo

---

## Task 1: Time-Proximity — User Transaction Sessions with Edge Cases

**Scenario:**
Group each user's transactions into sessions where consecutive transactions are within **30 minutes** of each other. This dataset is designed to test edge cases — pay attention to transactions that straddle the hour boundary.

Use this inline data:

```sql
WITH events (user_id, event_time, amount) AS (
    VALUES
    (1, '2024-01-01 08:00'::timestamp, 100),
    (1, '2024-01-01 08:25'::timestamp, 200),
    (1, '2024-01-01 08:52'::timestamp, 150),
    (1, '2024-01-01 09:18'::timestamp, 300),
    (1, '2024-01-01 10:00'::timestamp, 50),
    (1, '2024-01-01 10:20'::timestamp, 75),
    (2, '2024-01-01 09:00'::timestamp, 500),
    (2, '2024-01-01 09:28'::timestamp, 200),
    (2, '2024-01-01 10:05'::timestamp, 100)
)
```

Note: user 1's events at 08:52 and 09:18 are 26 minutes apart — same session. Events at 09:18 and 10:00 are 42 minutes apart — new session.

**Expected Output Columns:**
- `user_id` (integer)
- `session_id` (bigint) — sequential per user (1, 2, 3...)
- `session_start` (timestamp)
- `session_end` (timestamp)
- `event_count` (bigint)
- `total_amount` (numeric)
- `duration_minutes` (numeric) — use EPOCH, rounded to 1 decimal

**Requirements:**
- LAG → is_new_session flag → SUM() OVER → GROUP BY pattern
- Use `EXTRACT(EPOCH FROM ...) / 60` for duration
- Order by `user_id ASC`, `session_start ASC`

**Difficulty Rating:** 4/5

WITH events (user_id, event_time, amount) AS (
    VALUES
    (1, '2024-01-01 08:00'::timestamp, 100),
    (1, '2024-01-01 08:25'::timestamp, 200),
    (1, '2024-01-01 08:52'::timestamp, 150),
    (1, '2024-01-01 09:18'::timestamp, 300),
    (1, '2024-01-01 10:00'::timestamp, 50),
    (1, '2024-01-01 10:20'::timestamp, 75),
    (2, '2024-01-01 09:00'::timestamp, 500),
    (2, '2024-01-01 09:28'::timestamp, 200),
    (2, '2024-01-01 10:05'::timestamp, 100)
),
events_lag AS (
SELECT 
	*,
	LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) AS prev_event_time
FROM events
),
events_new_sessions AS (
SELECT 
	*,
	CASE WHEN prev_event_time IS NULL OR event_time - prev_event_time > INTERVAL '30 Minutes' THEN 1 ELSE 0 END AS is_new_session
FROM events_lag
),
event_session_ids AS (
SELECT
	*,
	SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY event_time) AS session_id
FROM events_new_sessions
)
SELECT 
	user_id,
	session_id,
	MIN(event_time) AS session_start,
	MAX(event_time) AS session_end,
	COUNT(*) AS event_count,
	SUM(amount) AS total_amount,
	EXTRACT(EPOCH FROM MAX(event_time) - MIN(event_time)) / 60 AS duration_minutes 
FROM event_session_ids
GROUP BY user_id, session_id
ORDER BY USER_id, session_start


Again, I love this pattern and find it very useful.




---

## Task 2: PIVOT — Delivery Status Revenue by Product Category

**Scenario:**
The operations team wants a revenue breakdown showing, for each product category, how much revenue is associated with each delivery status.

**Expected Output Columns:**
- `category_name` (text)
- `pending_revenue` (numeric) — rounded to 2 decimals
- `delivered_revenue` (numeric) — rounded to 2 decimals
- `total_revenue` (numeric) — rounded to 2 decimals

**Requirements:**
- Use `orders`, `orders_products`, `products`, `product_categories`, `deliveries`
- Revenue = `quantity × price`
- Join orders to deliveries to get status
- PIVOT on delivery status using `SUM(...) FILTER (WHERE status = '...')`
- Order by `total_revenue DESC`

**Difficulty Rating:** 4/5

SELECT 
	pc."name" AS category_name,
	SUM(p.price * op.quantity) FILTER (WHERE d.status = 'pending') AS pending_revenue,
	SUM(p.price * op.quantity) FILTER (WHERE d.status = 'delivered') AS delivered_revenue,
	SUM(p.price * op.quantity) AS total_revenue
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.products p ON op.product_id = p.id
JOIN crappy_data_db.product_categories pc ON p.category_id = pc.id
JOIN crappy_data_db.deliveries d ON d.order_id = op.order_id
GROUP BY pc."name"
ORDER BY total_revenue DESC

I followed your instructions at first, but I quickly realized that this is weird AND VERY DANGEROUS to calculate the total revenue with the deliveries JOINED, as effectively it multiplies the total revenue by every delivery status that IS THERE. IT'S ERRATIC WHAT YOU WANTED ME TO DO THERE. I've added another CTE in a reverse logic WITHOUT ADDING DELIVERIES to avoid that bias to calculate total_revenue.


WITH pending_delivered_revenues AS (
SELECT 
	pc."name" AS category_name,
	SUM(p.price * op.quantity) FILTER (WHERE d.status = 'pending') AS pending_revenue,
	SUM(p.price * op.quantity) FILTER (WHERE d.status = 'delivered') AS delivered_revenue
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.products p ON op.product_id = p.id
JOIN crappy_data_db.product_categories pc ON p.category_id = pc.id
JOIN crappy_data_db.deliveries d ON d.order_id = op.order_id
GROUP BY pc."name"
)
SELECT 
	pdr.category_name,
	pdr.delivered_revenue,
	pdr.pending_revenue,
	SUM(p.price * op.quantity) AS total_revenue
FROM pending_delivered_revenues pdr
JOIN crappy_data_db.product_categories pc ON pc."name" = pdr.category_name
JOIN crappy_data_db.products p ON p.category_id = pc.id
JOIN crappy_data_db.orders_products op ON p.id = op.product_id
GROUP BY pdr.category_name, pdr.delivered_revenue, pdr.pending_revenue

It turns out that total_revenue is equal to pending_revenue, which makes total sense.
delivered_revenue is smaller and it also makes sense as not all of the orders were delivered successfully.

Also, mind that I AM AWARE THAT YOU WANTED the results rounded to 2 decimals, AND THEY ARE ROUNDED TO 2 DECIMALS, so I didn't add the unnecessary ROUND there.

---

## Task 3: Anti-Join — Categories With No Sales in the Last 3 Months

**Scenario:**
The product team wants to identify product categories that have had zero sales in the last 3 months (relative to the most recent order date in the dataset — do not use `NOW()`).

**Expected Output Columns:**
- `category_id` (integer)
- `category_name` (text)
- `last_sale_date` (date) — most recent sale date for this category across all time, NULL if never sold

**Requirements:**
- Use `product_categories`, `products`, `orders_products`, `orders`
- "Last 3 months" = within 3 months before the most recent order date in the dataset
- Use `NOT EXISTS` to identify categories with no sales in that window
- Include `last_sale_date` — if a category has older sales but none recently, show when it last sold
- Order by `last_sale_date ASC NULLS FIRST`

**Difficulty Rating:** 5/5

THERE ARE NO SUCH CATEGORIES.
I've used a different approach here but it works just fine.


WITH categories_last_orders AS (
SELECT 
	pc.name AS category_name,
	MAX(o.created_at) AS last_order_time
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.orders o ON op.order_id = o.id
JOIN crappy_data_db.products p ON p.id = op.product_id
JOIN crappy_data_db.product_categories pc ON pc.id = p.category_id
GROUP BY pc.name
),
qualified_orders AS (
SELECT 
	* 
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.orders o ON op.order_id = o.id
JOIN crappy_data_db.products p ON p.id = op.product_id
JOIN crappy_data_db.product_categories pc ON pc.id = p.category_id
JOIN categories_last_orders clo ON clo.category_name = pc."name"
WHERE last_order_time - o.created_at < INTERVAL '3 Months'
)
SELECT 
	DISTINCT category_id, category_name, last_order_time
FROM qualified_orders

---

## Submission Instructions

1. Task 1 — Transaction session grouping with edge cases (4/5)
2. Task 2 — Delivery status revenue PIVOT by category (4/5)
3. Task 3 — Anti-join: categories with no recent sales (5/5)
