# Daily SQL Practice Tasks

**Generated:** 2026-01-15
**Week 5, Day 5 Focus:** Cumulative Distribution, Running Comparisons, Multi-Partition Analytics

---

## Task 1: NTILE vs PERCENT_RANK — Customer Spending Quartiles

**Scenario:**
The marketing team wants to segment customers into spending quartiles for targeted campaigns. They need both the quartile bucket (1-4) AND the exact percentile position within all customers.

**Expected Output Columns:**
- `user_id` (integer)
- `total_spending` (numeric) — sum of all order amounts for this user
- `spending_quartile` (integer) — NTILE(4) bucket (1 = lowest 25%, 4 = highest 25%)
- `spending_percentile` (numeric) — PERCENT_RANK (0 to 1), rounded to 3 decimals
- `quartile_label` (text) — 'Bronze' for Q1, 'Silver' for Q2, 'Gold' for Q3, 'Platinum' for Q4

**Requirements:**
- Use `orders` table
- Use both NTILE(4) and PERCENT_RANK() window functions
- Only include users who have at least one order
- Order by total_spending DESC

**Difficulty Rating:** 3/5

---

## Task 2: Running Difference — Month-over-Month Revenue Change

**Scenario:**
Finance needs a monthly revenue report showing not just totals, but also the change from the previous month and whether revenue is trending up or down.

**Expected Output Columns:**
- `month` (date) — first day of the month (use DATE_TRUNC)
- `monthly_revenue` (numeric) — sum of order amounts for that month, rounded to 2 decimals
- `previous_month_revenue` (numeric) — LAG of monthly_revenue
- `revenue_change` (numeric) — difference from previous month
- `change_percent` (numeric) — percentage change from previous month, rounded to 1 decimal
- `trend` (text) — 'UP' if positive change, 'DOWN' if negative, 'FLAT' if zero or NULL

**Requirements:**
- Use `orders` table
- Use DATE_TRUNC to group by month
- Use LAG() for previous month comparison
- Handle NULL for first month gracefully (show NULL, not error)
- Order by month ASC

**Difficulty Rating:** 4/5

---

## Task 3: Multi-Level Ranking — Product Performance Within Category AND Overall

**Scenario:**
The product team wants to identify products that rank in the top 3 within their category but fall outside the top 10 overall. These are "category champions" that might benefit from cross-category promotion.

**Expected Output Columns:**
- `product_id` (integer)
- `product_name` (varchar)
- `category_name` (varchar)
- `units_sold` (bigint) — total quantity sold
- `category_rank` (bigint) — RANK within category by units_sold
- `overall_rank` (bigint) — RANK across all products by units_sold
- `is_category_champion_needs_boost` (boolean) — TRUE if category_rank <= 3 AND overall_rank > 10

**Requirements:**
- Use `orders_products`, `products`, and `product_categories` tables
- Use RANK() with different partitions
- Only include products with at least one sale
- Order by category_name, category_rank ASC

**Difficulty Rating:** 4/5

---

## Submission Instructions

Submit your SQL solutions when ready. This is the final day of Week 5!

**Week 5 Concepts Covered:**
- DENSE_RANK vs RANK vs ROW_NUMBER
- PERCENT_RANK and NTILE for distribution analysis
- LAG/LEAD for row comparisons
- Custom window frames (ROWS BETWEEN)
- Gap-and-island detection
- Multi-partition window functions

After today's review, we'll do the Weekly Recap for Week 5.

Good luck on the final day!
