# Daily SQL Practice Tasks

**Generated:** 2026-01-16
**Week 6, Day 1 Focus:** Recursive CTEs, Hierarchical Data, Date Series Generation

---

## Task 1: Recursive CTE — Generate Date Series

**Scenario:**
Generate a complete date series for all months in 2025, then LEFT JOIN to show monthly order counts (including months with zero orders).

**Expected Output Columns:**
- `month_start` (date) — first day of each month (2025-01-01 through 2025-12-01)
- `order_count` (bigint) — number of orders in that month (0 if none)
- `total_revenue` (numeric) — sum of order amounts, rounded to 2 decimals (0 if none)

**Requirements:**
- Use a recursive CTE to generate all 12 months of 2025
- LEFT JOIN to orders table
- Use COALESCE to show 0 instead of NULL for months with no orders
- Order by month_start ASC

**Difficulty Rating:** 4/5

**Hint — Recursive CTE Structure:**
```sql
WITH RECURSIVE month_series AS (
    -- Anchor: starting point
    SELECT DATE '2025-01-01' AS month_start
    UNION ALL
    -- Recursive: add one month until December
    SELECT (month_start + INTERVAL '1 month')::DATE
    FROM month_series
    WHERE month_start < '2025-12-01'
)
SELECT ... FROM month_series LEFT JOIN ...
```

---

## Task 2: Self-Referential Hierarchy — User Referral Chain

**Scenario:**
Imagine users can refer other users. Given the transactions table, simulate a referral relationship: for each user, find another user who made their first transaction within 7 days AFTER them (potential "referred by" relationship). Build a chain showing who might have referred whom.

**Expected Output Columns:**
- `user_id` (integer) — the user
- `first_transaction_date` (timestamp) — when they first transacted
- `potential_referrer_id` (integer) — user who transacted 1-7 days before them (closest one)
- `referrer_first_transaction` (timestamp) — when the referrer first transacted
- `days_apart` (integer) — days between their first transactions

**Requirements:**
- Use `transactions` table
- Calculate each user's first transaction date
- Find the user whose first transaction was 1-7 days before (closest match)
- Use window functions or correlated subquery to find the closest referrer
- Only show users who have a potential referrer (exclude those with no match)
- Order by first_transaction_date ASC

**Difficulty Rating:** 5/5

---

## Task 3: Recursive CTE — Running Balance Simulation

**Scenario:**
For a specific user (user_id = 1), simulate their running account balance over time. Start with a balance of 1000, then add deposits/payments and subtract withdrawals/purchases/transfers.

**Expected Output Columns:**
- `transaction_id` (integer)
- `created_at` (timestamp)
- `type` (text) — transaction type
- `amount` (numeric)
- `balance_change` (numeric) — positive for deposits/payments, negative for withdrawals/purchases/transfers
- `running_balance` (numeric) — cumulative balance after this transaction

**Requirements:**
- Use `transactions` table filtered to user_id = 1
- Starting balance is 1000
- Deposits and payments ADD to balance
- Withdrawals, purchases, and transfers SUBTRACT from balance
- Use window function SUM() OVER (ORDER BY created_at) for running total
- Order by created_at ASC

**Difficulty Rating:** 4/5

---

## Submission Instructions

Welcome to Week 6! This week focuses on recursive CTEs and more complex analytical patterns.

**Key Concepts:**
- Recursive CTE = Anchor + UNION ALL + Recursive term with termination condition
- Self-joins for hierarchical/relationship discovery
- Running totals with conditional logic (CASE WHEN inside SUM)

Good luck!
