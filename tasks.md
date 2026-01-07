# Daily SQL Practice Tasks

**Generated:** 2026-01-01
**Week 4, Day 4 Focus:** Date Ranges, Session Analysis, Advanced NULL Handling

---

## Task 1: Users Active in Last 30 Days

**Scenario:**
The engagement team wants to identify recently active users. Find all users who have placed at least one order in the last 30 days from the current date.

**Expected Output Columns:**
- `user_id` (integer)
- `first_name` (varchar)
- `last_name` (varchar)
- `most_recent_order_date` (date) — date of their most recent order
- `days_since_last_order` (integer) — days between most recent order and current date
- `total_orders_last_30_days` (bigint) — count of orders in last 30 days

**Requirements:**
- Use `users` and `orders` tables
- Filter to orders within last 30 days from CURRENT_DATE
- Calculate most recent order date per user
- Count orders in the 30-day window
- Order by `days_since_last_order` ASC

**Difficulty Rating:** 3/5

---

## Task 2: Daily Session Patterns

**Scenario:**
The product team wants to understand session activity patterns. For each date, show total sessions, average sessions per active user, and identify dates with unusually high activity (>10 average sessions per user).

**Expected Output Columns:**
- `date` (date)
- `total_sessions` (numeric) — sum of all count_sessions for this date
- `active_users` (bigint) — count of users who had at least 1 session on this date
- `avg_sessions_per_user` (numeric) — average sessions per active user, rounded to 2 decimals
- `high_activity_day` (boolean) — TRUE if avg_sessions_per_user > 10

**Requirements:**
- Use `user_sessions_daily` table
- Calculate total sessions per date
- Count users who had sessions (count_sessions > 0)
- Calculate average sessions per active user
- Flag high activity days
- Order by `date` DESC

**Difficulty Rating:** 4/5

---

## Task 3: Transaction Amount Outliers

**Scenario:**
The fraud team wants to identify unusually large transactions. Find transactions where the amount is more than 3 times the average transaction amount for that user.

**Expected Output Columns:**
- `transaction_id` (integer)
- `user_id` (integer)
- `amount` (numeric)
- `user_avg_transaction` (numeric) — average transaction amount for this user, rounded to 2 decimals
- `times_above_avg` (numeric) — how many times above their average this transaction is, rounded to 2 decimals

**Requirements:**
- Use `transactions` table
- Calculate average transaction amount per user
- Find transactions > 3x their user's average
- Only include transactions with non-null amounts
- Order by `times_above_avg` DESC

**Difficulty Rating:** 4/5

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Date range filtering (INTERVAL, date arithmetic)
- Aggregation with conditional logic
- Window functions vs subqueries for averages
- Boolean flag creation

## Tips

- Date filtering: WHERE date >= CURRENT_DATE - INTERVAL '30 days'
- Or: WHERE date >= CURRENT_DATE - 30
- CASE WHEN for boolean flags
- Window functions can calculate user averages: AVG(amount) OVER (PARTITION BY user_id)
- Or use subquery/CTE for user averages

Good luck!
