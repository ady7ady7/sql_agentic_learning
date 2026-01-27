# Daily SQL Practice Tasks

**Generated:** 2026-01-23
**Week 7, Day 3 Focus:** Recursive CTEs — Real-World Complexity

---

## Stepping Up

You've mastered the basic patterns. Now we combine recursive CTEs with real table data, window functions, and complex business logic.

---

## Task 1: Running Balance per User with Carry-Forward

**Scenario:**
For each user who has transactions, calculate their running balance over time. Start each user at $0, then apply each transaction: deposits/payments add, withdrawals/purchases/transfers subtract.

**Expected Output Columns:**
- `user_id` (integer)
- `transaction_id` (integer)
- `created_at` (timestamp)
- `type` (text)
- `amount` (numeric)
- `running_balance` (numeric) — cumulative balance after this transaction

**Requirements:**
- Use a recursive CTE to process transactions in chronological order per user
- The anchor should be each user's first transaction
- The recursive term should find the next transaction for that user and update the balance
- Handle transaction types: deposit/payment = +amount, withdrawal/purchase/transfer = -amount
- Order by user_id, created_at

**Difficulty Rating:** 5/5

**Hint:** This is tricky because you need to:
1. Identify each user's first transaction (anchor)
2. For each iteration, find the NEXT transaction for that user (using ROW_NUMBER or similar)
3. Carry forward the balance

Alternative simpler approach: Use window function SUM() OVER (PARTITION BY user_id ORDER BY created_at) with CASE WHEN for +/- logic. If you use this approach, explain why it's better than recursive CTE here.

---

## Task 2: Consecutive Days with Orders (Streak Analysis)

**Scenario:**
Find the longest streak of consecutive days where at least one order was placed. A streak breaks when there's a day with no orders.

**Expected Output Columns:**
- `streak_start` (date) — first day of the streak
- `streak_end` (date) — last day of the streak
- `streak_length` (integer) — number of consecutive days

**Requirements:**
- First, identify which dates have orders
- Use recursive CTE or gap-and-island technique to find consecutive date ranges
- Find the longest streak(s)
- If multiple streaks have the same max length, show all of them

**Difficulty Rating:** 5/5

**This is a classic gap-and-island problem.** Approaches:
1. Recursive CTE: Start from each order date, recursively check if next day has orders
2. Gap-and-island with ROW_NUMBER: Subtract row_number from date to create groups

---

## Task 3: Monthly Revenue with Running Total and Month-over-Month Change

**Scenario:**
Generate all months from the earliest order to the latest, showing monthly revenue, running total, and percentage change from previous month.

**Expected Output Columns:**
- `month` (date) — first day of month
- `monthly_revenue` (numeric) — sum of order amounts that month (0 if none)
- `running_total` (numeric) — cumulative revenue up to this month
- `mom_change_pct` (numeric) — percentage change from previous month, rounded to 1 decimal (NULL for first month)

**Requirements:**
- Use recursive CTE to generate month series (from MIN to MAX order date)
- LEFT JOIN to orders for monthly aggregation
- Use window functions for running_total and LAG for previous month
- Calculate percentage: ((current - previous) / previous) * 100
- Handle division by zero (if previous month was 0)

**Difficulty Rating:** 4/5

**This combines:** Recursive date generation + LEFT JOIN + window functions (SUM OVER, LAG) + percentage calculation.

---

## Submission Instructions

Today's tasks combine recursive CTEs with:
1. Task 1 — Per-user transaction processing with balance carry-forward
2. Task 2 — Gap-and-island streak detection
3. Task 3 — Multi-technique combination (recursive + JOIN + window functions)

These are significantly harder. Take your time.

Good luck!
