# Daily SQL Practice Tasks

**Generated:** 2025-12-06
**Week 1, Day 3 Focus:** Recursive CTEs, RANK vs ROW_NUMBER, Advanced Aggregations

---

## Task 1: Category Hierarchy with Recursive CTE

**Scenario:**
The product team wants to analyze the total revenue for each product category. However, some categories might be related hierarchically (though not in current schema). For this exercise, find all distinct product categories and rank them by total revenue, showing cumulative revenue as you go down the ranking.

**Expected Output Columns:**
- `category_id` (integer)
- `category_name` (varchar)
- `total_revenue` (numeric) — total revenue from this category
- `revenue_rank` (bigint) — rank by revenue (1 = highest)
- `cumulative_revenue` (numeric) — running total of revenue from rank 1 to current

**Requirements:**
- Use `product_categories`, `products`, `orders_products`, `orders` tables
- Apply RANK() for revenue ranking
- Use window function with frame for cumulative sum
- Only include orders from 2025
- Order by `revenue_rank` ASC

**Difficulty Rating:** 4/5

---

## Task 2: User Cohort Analysis — First Order Month

**Scenario:**
The growth team wants to perform cohort analysis. For each user, determine which month they made their first order (their "cohort"), then calculate how many orders that cohort has made in total and what their average order value is.

**Expected Output Columns:**
- `cohort_month` (date) — first day of the month when users made their first order
- `user_count` (bigint) — number of users in this cohort
- `total_orders` (bigint) — total orders made by this cohort (all time)
- `avg_order_value` (numeric) — average order amount across all cohort orders

**Requirements:**
- Use `orders` table
- Determine each user's first order month using window functions
- Group by cohort month to aggregate metrics
- Only include cohorts from 2025
- Order by `cohort_month` ASC

**Difficulty Rating:** 4/5

---

## Task 3: ROW_NUMBER vs RANK — Handling Ties in Top Products

**Scenario:**
The inventory team needs two different views of top-selling products: one that breaks ties arbitrarily (ROW_NUMBER) and one that gives tied products the same rank (RANK). For each product, calculate total quantity sold and show both ranking methods.

**Expected Output Columns:**
- `product_id` (integer)
- `product_name` (varchar)
- `total_quantity_sold` (numeric) — sum of quantity from orders_products
- `row_number_rank` (bigint) — unique sequential number (breaks ties by product_id)
- `rank_with_ties` (bigint) — rank that gives same value for ties

**Requirements:**
- Use `products` and `orders_products` tables
- Calculate total quantity sold per product
- Apply both ROW_NUMBER() and RANK() ordered by total_quantity_sold DESC
- Include all products even if they haven't been ordered (show 0 quantity)
- Order by `total_quantity_sold` DESC

**Difficulty Rating:** 3/5

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Logic correctness and query structure
- Window function usage
- Efficiency considerations
- Alternative approaches

## Tips

- For cumulative sums, use: `SUM(column) OVER (ORDER BY ... ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)`
- DATE_TRUNC('month', timestamp) gives you the first day of the month
- ROW_NUMBER() assigns unique numbers; RANK() allows ties
- LEFT JOIN to include products with zero sales

Good luck!
