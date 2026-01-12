# Daily SQL Practice Tasks

**Generated:** 2026-01-10
**Week 5, Day 2 Focus:** JSON Operations, Advanced Window Frames, Complex Subqueries

---

## Task 1: First and Last Value in Same Row

**Scenario:**
The analytics team wants to see each user's transaction history with both their first and most recent transaction details in the same row for comparison.

**Expected Output Columns:**
- `user_id` (integer)
- `current_transaction_id` (integer)
- `current_amount` (numeric)
- `current_date` (date)
- `first_transaction_amount` (numeric) — amount of user's very first transaction
- `first_transaction_date` (date) — date of user's very first transaction
- `last_transaction_amount` (numeric) — amount of user's most recent transaction
- `last_transaction_date` (date) — date of user's most recent transaction

**Requirements:**
- Use `transactions` table
- Use FIRST_VALUE and LAST_VALUE window functions with proper frames
- Or use FIRST_VALUE with reversed ORDER BY for last values
- Include all transactions (every row should show first/last context)
- Order by user_id, current_date

**Difficulty Rating:** 3/5

---

## Task 2: Running Total with Reset

**Scenario:**
The finance team wants to see daily order amounts with a running total that resets each month. Calculate cumulative revenue within each month.

**Expected Output Columns:**
- `order_date` (date)
- `daily_revenue` (numeric) — total revenue for this date
- `month` (integer) — month number
- `year` (integer) — year number
- `monthly_running_total` (numeric) — cumulative sum within the month, reset at each new month

**Requirements:**
- Use `orders` table
- Extract year and month from order dates
- Calculate daily revenue sum
- Use window function with PARTITION BY year, month for monthly reset
- Order by year, month, order_date

**Difficulty Rating:** 3/5

---

## Task 3: Comparing Current Row to Next Row

**Scenario:**
The operations team wants to identify order amount increases. For each order, show the next order amount for that user to identify when spending increased.

**Expected Output Columns:**
- `order_id` (integer)
- `user_id` (integer)
- `amount` (numeric)
- `order_date` (date)
- `next_order_amount` (numeric) — amount of the next order for this user
- `next_order_date` (date) — date of the next order for this user
- `amount_increase` (numeric) — difference (next_order_amount - amount)
- `spending_increased` (boolean) — TRUE if next order amount is higher

**Requirements:**
- Use `orders` table
- Use LEAD window function to get next order details
- Calculate amount difference
- Include all orders (last order per user will have NULL next values)
- Order by user_id, order_date

**Difficulty Rating:** 3/5

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Window function frames (ROWS vs RANGE)
- PARTITION BY for grouping within window functions
- LEAD/LAG for row-to-row comparisons
- Proper NULL handling

## Tips

- FIRST_VALUE: `FIRST_VALUE(column) OVER (PARTITION BY group ORDER BY sort_col)`
- LAST_VALUE needs explicit frame: `LAST_VALUE(column) OVER (... ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)`
- Or use FIRST_VALUE with reversed ORDER BY: `FIRST_VALUE(column) OVER (PARTITION BY group ORDER BY sort_col DESC)`
- Running totals: `SUM(column) OVER (PARTITION BY group ORDER BY sort_col)`
- LEAD: `LEAD(column, 1) OVER (PARTITION BY group ORDER BY sort_col)` gets next row

Good luck!
