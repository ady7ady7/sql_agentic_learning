# Daily SQL Practice Tasks

**Generated:** 2026-01-14
**Week 5, Day 4 Focus:** Advanced Window Frames, Percentile Rankings, Gap Analysis

---

## Task 1: Rolling Window with Custom Frame

**Scenario:**
The analytics team wants to compare each user's daily session count to their own 3-day moving average (current day plus the 2 previous days). Flag days where the user's session count is more than 50% above their moving average as "spike" days.

**Expected Output Columns:**
- `user_id` (integer)
- `date` (date)
- `count_sessions` (integer)
- `moving_avg_3day` (numeric) — 3-day moving average rounded to 2 decimals
- `is_spike` (boolean) — TRUE if count_sessions > moving_avg_3day * 1.5

**Requirements:**
- Use `user_sessions_daily` table
- Use window function with explicit ROWS frame (ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)
- Only include rows where a full 3-day window exists (exclude first 2 days per user)
- Order by user_id, date

**Difficulty Rating:** 4/5

---

## Task 2: Percentile Ranking with Category Context

**Scenario:**
For each product, calculate its revenue percentile rank within its category and across all products. Identify products that are top performers in their category (top 25%) but underperformers overall (bottom 50%).

**Expected Output Columns:**
- `product_id` (integer)
- `category_name` (varchar)
- `total_revenue` (numeric) — sum of quantity * price
- `category_percentile` (numeric) — PERCENT_RANK within category (0 to 1)
- `global_percentile` (numeric) — PERCENT_RANK across all products (0 to 1)
- `category_star_global_underperformer` (boolean) — TRUE if category_percentile >= 0.75 AND global_percentile < 0.50

**Requirements:**
- Use `orders_products`, `products`, and `product_categories` tables
- Use PERCENT_RANK() window function with different partitions
- Only show products that have at least one sale
- Order by category_name, category_percentile DESC

**Difficulty Rating:** 4/5

---

## Task 3: Gap and Island Detection - User Order Gaps

**Scenario:**
Identify significant gaps in user ordering behavior. For each user, find periods where they went more than 30 days between consecutive orders. Calculate the gap duration and the revenue lost (compare average order amount during active periods vs the gap period's opportunity cost).

**Expected Output Columns:**
- `user_id` (integer)
- `previous_order_date` (timestamp) — date of order before the gap
- `next_order_date` (timestamp) — date of order after the gap
- `gap_days` (integer) — days between orders
- `user_avg_order_amount` (numeric) — user's average order amount, rounded to 2 decimals
- `estimated_missed_orders` (numeric) — gap_days / user's average days between orders (their normal frequency)
- `potential_lost_revenue` (numeric) — estimated_missed_orders * user_avg_order_amount, rounded to 2 decimals

**Requirements:**
- Use `orders` table
- Use LAG() to find previous order per user
- Calculate each user's normal ordering frequency (average days between their orders)
- Only show gaps > 30 days
- Only include users who have at least 3 orders (to establish a pattern)
- Order by potential_lost_revenue DESC

**Difficulty Rating:** 5/5

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Window frame specifications (ROWS vs RANGE, frame boundaries)
- PERCENT_RANK vs NTILE vs CUME_DIST usage
- Gap-and-island problem-solving techniques
- Self-referential calculations (comparing to own averages)

## Tips

- ROWS BETWEEN: Operates on physical row count (ROWS BETWEEN 2 PRECEDING AND CURRENT ROW = exactly 3 rows)
- RANGE BETWEEN: Operates on value ranges (useful for dates but trickier)
- PERCENT_RANK: Returns position as fraction (0 = first, 1 = last)
- CUME_DIST: Returns cumulative distribution (percentage of rows with value <= current)
- LAG(column, 1): Previous row's value in partition
- Gap detection: Compare current row to LAG, look for large differences
- Frame filtering: Use a CTE with ROW_NUMBER() to filter out incomplete windows

Good luck! These are more challenging as requested.
