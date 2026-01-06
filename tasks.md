# Daily SQL Practice Tasks

**Generated:** 2025-12-31
**Week 4, Day 3 Focus:** Complex Aggregations, Date Ranges, Business Logic

---

## Task 1: Monthly Active Users (MAU) Trend

**Scenario:**
The growth team wants to track Monthly Active Users over time. For each month in the data, count how many distinct users placed at least one order during that month.

**Expected Output Columns:**
- `year` (integer) — year from order date
- `month` (integer) — month from order date (1-12)
- `monthly_active_users` (bigint) — distinct count of users who placed orders in this month
- `month_over_month_change` (bigint) — change in MAU compared to previous month (can be negative)

**Requirements:**
- Use `orders` table
- Count distinct users per month/year
- Calculate change from previous month
- Order by year ASC, month ASC

**Difficulty Rating:** 4/5

---

## Task 2: High-Value vs Low-Value Customer Segmentation

**Scenario:**
The marketing team wants to segment customers into high-value (total lifetime spending > $1000) and low-value (total lifetime spending <= $1000) groups. Show counts and average metrics for each segment.

**Expected Output Columns:**
- `segment` (text) — 'High-Value' or 'Low-Value'
- `customer_count` (bigint) — number of customers in this segment
- `avg_lifetime_value` (numeric) — average total spending per customer in segment, rounded to 2 decimals
- `avg_order_count` (numeric) — average number of orders per customer in segment, rounded to 2 decimals

**Requirements:**
- Use `orders` table
- Calculate total spending per user
- Segment users based on $1000 threshold
- Aggregate metrics per segment
- Order by segment DESC (High-Value first)

**Difficulty Rating:** 4/5

---

## Task 3: Products Ordered Together Analysis

**Scenario:**
The product team wants to understand which products are frequently purchased together in the same order. Find the top 10 product pairs that appear together most often.

**Expected Output Columns:**
- `product_1_name` (varchar) — name of first product (alphabetically earlier)
- `product_2_name` (varchar) — name of second product (alphabetically later)
- `times_ordered_together` (bigint) — number of orders containing both products

**Requirements:**
- Use `products`, `orders_products` tables
- Find product pairs that appear in the same order_id
- Avoid duplicates (if A-B exists, don't show B-A)
- Ensure product_1_name comes before product_2_name alphabetically
- Show top 10 pairs by frequency
- Order by `times_ordered_together` DESC

**Difficulty Rating:** 5/5

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- LAG for time-series comparisons
- Conditional logic for segmentation
- Self-joins for finding pairs
- Deduplication strategies

## Tips

- LAG() OVER (ORDER BY year, month) for previous month's value
- CASE WHEN for conditional segmentation
- Self-join on same table with order_id match for pairs
- Use comparison operators to avoid duplicate pairs (e.g., p1.name < p2.name)

Good luck!
