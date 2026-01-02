# Daily SQL Practice Tasks

**Generated:** 2025-12-30
**Week 4, Day 1 Focus:** Practical Analytics, Data Quality Checks, Business Metrics

---

## Task 1: Product Inventory Analysis

**Scenario:**
The inventory team wants to understand which products are ordered most frequently and in what quantities. For each product, calculate total quantity sold, number of orders it appears in, and average quantity per order.

**Expected Output Columns:**
- `product_name` (varchar)
- `category_name` (varchar)
- `total_quantity_sold` (numeric) — sum of all quantities ordered
- `order_count` (bigint) — number of distinct orders containing this product
- `avg_quantity_per_order` (numeric) — average quantity ordered per order, rounded to 2 decimals

**Requirements:**
- Use `products`, `product_categories`, `orders_products` tables
- Calculate total quantity sold across all orders
- Count distinct orders (not total line items)
- Order by `total_quantity_sold` DESC

**Difficulty Rating:** 3/5

---

## Task 2: User Activity Cohort Analysis

**Scenario:**
The marketing team wants to segment users based on when they first registered (cohort analysis). Group users by their registration month/year and show how many are still active.

**Expected Output Columns:**
- `cohort_year` (integer) — year of registration
- `cohort_month` (integer) — month of registration (1-12)
- `total_users_in_cohort` (bigint) — total users who registered in this month
- `active_users_in_cohort` (bigint) — users from this cohort who are currently active (is_active = TRUE)
- `retention_rate` (numeric) — percentage of cohort that is still active, rounded to 2 decimals

**Requirements:**
- Use `users` table
- Extract year and month from created_at (registration date)
- Count total users per cohort
- Count active users per cohort (is_active = TRUE)
- Calculate retention rate as (active_users / total_users) * 100
- Order by `cohort_year` DESC, `cohort_month` DESC (newest cohorts first)

**Difficulty Rating:** 3/5

---

## Task 3: Transaction Type Distribution by User

**Scenario:**
The finance team wants to understand user transaction patterns. For each user, show how many transactions they've made of each type (withdrawal, payment, transfer, deposit, purchase).

**Expected Output Columns:**
- `user_id` (integer)
- `total_transactions` (bigint) — total number of transactions for this user
- `withdrawal_count` (bigint) — count of withdrawal transactions
- `payment_count` (bigint) — count of payment transactions
- `transfer_count` (bigint) — count of transfer transactions
- `deposit_count` (bigint) — count of deposit transactions
- `purchase_count` (bigint) — count of purchase transactions

**Requirements:**
- Use `transactions` table
- Count transactions by type for each user
- Use conditional aggregation (CASE WHEN or FILTER) to count each type
- Only include users who have at least one transaction
- Order by `total_transactions` DESC

**Difficulty Rating:** 3/5

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Multi-table JOINs and aggregations
- Cohort analysis patterns
- Conditional aggregation techniques (CASE WHEN vs FILTER)
- Percentage calculations and rounding

## Tips

- COUNT(DISTINCT column) for counting unique values
- EXTRACT for pulling year/month from timestamps
- Conditional aggregation: `SUM(CASE WHEN type = 'withdrawal' THEN 1 ELSE 0 END)`
- Or use FILTER: `COUNT(*) FILTER (WHERE type = 'withdrawal')`
- Retention rate = (active / total) * 100

Good luck!