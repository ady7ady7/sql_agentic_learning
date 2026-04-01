# Daily SQL Practice Tasks

**Generated:** 2026-04-01
**Week 16, Day 3 Focus:** NULLIF in Context + Type B Recursive CTE + Time-Proximity on Real Data

---

## Task 1: NULLIF — Safe Division in Order Metrics

**Scenario:**
The operations team wants per-user order metrics, but some users have orders with NULL amounts. A careless average would silently exclude those orders, skewing the per-user stats. They also want a conversion rate (orders with amount > 0 divided by total orders) — which requires safe division.

**Expected Output Columns:**
- `user_id` (integer)
- `total_orders` (bigint) — all orders including NULL amounts
- `orders_with_amount` (bigint) — orders where amount IS NOT NULL
- `avg_order_value` (numeric) — average of non-NULL amounts only (AVG handles this naturally)
- `conversion_rate` (numeric) — `orders_with_amount / NULLIF(total_orders, 0)` as a ratio, rounded to 3 decimals
- `has_null_amounts` (boolean) — true if any order has a NULL amount

**Requirements:**
- Use `orders` table
- Use `NULLIF(total_orders, 0)` in the division to guard against division by zero
- Only include users with at least 2 orders
- Order by `total_orders DESC`

**Difficulty Rating:** 3/5

WITH users_orders_metrics AS (
SELECT 
user_id,
COUNT(*) AS total_orders,
COUNT(*) FILTER (WHERE amount IS NOT NULL) AS orders_with_amount,
ROUND(AVG(NULLIF(amount, 0))::NUMERIC, 2) AS avg_order_amt
FROM crappy_data_db.orders o
GROUP BY user_id
)
SELECT 
	*,
	round(orders_with_amount::NUMERIC / total_orders, 3) AS conversion_rate,
	CASE WHEN orders_with_amount / total_orders < 1 THEN TRUE ELSE FALSE END AS has_null_amounts
FROM users_orders_metrics
WHERE total_orders >= 2
ORDER BY total_orders DESC


---

## Task 2: Type B Recursive CTE — Full Org Chart with Subordinate Count

**Scenario:**
Use this inline org chart data:

```sql
WITH employees (id, name, manager_id, department) AS (
    VALUES
    (1,  'CEO',        NULL::int, 'Executive'),
    (2,  'CTO',        1,         'Tech'),
    (3,  'CFO',        1,         'Finance'),
    (4,  'VP Eng',     2,         'Tech'),
    (5,  'VP Data',    2,         'Tech'),
    (6,  'FP&A Lead',  3,         'Finance'),
    (7,  'Eng Lead 1', 4,         'Tech'),
    (8,  'Eng Lead 2', 4,         'Tech'),
    (9,  'Data Lead',  5,         'Tech'),
    (10, 'Analyst',    6,         'Finance')
)
```

Traverse the full hierarchy and for each person show their depth, path, and how many direct reports they have.

**Expected Output Columns:**
- `id` (integer)
- `name` (text)
- `depth` (integer) — 1 for CEO
- `path` (text) — e.g. `'CEO -> CTO -> VP Eng -> Eng Lead 1'`
- `direct_reports` (bigint) — count of employees whose manager_id = this person's id

**Requirements:**
- Anchor: `WHERE manager_id IS NULL` — no hardcoding
- Recursive: JOIN employees back to CTE on `employees.manager_id = cte.id`
- `direct_reports`: compute via a subquery or LEFT JOIN to the same employees table
- Path separator: ` -> `
- Order by `path ASC`

**Difficulty Rating:** 4/5


WITH RECURSIVE employees (id, name, manager_id, department) AS (
    VALUES
    (1,  'CEO',        NULL::int, 'Executive'),
    (2,  'CTO',        1,         'Tech'),
    (3,  'CFO',        1,         'Finance'),
    (4,  'VP Eng',     2,         'Tech'),
    (5,  'VP Data',    2,         'Tech'),
    (6,  'FP&A Lead',  3,         'Finance'),
    (7,  'Eng Lead 1', 4,         'Tech'),
    (8,  'Eng Lead 2', 4,         'Tech'),
    (9,  'Data Lead',  5,         'Tech'),
    (10, 'Analyst',    6,         'Finance')
),
HIERARCHY AS (
SELECT 
	*,
	1 AS DEPTH,
	name AS PATH
FROM employees
WHERE manager_id IS NULL
UNION ALL
SELECT
	e.id,
	e.name,
	e.manager_id,
	e.department,
	h.DEPTH + 1,
	h.PATH || '->' || e.name
FROM HIERARCHY h JOIN employees e ON h.id = e.manager_id
),
direct_reports AS (
SELECT 
	manager_id,
	COUNT(*) AS direct_reports
FROM HIERARCHY
GROUP BY manager_id
)
SELECT 
	id,
	name,
	DEPTH,
	PATH,
	COALESCE(direct_reports, 0) AS direct_reports
FROM HIERARCHY h
LEFT JOIN direct_reports dr ON h.id = dr.manager_id

That was an unusual approach and I had to think for a while, but it wasn't too difficult so I figured it out - nice.

---

## Task 3: NULLIF + Window Functions — Transaction Anomaly Detection

**Scenario:**
The fraud team wants to flag transactions where the amount is unusually high relative to that user's typical behavior. Specifically, flag transactions where the amount is more than 2x the user's average — but handle users who have only one transaction (stddev = 0 or NULL) gracefully using NULLIF.

**Expected Output Columns:**
- `id` (integer)
- `user_id` (integer)
- `amount` (numeric)
- `user_avg` (numeric) — user's average transaction amount
- `ratio` (numeric) — `amount / NULLIF(user_avg, 0)`, rounded to 2 decimals
- `is_anomaly` (boolean) — true if ratio > 2.0

**Requirements:**
- Use `transactions` table, exclude NULL amounts and NULL user_ids
- Compute `user_avg` as a window AVG partitioned by user_id
- Use `NULLIF(user_avg, 0)` in the ratio to guard against division by zero
- Order by `ratio DESC NULLS LAST`

**Difficulty Rating:** 4/5

WITH transactions_w_avg AS (
SELECT 
	*,
	ROUND(AVG(amount) OVER (PARTITION BY t.user_id)::NUMERIC, 2) AS user_avg
FROM crappy_data_db.transactions t
WHERE amount IS NOT NULL -- a way simpler method to handle NULL amounts 
AND user_id IS NOT NULL-- NO NEED TO HANDLE NULL user_ids, HONESTLY, but here you are
),
users_transactions_cnt AS (
SELECT 
	user_id,
	COUNT(*) AS transactions_cnt
FROM transactions_w_avg
GROUP BY user_id
)
SELECT 
	tw.id,
	tw.user_id,
	tw.amount,
	tw.user_avg,
	ROUND(tw.amount / tw.user_avg, 2) AS ratio,
	CASE WHEN ROUND(tw.amount / tw.user_avg, 2) > 2.0 THEN TRUE ELSE FALSE END AS is_anomaly
FROM transactions_w_avg tw
JOIN users_transactions_cnt ut ON tw.user_id = ut.user_id AND ut.transactions_cnt > 0

Your instructions ARE CONTRADICTING EACH OTHER and unclear.
I've used a very simple approach to handle everything properly and knowing the data I know this is all correct.

---

## Submission Instructions

1. Task 1 — NULLIF safe division in order metrics (3/5)
2. Task 2 — Type B recursive CTE org chart with direct reports (4/5)
3. Task 3 — NULLIF + window functions for anomaly detection (4/5)
