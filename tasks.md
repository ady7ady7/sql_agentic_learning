# Daily SQL Practice Tasks

**Generated:** 2026-01-11
**Week 5, Day 3 Focus:** RANK vs ROW_NUMBER, Conditional Aggregation, Subquery Performance

---

## Task 1: Dense Ranking with Ties

**Scenario:**
The sales team wants to rank products by revenue, but when multiple products have the same revenue, they should share the same rank (with no gaps in the sequence).

**Expected Output Columns:**
- `product_id` (integer)
- `product_name` (varchar)
- `total_revenue` (numeric) — sum of all revenue for this product
- `revenue_rank` (bigint) — dense ranking (1,2,2,3 not 1,2,2,4)

**Requirements:**
- Use `orders_products` and `products` tables
- Calculate total revenue per product (quantity * price)
- Use DENSE_RANK window function
- Order by revenue_rank ASC

**Difficulty Rating:** 2/5

---

## Task 2: Conditional Aggregation with Multiple Conditions

**Scenario:**
The finance team wants a single-row summary showing order counts and totals broken down by different criteria.

**Expected Output Columns:**
- `total_orders` (bigint) — count of all orders
- `high_value_orders` (bigint) — count where amount > 100
- `high_value_revenue` (numeric) — sum of amounts where amount > 100
- `low_value_orders` (bigint) — count where amount <= 100
- `low_value_revenue` (numeric) — sum of amounts where amount <= 100
- `avg_order_amount` (numeric) — average of all order amounts, rounded to 2 decimals

**Requirements:**
- Use `orders` table
- Return a single row with all aggregations
- Use CASE WHEN or FILTER clause for conditional counts/sums
- Round averages to 2 decimals

**Difficulty Rating:** 3/5

---

## Task 3: Correlated Subquery - Users Above Category Average

**Scenario:**
Find users whose total spending in each product category exceeds the average spending in that category across all users.

**Expected Output Columns:**
- `user_id` (integer)
- `category_name` (varchar)
- `user_category_spending` (numeric) — total spent by this user in this category
- `category_avg_spending` (numeric) — average spent per user in this category, rounded to 2 decimals
- `amount_above_avg` (numeric) — difference between user spending and average

**Requirements:**
- Use `orders`, `orders_products`, `products`, and `product_categories` tables
- Calculate user spending per category
- Calculate average spending per category (across all users)
- Only show users exceeding the category average
- Order by category_name, amount_above_avg DESC

**Difficulty Rating:** 4/5

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- DENSE_RANK vs RANK vs ROW_NUMBER usage
- Conditional aggregation patterns (CASE WHEN vs FILTER)
- Correlated subqueries vs window functions for averages
- Performance considerations

## Tips

- DENSE_RANK: No gaps in ranking sequence even with ties
- RANK: Gaps after ties (1,2,2,4)
- ROW_NUMBER: Always unique, even for identical values (1,2,3,4)
- Conditional aggregation: `SUM(CASE WHEN condition THEN amount ELSE 0 END)`
- Or: `COUNT(*) FILTER (WHERE condition)`
- Correlated subquery: Often replaced by window functions for better performance

Good luck!
