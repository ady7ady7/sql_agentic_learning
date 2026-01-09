# Daily SQL Practice Tasks

**Generated:** 2026-01-09
**Week 5, Day 1 Focus:** Recursive CTEs, Advanced String Manipulation, Complex Date Logic

---

## Task 1: Date Series Generation with Recursive CTE

**Scenario:**
The analytics team needs a complete date series for reporting, even for dates with no activity. Generate all dates between the first and last order in the database, then show daily order counts (including zero for dates with no orders).

**Expected Output Columns:**
- `date` (date) — each date in the series
- `order_count` (bigint) — number of orders on this date (0 if none)
- `cumulative_orders` (bigint) — running total of orders from first date to current date
- `days_since_first_order` (integer) — days elapsed since the very first order

**Requirements:**
- Use a recursive CTE to generate the date series
- Start from the earliest order date, end at the latest order date
- LEFT JOIN with orders to get counts (use COALESCE for zero counts)
- Calculate cumulative sum using window function
- Order by date ASC

**Difficulty Rating:** 4/5

---

## Task 2: Email Validation and Domain Categorization

**Scenario:**
The data quality team wants to identify and categorize email addresses. Find users with potentially invalid emails and categorize email domains into business types.

**Expected Output Columns:**
- `user_id` (integer)
- `email` (varchar)
- `domain` (varchar) — extracted domain from email
- `domain_category` (text) — "Free" (gmail, yahoo, hotmail), "Business" (company domains), "Educational" (.edu), "Other"
- `email_format_valid` (boolean) — TRUE if email contains exactly one @ and at least one dot after @

**Requirements:**
- Use `users` table
- Extract domain using string functions (SPLIT_PART or SUBSTRING)
- Validate email format: must have exactly 1 @, and at least 1 dot after @
- Categorize domains using CASE WHEN
- Only include users with non-null emails
- Order by domain_category, then domain

**Difficulty Rating:** 3/5

---

## Task 3: Transaction Frequency Patterns

**Scenario:**
The fraud team wants to identify unusual transaction patterns. For each user, calculate their typical transaction frequency, then flag periods where they deviated significantly.

**Expected Output Columns:**
- `user_id` (integer)
- `transaction_id` (integer)
- `transaction_date` (date)
- `days_since_prev` (integer) — days since previous transaction
- `user_avg_gap` (numeric) — this user's average gap between transactions, rounded to 2 decimals
- `deviation_from_avg` (numeric) — how far this gap is from their average (days_since_prev - user_avg_gap), rounded to 2 decimals
- `unusual_pattern` (boolean) — TRUE if days_since_prev is more than 2x the user's average gap

**Requirements:**
- Use `transactions` table
- Calculate days between consecutive transactions using LAG
- Calculate each user's average gap between transactions
- Compare current gap to average
- Only include transactions with a previous transaction (exclude each user's first transaction)
- Order by user_id, transaction_date

**Difficulty Rating:** 4/5

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Recursive CTE construction and termination conditions
- String manipulation and validation logic
- Window functions with LAG and averages
- NULL handling in date calculations

## Tips

- Recursive CTE structure:
  ```sql
  WITH RECURSIVE series AS (
      SELECT base_value
      UNION ALL
      SELECT next_value FROM series WHERE condition
  )
  ```
- Email validation: `LENGTH(email) - LENGTH(REPLACE(email, '@', '')) = 1`
- LAG with NULL handling: Use WHERE prev_transaction IS NOT NULL to filter first transactions
- Window function for averages: Can use AVG() OVER (PARTITION BY user_id)

Good luck!
