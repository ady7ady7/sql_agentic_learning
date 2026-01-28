# Daily SQL Practice Tasks

**Generated:** 2026-01-24
**Week 7, Day 4 Focus:** Recursive CTEs + Window Functions Combined

---

## Today's Goal

Yesterday Task 1 was perfect. Today we'll reinforce that pattern and properly combine recursive CTEs with window functions.

---

## Task 1: Monthly Order Stats with Dynamic Date Range

**Scenario:**
Generate all months between the first and last order, then show order count, revenue, AND running cumulative revenue.

**Expected Output Columns:**
- `month` (date) — first day of each month
- `order_count` (bigint) — orders that month (0 if none)
- `monthly_revenue` (numeric) — sum of amounts that month (0 if none)
- `cumulative_revenue` (numeric) — running total of revenue up to this month

**Requirements:**
- Use recursive CTE to generate months from MIN to MAX order date
- LEFT JOIN to orders for aggregation
- Use window function SUM() OVER (ORDER BY month) for cumulative revenue
- Handle months with zero orders using COALESCE

**Difficulty Rating:** 4/5

**This combines:** Yesterday's Task 1 pattern + proper window function usage.

---

## Task 2: Depreciation Schedule

**Scenario:**
An asset worth $10,000 depreciates by 15% each year. Generate a 7-year depreciation schedule showing the value at the start of each year and the depreciation amount.

**Expected Output Columns:**
- `year` (integer) — 1 through 7
- `start_value` (numeric) — value at start of year, rounded to 2 decimals
- `depreciation` (numeric) — amount lost that year (15% of start_value)
- `end_value` (numeric) — value at end of year (start_value - depreciation)

**Requirements:**
- Use recursive CTE
- Year 1 starts with $10,000
- Each subsequent year's start_value = previous year's end_value
- Calculate depreciation as start_value * 0.15

**Difficulty Rating:** 3/5

**This is like compound interest but subtracting instead of adding.**

---

## Task 3: User Registration by Week with Running Total

**Scenario:**
Show weekly user registration counts with a running total of all users registered up to that week.

**Expected Output Columns:**
- `week_start` (date) — Monday of each week
- `new_users` (bigint) — users registered that week
- `total_users` (bigint) — cumulative users up to and including that week

**Requirements:**
- Get weekly registration counts from users table (GROUP BY week)
- Use DATE_TRUNC('week', created_at) to get week start
- Use window function SUM() OVER (ORDER BY week_start) for running total
- Order by week_start

**Difficulty Rating:** 3/5

**Note:** No recursive CTE needed — this practices proper window function running totals (fixing yesterday's Task 3 pattern).

---

## Submission Instructions

Today's tasks:
1. Task 1 — Recursive months + LEFT JOIN + window function (4/5)
2. Task 2 — Depreciation with carry-forward (3/5)
3. Task 3 — Window function running total done correctly (3/5)

Good luck!
