# Daily SQL Practice Tasks

**Generated:** 2025-12-12
**Week 2, Day 2 Focus:** Advanced Date Functions, CASE Expressions, Multiple Aggregations

---

## Task 1: Monthly Active Users (MAU) Calculation

**Scenario:**
The analytics team needs to calculate Monthly Active Users (MAU) for each month. A user is "active" in a month if they have at least one session with count_sessions > 0 during that month.

**Expected Output Columns:**
- `year` (integer) — extracted from date
- `month` (integer) — extracted from date
- `active_users` (bigint) — count of distinct users with sessions > 0 in that month
- `total_sessions` (numeric) — sum of all count_sessions for that month

**Requirements:**
- Use `user_sessions_daily` table
- Extract year and month from date column
- Count distinct users with count_sessions > 0 per month
- Calculate total sessions per month
- Order by `year` ASC, `month` ASC

**Difficulty Rating:** 3/5

---

## Task 2: Transaction Type Distribution with CASE

**Scenario:**
The finance team wants to analyze transaction types and categorize them into "Income" (deposit, transfer incoming) vs "Expense" (withdrawal, payment, purchase). For each user, show counts and totals for each category.

**Expected Output Columns:**
- `user_id` (integer)
- `income_count` (bigint) — count of deposit/transfer transactions
- `income_total` (numeric) — sum of amounts for deposit/transfer
- `expense_count` (bigint) — count of withdrawal/payment/purchase transactions
- `expense_total` (numeric) — sum of amounts for withdrawal/payment/purchase
- `net_balance` (numeric) — income_total - expense_total

**Requirements:**
- Use `transactions` table
- Use CASE expressions to categorize transaction types
- Consider: "deposit" and "transfer" as income, others as expense
- Exclude transactions with NULL user_id or NULL amount
- Only include users with at least one transaction
- Order by `net_balance` DESC

**Difficulty Rating:** 4/5

---

## Task 3: Support Ticket Response Time Analysis

**Scenario:**
The support team wants to analyze response times. For each ticket, calculate the time difference between ticket creation and the first message, then find the average response time per priority level.

**Expected Output Columns:**
- `priority` (varchar) — ticket priority
- `ticket_count` (bigint) — number of tickets with this priority
- `avg_response_minutes` (numeric) — average minutes between ticket creation and first message, rounded to 2 decimals
- `median_response_minutes` (numeric) — median response time in minutes

**Requirements:**
- Use `chat_tickets` and `chat_messages` tables
- Calculate time difference in minutes using EXTRACT(EPOCH FROM (timestamp1 - timestamp2))/60
- Use window function FIRST_VALUE to get first message per ticket
- Use PERCENTILE_CONT(0.5) for median
- Group by priority
- Order by `avg_response_minutes` ASC

**Difficulty Rating:** 4/5

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Logic correctness and query structure
- Date extraction and manipulation
- CASE expression usage
- Aggregation patterns
- Alternative approaches

## Tips

- EXTRACT(YEAR FROM date_column) and EXTRACT(MONTH FROM date_column) extract date parts
- CASE expressions in aggregations: SUM(CASE WHEN type = 'deposit' THEN amount ELSE 0 END)
- Time differences: EXTRACT(EPOCH FROM (timestamp1 - timestamp2))/60 gives minutes
- PERCENTILE_CONT requires WITHIN GROUP(ORDER BY column)

Good luck!
