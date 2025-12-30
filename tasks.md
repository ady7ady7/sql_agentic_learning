# Daily SQL Practice Tasks

**Generated:** 2025-12-30
**Week 3, Day 5 Focus:** Advanced Date Arithmetic, Complex Filtering, Multi-Table Analysis

---

## Task 1: Order Streaks — Users with Consecutive Day Ordering

**Scenario:**
The marketing team wants to identify highly engaged users who made purchases on consecutive days (not just consecutive months, but actual back-to-back days). Find users who have ordered on at least 3 consecutive days at some point in their history.

**Expected Output Columns:**
- `user_id` (integer)
- `streak_start_date` (date) — the first day of the consecutive streak
- `streak_end_date` (date) — the last day of the consecutive streak
- `streak_length` (integer) — number of consecutive days in the streak
- `total_orders_in_streak` (bigint) — total number of orders during the streak period

**Requirements:**
- Use `orders` table
- Identify sequences where a user ordered on consecutive calendar days
- Only include streaks of 3 or more consecutive days
- If a user has multiple streaks, show all of them
- Order by `streak_length` DESC, `user_id` ASC

**Difficulty Rating:** 5/5

---

## Task 2: Product Category Performance Comparison

**Scenario:**
The product team wants to compare category performance. For each product category, show total revenue, average order value, and how it compares to the overall average across all categories.

**Expected Output Columns:**
- `category_name` (varchar)
- `total_revenue` (numeric) — total revenue for this category, rounded to 2 decimals
- `order_count` (bigint) — number of orders containing products from this category
- `avg_order_value` (numeric) — average revenue per order for this category, rounded to 2 decimals
- `overall_avg_order_value` (numeric) — average order value across all categories, rounded to 2 decimals
- `performance_vs_avg` (numeric) — difference between category avg and overall avg, rounded to 2 decimals

**Requirements:**
- Use `products`, `product_categories`, `orders_products` tables
- Calculate revenue as price × quantity
- Calculate average order value per category
- Compare each category's performance to the overall average
- Order by `total_revenue` DESC

**Difficulty Rating:** 4/5

---

## Task 3: Support Ticket Resolution Analysis by Priority

**Scenario:**
The support team wants to analyze ticket resolution patterns by priority level. Show average resolution time and ticket count for each priority level, but only for tickets that have been resolved (status = 'resolved' or 'closed').

**Expected Output Columns:**
- `priority` (text) — ticket priority level
- `resolved_ticket_count` (bigint) — number of resolved/closed tickets at this priority
- `avg_resolution_time_hours` (numeric) — average time from creation to last update for resolved tickets, in hours, rounded to 2 decimals
- `min_resolution_time_hours` (numeric) — fastest resolution time in hours, rounded to 2 decimals
- `max_resolution_time_hours` (numeric) — slowest resolution time in hours, rounded to 2 decimals

**Requirements:**
- Use `chat_tickets` table
- Only include tickets with status 'resolved' or 'closed'
- Calculate resolution time as difference between created_at and updated_at
- Convert time to hours
- Group by priority level
- Order by `avg_resolution_time_hours` ASC

**Difficulty Rating:** 3/5

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Date arithmetic and consecutive sequence detection
- Multi-table aggregations and comparisons
- Timestamp differences and time unit conversions
- Grouping and filtering strategies

## Tips

- Consecutive day detection often requires LAG or complex date arithmetic
- Multi-table revenue calculations need careful JOIN conditions
- Time differences: EXTRACT(EPOCH FROM interval) gives seconds, divide by 3600 for hours
- Consider using CTEs to break complex problems into manageable steps

Good luck!