# Daily SQL Practice Tasks

**Generated:** 2025-12-11
**Week 2, Day 1 Focus:** Recursive CTEs, Advanced Aggregations, Grouping Sets

---

## Task 1: Running Balance with Window Functions

**Scenario:**
The finance team needs to see each user's running balance over time. For each transaction, calculate the cumulative sum of transaction amounts (partitioned by user, ordered by transaction timestamp).

**Expected Output Columns:**
- `user_id` (integer)
- `transaction_id` (integer)
- `created_at` (timestamp)
- `amount` (numeric)
- `running_balance` (numeric) — cumulative sum of amounts for this user up to this transaction

**Requirements:**
- Use `transactions` table
- Use SUM() OVER with proper window frame
- Partition by user_id, order by created_at
- Exclude transactions with NULL user_id or NULL amount
- Order by `user_id` ASC, `created_at` ASC

**Difficulty Rating:** 3/5

---

## Task 2: Category Rollup — Total and Subtotals

**Scenario:**
The product team wants a report showing revenue by product category with subtotals. Show individual category revenues AND a grand total row using ROLLUP or GROUPING SETS.

**Expected Output Columns:**
- `category_name` (varchar) — category name, or NULL for grand total row
- `total_revenue` (numeric) — sum of (quantity * price)
- `order_count` (bigint) — count of distinct orders

**Requirements:**
- Use `product_categories`, `products`, `orders_products`, `orders` tables
- Use GROUP BY ROLLUP or GROUPING SETS to generate subtotal row
- Only include orders from 2025
- Order by `total_revenue` DESC NULLS LAST (grand total last)

**Difficulty Rating:** 4/5

---

## Task 3: Self-Join — Users from Same City

**Scenario:**
The marketing team wants to identify pairs of users from the same city for a referral program. Find all unique pairs of users who share the same city (exclude NULL cities).

**Expected Output Columns:**
- `city` (varchar)
- `user_id_1` (integer) — first user in pair
- `user_id_2` (integer) — second user in pair (always > user_id_1 to avoid duplicates)
- `users_in_city` (bigint) — total count of users in this city

**Requirements:**
- Use `users` table with self-join
- Exclude users with NULL city
- Ensure user_id_1 < user_id_2 to avoid duplicate pairs
- Calculate total users per city using window function
- Order by `city` ASC, `user_id_1` ASC

**Difficulty Rating:** 3/5

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Logic correctness and query structure
- Window frame usage
- GROUPING SETS / ROLLUP implementation
- Self-join efficiency
- Alternative approaches

## Tips

- For running balance: `SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)`
- ROLLUP generates subtotal rows automatically: `GROUP BY ROLLUP(category_name)`
- Self-join deduplication: `FROM users u1 JOIN users u2 ON u1.city = u2.city AND u1.id < u2.id`
- Filter early in JOIN conditions to reduce result set size

Good luck!
