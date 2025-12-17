# Daily SQL Practice Tasks

**Generated:** 2025-12-14
**Week 2, Day 4 Focus:** Correlated Subqueries, GREATEST/LEAST Functions, Complex JOINs

---

## Task 1: Users Who Spent Above Their Category Average

**Scenario:**
Find users whose total spending is above the average for their country. Use a correlated subquery to compare each user's spending against their country's average.

**Expected Output Columns:**
- `user_id` (integer)
- `country` (varchar)
- `user_total_spent` (numeric) — sum of all order amounts for this user
- `country_avg_spent` (numeric) — average spending for users in this country, rounded to 2 decimals

**Requirements:**
- Use `users` and `orders` tables
- Use correlated subquery in WHERE clause to filter users above their country average
- Exclude users with NULL country
- Calculate both user total and country average
- Order by `user_total_spent` DESC

**Difficulty Rating:** 4/5

---

## Task 2: Latest Transaction Per User with GREATEST

**Scenario:**
For each user, show their most recent transaction and use GREATEST to find the maximum amount between their last transaction and their average transaction amount.

**Expected Output Columns:**
- `user_id` (integer)
- `last_transaction_date` (timestamp) — most recent transaction timestamp
- `last_transaction_amount` (numeric) — amount of most recent transaction
- `avg_transaction_amount` (numeric) — average of all transaction amounts, rounded to 2 decimals
- `max_of_last_and_avg` (numeric) — GREATEST(last_amount, avg_amount)

**Requirements:**
- Use `transactions` table
- Use window function to get last transaction per user
- Calculate average transaction amount per user
- Use GREATEST function to compare last vs average
- Exclude transactions with NULL user_id or NULL amount
- Order by `user_id` ASC

**Difficulty Rating:** 4/5

---

## Task 3: Product Pairs Frequently Bought Together

**Scenario:**
Find pairs of products that appear together in at least 5 orders. Use a self-join on orders_products to identify product pairs within the same order.

**Expected Output Columns:**
- `product_id_1` (integer) — first product (always < product_id_2)
- `product_id_2` (integer) — second product
- `product_name_1` (varchar) — name of first product
- `product_name_2` (varchar) — name of second product
- `times_bought_together` (bigint) — count of distinct orders containing both products

**Requirements:**
- Use `orders_products` and `products` tables
- Self-join orders_products on order_id to find product pairs
- Ensure product_id_1 < product_id_2 to avoid duplicates
- Filter for pairs appearing in at least 5 orders
- Order by `times_bought_together` DESC

**Difficulty Rating:** 4/5

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Logic correctness and query structure
- Correlated subquery usage
- GREATEST/LEAST function application
- Self-join efficiency
- Alternative approaches

## Tips

- Correlated subquery: `WHERE user_total > (SELECT AVG(total) FROM orders o2 JOIN users u2 ON o2.user_id = u2.id WHERE u2.country = u.country)`
- GREATEST picks the maximum value: `GREATEST(value1, value2, value3)`
- Self-join for pairs: `FROM orders_products op1 JOIN orders_products op2 ON op1.order_id = op2.order_id AND op1.product_id < op2.product_id`
- Use HAVING to filter aggregated results

Good luck!
