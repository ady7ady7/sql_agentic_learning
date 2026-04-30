# Daily SQL Practice Tasks

**Generated:** 2026-04-30
**Week 20, Day 4 Focus:** Anti-join (NOT EXISTS) + YoY with NULLIF + window frame comparison

---

## Task 1: Anti-Join — Orders With No Delivery (NOT EXISTS)

**Scenario:**
The ops team wants to find orders that have no delivery record at all — they may have slipped through the system.

Write this using `NOT EXISTS` only. Start from `orders`, check against `deliveries`.

**Expected Output Columns:** `order_id`, `user_id`

**Tables:** `crappy_data_db.orders`, `crappy_data_db.deliveries`

**Order by:** `order_id ASC`

**Difficulty Rating:** 3/5


SELECT o.id 
FROM crappy_data_db.orders o
WHERE NOT EXISTS
(SELECT * FROM crappy_data_db.deliveries d
WHERE d.order_id = o.id
)


There are no such orders - all of them have delivery records, and I remember that as I've used to check that in different tasks. Before the next week we need to add another scheme to our data. Let's make it one of the tasks for tomorrow - to create a new schema, add data there and let you know about it, so you can put all the related data in your memory, so you'll remember that forever. FYI: I have some schemas to chosoe from, as I've purchased a premium package for SQL course which included tasks with different datasets. They all have CREATE TABLE + INSERT DATA code in them, with new data. I just need to get through them and pick one.

As an alternative, we could think about working with a real dataset with sales of smarthphones, which is exceptionally big.


---

## Task 2: YoY — Monthly Order Revenue with NULLIF Division

**Scenario:**
The finance team wants month-by-month order revenue with prior year comparison and percentage change.

For each month show:
- `month` (DATE_TRUNC to month)
- `revenue` — total order amount for that month
- `prev_year_revenue` — same month one year prior via `LAG(revenue, 12)`
- `yoy_pct_change` — `(revenue - prev_year_revenue) / NULLIF(prev_year_revenue, 0) * 100`, rounded to 1 decimal, NULL if no prior year

**Tables:** `crappy_data_db.orders`

**Requirements:**
- Exclude NULL amounts
- Use `NULLIF(prev_year_revenue, 0)` in the denominator — not just `prev_year_revenue`
- Order by `month ASC`

**Difficulty Rating:** 4/5

WITH orders_years AS (
SELECT 
*,
DATE_TRUNC('Month', created_at) AS MONTH,
DATE_TRUNC('Year', created_at) AS year
FROM crappy_data_db.orders o
),
monthly_revenues AS (
SELECT
	MONTH,
	sum(AMOUNT) AS revenue
FROM orders_years
GROUP BY MONTH
)
SELECT 
	*,
	lag(revenue, 12) OVER (ORDER BY MONTH) AS prev_year_revenue,
	round(((revenue - lag(revenue, 12) OVER (ORDER BY MONTH)) / NULLIF(lag(revenue, 12) OVER (ORDER BY MONTH), 0) * 100)::NUMERIC, 1) AS yoy_pct_change
FROM monthly_revenues


---

## Task 3: Window Frame Comparison — Three Ways to SUM

**Scenario:**
The finance team wants to understand cumulative vs rolling vs total revenue patterns on orders per user.

For every order write **one query** that produces all three columns side by side:
- `order_id`
- `user_id`
- `amount`
- `running_total` — cumulative SUM from the first order to now: `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`
- `rolling_3` — rolling SUM over current + 2 prior orders: `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW`
- `partition_total` — total SUM for the entire user, same on every row: `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`

**Then answer in a comment:** What's the difference between `running_total` and `partition_total`, and when would you use each?

**Tables:** `crappy_data_db.orders`

**Requirements:**
- Exclude NULL amounts
- All three in a single SELECT, no CTEs needed
- Order by `user_id ASC`, `created_at ASC`

**Difficulty Rating:** 5/5

SELECT 
	id AS order_id,
	user_id,
	amount,
	SUM(amount) OVER (PARTITION BY USER_id ORDER BY created_at) AS running_total,
	SUM(amount) OVER (PARTITION BY USER_id ORDER BY created_at ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS rolling_3,
	SUM(amount) OVER (PARTITION BY USER_id ORDER BY created_at ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS partition_total
FROM crappy_data_db.orders o


Very interesting - that UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING.



---

## Submission Instructions

1. Task 1 — NOT EXISTS anti-join, orders with no delivery (3/5)
2. Task 2 — YoY monthly order revenue with NULLIF in denominator (4/5)
3. Task 3 — Three window frames in one query + explanation (5/5)
