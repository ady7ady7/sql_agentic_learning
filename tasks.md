# Daily SQL Practice Tasks

**Generated:** 2025-12-22
**Week 3, Day 2 Focus:** Advanced Ranking, Running Totals with Frames, Multiple Window Functions

---

## Task 1: Top 3 Products per Category by Revenue

**Scenario:**
The product team wants to identify the top 3 best-selling products in each category based on total revenue. Show products ranked within their category, but only include the top 3 from each category.

**Expected Output Columns:**
- `category_name` (varchar) — category name from product_categories
- `product_name` (varchar) — product name
- `total_revenue` (numeric) — total revenue for this product (price × quantity across all orders), rounded to 2 decimals
- `category_rank` (bigint) — rank within category (1 = highest revenue in category)

**Requirements:**
- Use `products`, `product_categories`, `orders_products` tables
- Calculate revenue as price × quantity, then SUM for each product
- Use DENSE_RANK() OVER (PARTITION BY category_id ORDER BY total_revenue DESC)
- Filter to only include ranks 1, 2, 3
- Order by `category_name` ASC, `category_rank` ASC

**Difficulty Rating:** 4/5

---

## Task 2: Running Total of Daily Revenue with Month Reset

**Scenario:**
Finance wants to see a running total of daily revenue that resets at the start of each month. For each day, show the cumulative revenue within that month up to and including that day.

**Expected Output Columns:**
- `order_date` (date) — the date orders were created
- `daily_revenue` (numeric) — total revenue for that specific day, rounded to 2 decimals
- `running_monthly_total` (numeric) — cumulative revenue within the month up to this day, rounded to 2 decimals
- `year` (integer) — year from order_date
- `month` (integer) — month from order_date

**Requirements:**
- Use `orders` table
- Extract date from created_at timestamp
- Calculate daily revenue: SUM of amount per date
- Use window function with PARTITION BY year, month and ORDER BY date
- Use appropriate frame: ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
- Order by `order_date` ASC

**Difficulty Rating:** 4/5

---

## Task 3: User Quartiles by Transaction Amount

**Scenario:**
The analytics team wants to segment users into quartiles (4 equal groups) based on their total transaction amount. Assign each user to a quartile (1 = lowest 25%, 4 = highest 25%) and show summary statistics.

**Expected Output Columns:**
- `user_id` (integer)
- `total_transaction_amount` (numeric) — sum of all transaction amounts for this user, rounded to 2 decimals
- `transaction_count` (bigint) — number of transactions for this user
- `user_quartile` (integer) — quartile assignment (1, 2, 3, or 4)

**Requirements:**
- Use `transactions` table
- Calculate total_transaction_amount: SUM(amount) per user
- Calculate transaction_count: COUNT(*) per user
- Use NTILE(4) OVER (ORDER BY total_transaction_amount DESC) for quartile assignment
- Only include users who have at least 1 transaction with non-null amount
- Order by `user_quartile` ASC, `total_transaction_amount` DESC

**Difficulty Rating:** 3/5

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- DENSE_RANK vs RANK vs ROW_NUMBER usage
- Window frame specifications (ROWS vs RANGE)
- NTILE for bucketing/segmentation
- PARTITION BY with multiple columns
- Filtering ranked results (WHERE vs HAVING vs subquery)

## Tips

- DENSE_RANK: No gaps in ranking when ties exist (1, 2, 2, 3)
- RANK: Gaps after ties (1, 2, 2, 4)
- ROW_NUMBER: Always unique (1, 2, 3, 4)
- Frame clause: `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` for running totals
- NTILE(n): Divides rows into n roughly equal buckets
- Filtering ranked results: Use a subquery/CTE, then filter in outer query WHERE clause

Good luck!
