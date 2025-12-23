# Daily SQL Practice Tasks

**Generated:** 2025-12-22
**Week 3, Day 3 Focus:** Complex Window Functions, Self-Joins, Advanced Aggregations

---

## Task 1: Revenue Percentile Analysis

**Scenario:**
The finance team wants to understand the distribution of order values. For each order, calculate what percentile it falls into compared to all other orders (e.g., an order at the 75th percentile is larger than 75% of all orders).

**Expected Output Columns:**
- `order_id` (integer)
- `user_id` (integer)
- `amount` (double precision) — order amount
- `revenue_percentile` (double precision) — percentile rank (0.0 to 1.0, rounded to 4 decimals)

**Requirements:**
- Use `orders` table
- Calculate the percentile rank for each order based on its amount
- Higher amounts should have higher percentile values
- Only include orders with non-null amounts
- Order by `revenue_percentile` DESC, `order_id` ASC

**Difficulty Rating:** 3/5

---

## Task 2: Customer Retention — Users with Orders in Consecutive Months

**Scenario:**
The product team wants to identify users who made purchases in consecutive months (e.g., ordered in January and February, or March and April). Find all users who have made orders in at least one pair of consecutive months.

**Expected Output Columns:**
- `user_id` (integer)
- `first_month` (integer) — the earlier month of the consecutive pair (1-12)
- `second_month` (integer) — the later month of the consecutive pair (1-12)
- `year` (integer) — year when this happened
- `orders_in_first_month` (bigint) — count of orders in the first month
- `orders_in_second_month` (bigint) — count of orders in the second month

**Requirements:**
- Use `orders` table
- Find users with orders in consecutive calendar months within the same year
- A user may have multiple consecutive month pairs (e.g., Jan-Feb AND Feb-Mar)
- Order by `user_id` ASC, `year` ASC, `first_month` ASC

**Difficulty Rating:** 5/5

---

## Task 3: Support Ticket Response Time Analysis

**Scenario:**
The support team wants to analyze how quickly they respond to tickets. Calculate the time between ticket creation and the first message sent by support (where `author_id` IS NOT NULL, indicating a support agent message, not a user message).

**Expected Output Columns:**
- `ticket_id` (bigint)
- `ticket_created_at` (timestamp with time zone) — when ticket was created
- `first_response_at` (timestamp with time zone) — timestamp of first support message
- `response_time_minutes` (numeric) — time difference in minutes, rounded to 2 decimals

**Requirements:**
- Use `chat_tickets` and `chat_messages` tables
- Find the first message where `author_id IS NOT NULL` for each ticket
- Calculate time difference in minutes between ticket creation and first response
- Only include tickets that have received at least one support response
- Order by `response_time_minutes` DESC

**Difficulty Rating:** 4/5

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Percentile and ranking functions (PERCENT_RANK, CUME_DIST, etc.)
- Self-joins and date arithmetic for consecutive period detection
- Window functions with filtering (FIRST_VALUE, MIN, etc.)
- Time difference calculations (EXTRACT EPOCH, date subtraction)

## Tips

- PERCENT_RANK() returns values from 0 to 1 showing relative position
- CUME_DIST() returns cumulative distribution (percentage of values <= current)
- For consecutive months, consider date arithmetic and comparisons
- FIRST_VALUE with proper ordering can find earliest/latest values
- EXTRACT(EPOCH FROM interval) converts intervals to seconds

Good luck!
