# SQL Tasks — 2026-08-05 (Week 33, Day 3)

**Dataset:** orders / transactions / users  
**Focus:** Cohort retention · Self-join type affinity · Cumulative SUM frame specs

---

## Task 1 — Monthly Cohort Retention
**Difficulty: 4/5**

**Business question:**  
Group users into monthly cohorts by their `created_at` (registration month). For each cohort, track how many users placed at least one order in their signup month (month 0), and in each of the following 3 months (month 1, 2, 3).

Show retention as both raw count and percentage of the original cohort size.

**Expected output columns:**  
`cohort_month, cohort_size, retained_m0, retained_m1, retained_m2, retained_m3, pct_m0, pct_m1, pct_m2, pct_m3`

Where `cohort_month` is the truncated registration month (e.g. `2024-10-01`), and `retained_mN` = number of users from that cohort who placed an order in month N after signup.

Order by `cohort_month`.

**Difficulty: 4/5**

WITH users_cohorts AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS registration_month,
	DATE_TRUNC('Month', created_at) + INTERVAL '1 Month' AS m1,
	DATE_TRUNC('Month', created_at) + INTERVAL '2 Month' AS m2,
	DATE_TRUNC('Month', created_at) + INTERVAL '1 Month' AS m3
FROM crappy_data_db.users u
),
users_orders_retains AS (
SELECT 
	user_id,
	registration_month AS cohort_month,
	o.created_at,
	CASE WHEN DATE_TRUNC('Month', o.created_at) = registration_month THEN 1 ELSE 0 END AS retained_m0,
	CASE WHEN DATE_TRUNC('Month', o.created_at) = m1 THEN 1 ELSE 0 END AS retained_m1,
	CASE WHEN DATE_TRUNC('Month', o.created_at) = m2 THEN 1 ELSE 0 END AS retained_m2,
	CASE WHEN DATE_TRUNC('Month', o.created_at) = m3 THEN 1 ELSE 0 END AS retained_m3
FROM users_cohorts uc
LEFT JOIN crappy_data_db.orders o ON uc.id = o.user_id
),
users_retentions AS (
SELECT 
	user_id,
	cohort_month,
	COALESCE(SUM(retained_m0), 0) AS retained_m0,
	COALESCE(SUM(retained_m1), 0) AS retained_m1,
	COALESCE(SUM(retained_m2), 0) AS retained_m2,
	COALESCE(SUM(retained_m3), 0) AS retained_m3
FROM users_orders_retains
WHERE user_id IS NOT NULL
GROUP BY user_id, cohort_month
)
SELECT 
	cohort_month,
	COUNT(*) AS cohort_size,
	COUNT(*) FILTER (WHERE retained_m0 > 0) AS retained_m0,
	round(COUNT(*) FILTER (WHERE retained_m0 > 0) / COUNT(*)::NUMERIC * 100, 2) AS retained_m0_pct,
	COUNT(*) FILTER (WHERE retained_m1 > 0) AS retained_m1,
	round(COUNT(*) FILTER (WHERE retained_m1 > 0) / COUNT(*)::NUMERIC * 100, 2) AS retained_m1_pct,
	COUNT(*) FILTER (WHERE retained_m2 > 0) AS retained_m2,
	round(COUNT(*) FILTER (WHERE retained_m2 > 0) / COUNT(*)::NUMERIC * 100, 2) AS retained_m2_pct,
	COUNT(*) FILTER (WHERE retained_m3 > 0) AS retained_m3,
	round(COUNT(*) FILTER (WHERE retained_m3 > 0) / COUNT(*)::NUMERIC * 100, 2) AS retained_m3_pct
FROM users_retentions 
GROUP BY cohort_month
ORDER BY cohort_month

