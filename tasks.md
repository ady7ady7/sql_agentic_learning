# Daily SQL Practice Tasks

**Generated:** 2025-12-15
**Week 2, Day 5 Focus:** UNION/INTERSECT/EXCEPT, Complex CASE Expressions, JSON Functions

---

## Task 1: Combined User Activity — UNION ALL

**Scenario:**
The analytics team wants a unified view of all user activity. Combine data from orders and transactions tables to show all user financial activity in chronological order.

**Expected Output Columns:**
- `user_id` (integer)
- `activity_date` (timestamp)
- `activity_type` (varchar) — 'order' or 'transaction'
- `amount` (numeric)
- `source_table` (varchar) — 'orders' or 'transactions'

**Requirements:**
- Use UNION ALL to combine orders and transactions
- Extract created_at as activity_date from both tables
- Label each row with its source table and activity type
- Exclude rows with NULL user_id or NULL amount
- Order by `user_id` ASC, `activity_date` ASC

**Difficulty Rating:** 3/5

---

## Task 2: Tiered Pricing with Complex CASE

**Scenario:**
Create a tiered discount system for products based on their price. Calculate the discount percentage and final price after discount using a complex CASE expression.

**Expected Output Columns:**
- `product_id` (integer)
- `product_name` (varchar)
- `original_price` (numeric)
- `discount_pct` (integer) — percentage: 0%, 5%, 10%, 15%, or 20%
- `final_price` (numeric) — price after discount, rounded to 2 decimals

**Requirements:**
- Use `products` table
- CASE expression for discount tiers:
  - price >= 100: 20% discount
  - price >= 75: 15% discount
  - price >= 50: 10% discount
  - price >= 25: 5% discount
  - price < 25: 0% discount
- Calculate final_price = original_price * (1 - discount_pct/100)
- Order by `original_price` DESC

**Difficulty Rating:** 3/5

---

## Task 3: Users Active in Both Orders and Sessions

**Scenario:**
Find users who are active in BOTH orders (placed at least 1 order) AND sessions (had at least 1 session with count_sessions > 0). Use INTERSECT or an alternative approach.

**Expected Output Columns:**
- `user_id` (integer)
- `order_count` (bigint) — count of orders
- `total_sessions` (numeric) — sum of count_sessions
- `first_order_date` (timestamp) — date of first order
- `last_session_date` (date) — date of most recent session

**Requirements:**
- Use `orders` and `user_sessions_daily` tables
- Find users present in both datasets (INTERSECT or INNER JOIN approach)
- Calculate metrics for matched users
- Order by `order_count` DESC

**Difficulty Rating:** 4/5

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Logic correctness and query structure
- UNION/INTERSECT usage
- Complex CASE expression implementation
- Set operation alternatives
- Alternative approaches

## Tips

- UNION ALL includes duplicates, UNION removes them
- INTERSECT finds common elements between two sets
- Complex CASE: `CASE WHEN condition1 THEN value1 WHEN condition2 THEN value2 ELSE default END`
- INTERSECT alternative: `WHERE user_id IN (SELECT user_id FROM other_table)`

Good luck!
