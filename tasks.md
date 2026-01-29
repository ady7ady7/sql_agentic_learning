# Daily SQL Practice Tasks

**Generated:** 2026-01-25
**Week 7, Day 5 Focus:** Recursive CTEs — Week Wrap-Up

---

## Final Day of Week 7

Strong finish yesterday. Today we consolidate with practical scenarios combining everything you've learned.

---

## Task 1: Daily Revenue with 7-Day Moving Average

**Scenario:**
For each day with orders, show the daily revenue AND a 7-day trailing moving average (average of that day plus the 6 preceding days).

**Expected Output Columns:**
- `order_date` (date)
- `daily_revenue` (numeric) — total revenue that day
- `moving_avg_7day` (numeric) — average of last 7 days, rounded to 2 decimals

**Requirements:**
- Aggregate orders by date
- Use window function AVG() with frame: ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
- Order by date

**Difficulty Rating:** 3/5

**Note:** No recursive CTE needed — this practices window frames.

---

## Task 2: Loan Amortization Schedule

**Scenario:**
A $5,000 loan at 1% monthly interest is paid off with $500 monthly payments. Generate the amortization schedule until the loan is paid off.

**Expected Output Columns:**
- `month` (integer) — payment number (1, 2, 3, ...)
- `starting_balance` (numeric) — balance at start of month
- `interest` (numeric) — interest charged (1% of starting balance)
- `payment` (numeric) — monthly payment ($500, or less if balance + interest < $500)
- `ending_balance` (numeric) — balance after payment (starting + interest - payment)

**Requirements:**
- Use recursive CTE
- Start with $5,000 balance
- Each month: add 1% interest, subtract $500 payment
- Final payment should be exactly what's owed (not overpay)
- Terminate when ending_balance <= 0

**Difficulty Rating:** 4/5

**Hint:** Use LEAST(500, starting_balance + interest) for the final payment logic.

---

## Task 3: Product Sales Ranking by Month

**Scenario:**
For each month, rank products by their total revenue and show only the top 3 products per month.

**Expected Output Columns:**
- `month` (date) — first day of month
- `product_name` (varchar)
- `monthly_revenue` (numeric) — sum of quantity * price for that product that month
- `rank` (bigint) — rank within that month (1, 2, 3)

**Requirements:**
- Join orders, orders_products, and products
- Aggregate by month and product
- Use RANK() or ROW_NUMBER() with PARTITION BY month
- Filter to only show rank <= 3
- Order by month, rank

**Difficulty Rating:** 4/5

**Note:** No recursive CTE — practices window ranking with PARTITION BY.

---

## Submission Instructions

Today's tasks:
1. Task 1 — Window frame (moving average) (3/5)
2. Task 2 — Recursive loan amortization with conditional logic (4/5)
3. Task 3 — Window ranking with PARTITION BY (4/5)

After today's review, we'll do the **Week 7 Recap**.

Good luck!