2024-01-01 00:00:00.000	2	2	100.00	2	100.00	0	0.00	2	100.00
2024-02-01 00:00:00.000	3	3	100.00	2	66.67	0	0.00	2	66.67
2024-03-01 00:00:00.000	5	3	60.00	5	100.00	0	0.00	5	100.00
2024-04-01 00:00:00.000	2	2	100.00	1	50.00	0	0.00	1	50.00
2024-05-01 00:00:00.000	5	5	100.00	3	60.00	0	0.00	3	60.00
2024-06-01 00:00:00.000	4	3	75.00	4	100.00	0	0.00	4	100.00
2024-07-01 00:00:00.000	4	2	50.00	3	75.00	0	0.00	3	75.00
2024-08-01 00:00:00.000	4	3	75.00	4	100.00	0	0.00	4	100.00
2024-09-01 00:00:00.000	5	4	80.00	5	100.00	0	0.00	5	100.00
2024-10-01 00:00:00.000	7	6	85.71	7	100.00	0	0.00	7	100.00
2024-11-01 00:00:00.000	1	0	0.00	1	100.00	0	0.00	1	100.00
2024-12-01 00:00:00.000	4	4	100.00	2	50.00	0	0.00	2	50.00
2025-01-01 00:00:00.000	8	7	87.50	6	75.00	0	0.00	6	75.00
2025-02-01 00:00:00.000	6	5	83.33	4	66.67	0	0.00	4	66.67
2025-03-01 00:00:00.000	11	7	63.64	11	100.00	0	0.00	11	100.00
2025-04-01 00:00:00.000	7	4	57.14	7	100.00	0	0.00	7	100.00
2025-05-01 00:00:00.000	11	9	81.82	4	36.36	0	0.00	4	36.36
2025-06-01 00:00:00.000	11	8	72.73	9	81.82	0	0.00	9	81.82
2025-07-01 00:00:00.000	12	11	91.67	10	83.33	0	0.00	10	83.33

Seems to be fine.

---

## Task 2 — Transaction Type Co-occurrence
**Difficulty: 4/5**

**Business question:**  
Find which pairs of transaction types most commonly co-occur within the same user — i.e. users who have used both types at least once.

For every combination of two distinct types (e.g. `deposit` + `withdrawal`, `payment` + `transfer`, etc.), count how many users have at least one transaction of each type.

Only include pairs where at least 10 users have both types. Order by `user_count DESC`.

**Expected output columns:**  
`type_a, type_b, user_count`

Note: each pair should appear only once — (`deposit`, `withdrawal`) and (`withdrawal`, `deposit`) are the same pair. Use `type_a < type_b` to enforce this.

**Difficulty: 4/5**

WITH users_with_most_co_occuring_transactions AS (
SELECT 
	t1.user_id,
	t1.TYPE AS type1,
	t2.TYPE AS type2,
	COUNT(*)
FROM crappy_data_db.transactions t1
JOIN crappy_data_db.transactions t2 ON t1.user_id = t2.user_id
WHERE t1.TYPE > t2.TYPE
GROUP BY t1.user_id, t1.TYPE, t2.TYPE
ORDER BY count DESC
)
SELECT 
	type1,
	type2,
	COUNT(user_id) AS user_count
FROM users_with_most_co_occuring_transactions
GROUP BY type1, type2
HAVING count(user_id) >= 10

Both objectives achieved


---

## Task 3 — Running Total: ROWS vs RANGE
**Difficulty: 5/5**

**Business question:**  
For each user, compute a **daily running total** of transaction amounts — the cumulative sum of all transactions up to and including each day.

There's a catch: some users have multiple transactions on the same day.

Write the query **twice** in the same file:

**Version A:** Using `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`  
**Version B:** Using `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`

Above each version, add a comment explaining:
- What the difference is between ROWS and RANGE when multiple rows share the same `created_at` day
- Which version gives the correct "end of day cumulative total" and why

**Expected output columns (both versions):**  
`user_id, tx_date, daily_amount, running_total`

Where `tx_date = created_at::date` and `daily_amount = SUM(amount)` per user per day (pre-aggregate before the window).

Order by `user_id`, `tx_date`.

**Difficulty: 5/5**


I get what you wanted to do here, but your approach is completely usesless for this task and an overengineered bs.

Look, here I can simply partition these transacitons by tx_date and easily get the required amounts. 

You'd need to design a task that's designed better for the RANGE/ROWS or give me a specified number of days/transactions you'd like to see cause this is where the difference lies.

WITH users_dates AS (
SELECT 
	*,
	t.created_at::date AS tx_date
FROM crappy_data_db.transactions t
)
SELECT 
	*,
	SUM(amount) OVER (PARTITION BY user_id, tx_date) AS daily_amount,
	SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS user_running_total
FROM users_dates

---

## Submission Instructions

Paste your queries below each task. For Task 3, include the comments — they're part of the answer.
