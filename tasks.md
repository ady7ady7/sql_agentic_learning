# Daily SQL Practice Tasks

**Generated:** 2025-12-05
**Week 1, Day 2 Focus:** Advanced Aggregations, LAG/LEAD, Cross Joins, CTEs

---

## Task 1: Customer Churn Analysis — LAG Function

**Scenario:**
The business analytics team needs to identify users who are at risk of churning. Calculate the gap (in days) between consecutive orders for each user, then find users whose most recent order gap is more than 2x their average gap between orders.

**Expected Output Columns:**
- `user_id` (integer)
- `first_name` (varchar)
- `last_name` (varchar)
- `avg_days_between_orders` (numeric) — average gap between all consecutive orders
- `most_recent_gap_days` (numeric) — days since their last order
- `churn_risk_ratio` (numeric) — most_recent_gap / avg_gap

**Requirements:**
- Use `orders` and `users` tables
- Apply LAG() window function to calculate gaps between consecutive orders
- Only include users with at least 3 orders (to calculate meaningful averages)
- Filter for users where `churn_risk_ratio > 2.0`
- Order by `churn_risk_ratio` DESC

**Difficulty Rating:** 4/5

---

## Task 2: Product Category Performance Matrix — Cross Join

**Scenario:**
The product team wants a complete matrix showing revenue for each product category in each month of 2025, including months where a category had zero sales. This helps visualize seasonal trends.

**Expected Output Columns:**
- `category_name` (varchar)
- `month` (integer) — 1-12
- `total_revenue` (numeric) — total revenue for that category in that month (0 if no sales)
- `order_count` (bigint) — number of orders containing that category (0 if none)

**Requirements:**
- Use `product_categories`, `products`, `orders_products`, `orders` tables
- Use CROSS JOIN to create all category × month combinations
- LEFT JOIN actual sales data
- Use COALESCE to show 0 for months with no sales
- Only include data from year 2025
- Order by `category_name` ASC, `month` ASC

**Difficulty Rating:** 4/5

---

## Task 3: Transaction Patterns — LEAD and LAG Together

**Scenario:**
The fraud detection team wants to identify unusual transaction patterns. For each transaction, show the previous and next transaction amounts for the same user, along with the time gaps.

**Expected Output Columns:**
- `user_id` (integer)
- `transaction_id` (integer) — current transaction id
- `amount` (numeric) — current transaction amount
- `prev_amount` (numeric) — previous transaction amount (NULL if first)
- `next_amount` (numeric) — next transaction amount (NULL if last)
- `time_since_prev` (interval) — time gap from previous transaction
- `time_until_next` (interval) — time gap to next transaction

**Requirements:**
- Use `transactions` table
- Apply both LAG() and LEAD() window functions partitioned by user_id
- Order transactions by created_at within each user
- Only include transactions from users with at least 3 transactions
- Order final output by `user_id` ASC, transaction `created_at` ASC

**Difficulty Rating:** 3/5

---

## Task 4: Support Ticket Escalation Analysis

**Scenario:**
The customer support manager wants to identify tickets that required escalation (multiple status changes). Find tickets that changed status at least 3 times and calculate the average time between status changes.

**Expected Output Columns:**
- `ticket_id` (bigint)
- `user_id` (bigint)
- `priority` (text)
- `status_change_count` (bigint) — number of status changes
- `avg_time_between_changes` (interval) — average time between status changes
- `final_status` (text) — most recent status

**Requirements:**
- Use `chat_tickets` and `chat_messages` tables
- Filter for messages where `message_type = 'statuschange'`
- Count status changes per ticket
- Calculate time gaps between consecutive status changes using LAG()
- Only include tickets with `status_change_count >= 3`
- Order by `status_change_count` DESC, then `ticket_id` ASC

**Difficulty Rating:** 5/5

---

## Task 5: Monthly Revenue Growth Percentage

**Scenario:**
The finance team needs a monthly revenue report showing month-over-month growth percentages. Calculate total order revenue for each month in 2025 and compare it to the previous month.

**Expected Output Columns:**
- `year` (integer)
- `month` (integer)
- `monthly_revenue` (numeric) — total revenue for the month
- `prev_month_revenue` (numeric) — previous month's revenue (NULL for January)
- `growth_percentage` (numeric) — ((current - prev) / prev) * 100

**Requirements:**
- Use `orders` table
- Extract year and month from created_at
- Aggregate revenue by month
- Use LAG() to get previous month's revenue
- Calculate growth percentage (handle division by zero with NULLIF)
- Only include months from 2025
- Order by `year` ASC, `month` ASC

**Difficulty Rating:** 3/5

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Logic correctness and syntax
- Efficiency and performance
- Best practices
- Alternative approaches

## Tips

- Remember to use `FIRST_VALUE(...ORDER BY ... DESC)` pattern for getting last values
- Pay attention to required output columns — include ALL specified columns
- Use CTEs to break down complex multi-step logic
- Consider NULL handling in calculations (COALESCE, NULLIF)
- Test LAG/LEAD with appropriate PARTITION BY and ORDER BY clauses

Good luck!
