# Daily SQL Practice Tasks

**Generated:** 2025-12-04
**Week 1 Focus:** Advanced JOINs, Window Functions, CTEs, Aggregations

---

## Task 1: Self-Join — Session Buddies

**Scenario:**
The product analytics team wants to identify "session buddies" — pairs of users who had the exact same number of sessions on at least 3 different dates. This helps understand synchronized user behavior patterns.

**Expected Output Columns:**
- `user_id_1` (integer) — first user ID (should be less than user_id_2)
- `user_id_2` (integer) — second user ID
- `matching_days` (bigint) — count of dates where they had identical session counts

**Requirements:**
- Use `user_sessions_daily` table
- Self-join to compare users
- Only include pairs where matching_days >= 3
- Ensure user_id_1 < user_id_2 to avoid duplicate pairs
- Order by `matching_days` DESC, then `user_id_1` ASC

**Difficulty Rating:** 3/5

---

## Task 2: First vs. Last Transaction Type Per User

**Scenario:**
The finance team wants to analyze user transaction journey patterns. Find all users whose first transaction type differs from their last transaction type, and show how many transactions they made in total.

**Expected Output Columns:**
- `user_id` (integer)
- `first_name` (varchar)
- `last_name` (varchar)
- `first_transaction_type` (text)
- `last_transaction_type` (text)
- `total_transactions` (bigint)

**Requirements:**
- Use `transactions` and `users` tables
- Apply window functions: FIRST_VALUE and LAST_VALUE
- Include only users with at least 2 transactions
- Filter for cases where first_transaction_type != last_transaction_type
- Order by `total_transactions` DESC

**Difficulty Rating:** 3/5

---

## Task 3: Rolling 7-Day Order Revenue

**Scenario:**
Calculate a 7-day rolling average of daily order revenue. For each date, compute the average order amount over that date plus the 6 preceding days. Include all dates from the dates table, even if there were no orders.

**Expected Output Columns:**
- `date` (date)
- `daily_revenue` (numeric) — total order amount for that specific date (NULL if no orders)
- `rolling_7day_avg` (numeric) — average daily revenue over 7-day window

**Requirements:**
- Use `dates` table as base (to include all dates)
- LEFT JOIN with aggregated orders data
- Window frame: ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
- Only return dates from year 2025
- Order by date ASC

**Difficulty Rating:** 4/5

---

## Task 4: Anti-Join — Users Who Never Bought Travel Products

**Scenario:**
The marketing team wants to launch a travel campaign targeting users who have made purchases but have NEVER bought anything from the "travel" category. Find these users.

**Expected Output Columns:**
- `user_id` (integer)
- `email` (varchar)
- `city` (varchar)
- `total_orders` (bigint) — count of all orders made by this user

**Requirements:**
- Use `users`, `orders`, `orders_products`, `products`, `product_categories` tables
- Use an anti-join pattern (NOT EXISTS or LEFT JOIN ... WHERE NULL)
- Exclude users who have purchased ANY product from category "travel"
- Include only users with at least 1 order
- Order by `total_orders` DESC

**Difficulty Rating:** 4/5

---

## Task 5: Ticket Resolution Time Ranking

**Scenario:**
For each priority level, rank support tickets by their resolution time (from creation to the timestamp of the last message). Use DENSE_RANK so that tickets with identical resolution times receive the same rank.

**Expected Output Columns:**
- `ticket_id` (bigint)
- `priority` (text)
- `resolution_time` (interval) — time from ticket creation to last message
- `rank_in_priority` (bigint) — dense rank within priority group (1 = fastest)

**Requirements:**
- Use `chat_tickets` and `chat_messages` tables
- Only include tickets with status = 'resolved' OR status = 'closed'
- Calculate resolution_time as: MAX(chat_messages.created_at) - chat_tickets.created_at
- Apply DENSE_RANK() OVER (PARTITION BY priority ORDER BY resolution_time ASC)
- Order final output by `priority` ASC, then `rank_in_priority` ASC

**Difficulty Rating:** 4/5

---

## Submission Instructions

When you're ready, submit your SQL solutions one at a time or all together. I'll provide detailed feedback including:
- Correctness of logic and syntax
- Efficiency and performance considerations
- Best practice suggestions
- Advanced alternatives (if applicable)

## Tips

- Build your queries incrementally — start simple, then add complexity
- Use CTEs to break down multi-step logic for readability
- Always check the schema.md file for exact column names and data types
- Consider NULL handling in your joins and aggregations
- Test window function frames carefully (ROWS vs RANGE)

Good luck!
