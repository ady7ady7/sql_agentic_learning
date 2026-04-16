# Daily SQL Practice Tasks

**Generated:** 2026-04-17
**Week 18, Day 5 Focus:** Query optimization concepts + session gaps-and-islands + YoY window

---

## Task 1: Query Optimization — Rewrite a Slow Query

**Scenario:**
The following query is logically correct but poorly written — it uses a correlated subquery in the SELECT clause that re-executes for every row, and a WHERE subquery that also re-scans the table. Your job is to rewrite it to be more efficient using CTEs or JOINs, without changing the result.

**Original slow query:**
```sql
SELECT
    u.id AS user_id,
    u.country,
    (SELECT SUM(o.amount) FROM orders o WHERE o.user_id = u.id) AS total_spent,
    (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) AS order_count
FROM users u
WHERE (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) >= 5
ORDER BY total_spent DESC;
```

**Your task:**
1. Rewrite this query using a CTE + JOIN to eliminate the correlated subqueries
2. Add a brief SQL comment (1-2 lines) explaining why the original is slow and why your version is faster

**Expected Output Columns:** same as original — `user_id`, `country`, `total_spent`, `order_count`

**Difficulty Rating:** 3/5

---

## Task 2: Time-Based Gaps — User Session Inactivity Periods (5/5)

**Scenario:**
The product team wants to find users who had a significant gap in their daily session activity — specifically, find all pairs of consecutive active days (days with count_sessions > 0) per user where the gap between them is **more than 14 days**, and report how long that inactivity gap was.

**Expected Output Columns:**
- `user_id` (integer)
- `last_active_date` (date) — the last active day before the gap
- `next_active_date` (date) — the first active day after the gap
- `gap_days` (integer) — number of days between them

**Requirements:**
- Use `user_sessions_daily`, only include days where `count_sessions > 0`
- Use LEAD to get the next active date per user
- Only return rows where gap_days > 14
- Order by `gap_days DESC`, `user_id ASC`

**Difficulty Rating:** 5/5

---

## Task 3: YoY Comparison with Window — Monthly Revenue Trend

**Scenario:**
The finance team wants a month-by-month order revenue table that also shows the same month's revenue from the prior year — so they can compare directly in one row.

**Expected Output Columns:**
- `month` (date) — DATE_TRUNC to month
- `revenue` (numeric) — total order amount for this month, rounded to 2 decimals
- `prev_year_revenue` (numeric) — revenue for the same calendar month one year prior, NULL if no data, rounded to 2 decimals
- `yoy_diff` (numeric) — revenue minus prev_year_revenue, NULL if no prior year data

**Requirements:**
- Use `orders`, exclude NULL amounts
- Use LAG(revenue, 12) OVER (ORDER BY month ASC) to get same-month prior year value
- Only include months with at least 1 order
- Order by `month ASC`

**Difficulty Rating:** 4/5

---

## Submission Instructions

1. Task 1 — Rewrite correlated subquery as CTE + JOIN with explanation comment (3/5)
2. Task 2 — LEAD-based inactivity gap detection > 14 days (5/5)
3. Task 3 — Monthly revenue with LAG(12) for prior year comparison (4/5)
