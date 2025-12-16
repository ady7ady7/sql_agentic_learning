# Daily SQL Practice Tasks

**Generated:** 2025-12-13
**Week 2, Day 3 Focus:** Complex Window Frames, HAVING with Aggregations, Subqueries in SELECT

---

## Task 1: Product Sales Trend — Moving Average

**Scenario:**
The sales team wants to smooth out daily fluctuations in product sales. For each product and date, calculate a 7-day moving average of quantity sold.

**Expected Output Columns:**
- `product_id` (integer)
- `date` (date)
- `daily_quantity` (numeric) — total quantity sold on this date
- `moving_avg_7day` (numeric) — average quantity over current day + 6 preceding days, rounded to 2 decimals

**Requirements:**
- Use `orders`, `orders_products` tables
- Join with `dates` table to ensure all dates included (even if no sales)
- Use window function with ROWS frame (ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
- Calculate daily totals per product first, then apply moving average
- Only include products that have been sold at least once
- Order by `product_id` ASC, `date` ASC

**Difficulty Rating:** 4/5

---

## Task 2: High-Value Customers — HAVING Filter

**Scenario:**
The marketing team wants to identify "high-value" customers who have spent more than $5000 total AND placed more than 10 orders. Show their spending summary.

**Expected Output Columns:**
- `user_id` (integer)
- `total_spent` (numeric) — sum of all order amounts
- `order_count` (bigint) — count of orders
- `avg_order_value` (numeric) — average order amount, rounded to 2 decimals
- `first_order_date` (timestamp) — date of first order
- `last_order_date` (timestamp) — date of most recent order

**Requirements:**
- Use `orders` table
- Use HAVING clause to filter for total_spent > 5000 AND order_count > 10
- Calculate all metrics in one query with GROUP BY
- Order by `total_spent` DESC

**Difficulty Rating:** 3/5

---

## Task 3: Category Market Share — Subquery in SELECT

**Scenario:**
The product team wants to see each category's revenue as a percentage of total company revenue. Use a subquery in the SELECT clause to calculate the grand total.

**Expected Output Columns:**
- `category_id` (integer)
- `category_name` (varchar)
- `category_revenue` (numeric) — sum of (quantity * price) for this category
- `total_company_revenue` (numeric) — sum of all revenue (calculated via subquery)
- `market_share_pct` (numeric) — (category_revenue / total) * 100, rounded to 2 decimals

**Requirements:**
- Use `product_categories`, `products`, `orders_products` tables
- Use scalar subquery in SELECT to get total company revenue
- Calculate market share percentage
- Only include categories with at least one sale
- Order by `market_share_pct` DESC

**Difficulty Rating:** 4/5

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Logic correctness and query structure
- Window frame usage (ROWS vs RANGE)
- HAVING clause effectiveness
- Subquery patterns
- Alternative approaches

## Tips

- Moving average frame: `AVG(quantity) OVER (PARTITION BY product_id ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)`
- HAVING filters aggregated results: `GROUP BY user_id HAVING SUM(amount) > 5000 AND COUNT(*) > 10`
- Scalar subquery: `(SELECT SUM(revenue) FROM ...) AS total_revenue`
- For dates with no sales, use LEFT JOIN and COALESCE to show 0

Good luck!
