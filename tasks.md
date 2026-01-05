# Daily SQL Practice Tasks

**Generated:** 2025-12-31
**Week 4, Day 2 Focus:** Advanced Filtering, Subqueries, NULL Handling

---

## Task 1: Users Without Any Orders

**Scenario:**
The customer success team wants to identify registered users who have never placed an order. This helps them target inactive users for re-engagement campaigns.

**Expected Output Columns:**
- `user_id` (integer)
- `first_name` (varchar)
- `last_name` (varchar)
- `email` (varchar)
- `days_since_registration` (integer) — days between created_at and current date

**Requirements:**
- Use `users` and `orders` tables
- Find users who do not have any orders
- Calculate how long they've been registered
- Only include users with non-null email addresses
- Order by `days_since_registration` DESC (longest registered users first)

**Difficulty Rating:** 3/5

---

## Task 2: Products Never Ordered

**Scenario:**
The inventory team wants to identify products that exist in the catalog but have never been ordered. These might be discontinued items or products with poor market fit.

**Expected Output Columns:**
- `product_id` (integer)
- `product_name` (varchar)
- `category_name` (varchar)
- `price` (numeric)

**Requirements:**
- Use `products`, `product_categories`, `orders_products` tables
- Find products that have never appeared in any order
- Include product category information
- Order by `price` DESC

**Difficulty Rating:** 3/5

---

## Task 3: Cities with Above-Average User Count

**Scenario:**
The expansion team wants to identify high-concentration cities. Show cities that have more users than the average number of users per city, along with their exact counts.

**Expected Output Columns:**
- `city` (varchar)
- `user_count` (bigint) — number of users in this city
- `avg_users_per_city` (numeric) — average number of users across all cities, rounded to 2 decimals
- `users_above_avg` (numeric) — how many users above average this city has, rounded to 2 decimals

**Requirements:**
- Use `users` table
- Calculate user count per city
- Calculate average users across all cities
- Filter to only cities with above-average user counts
- Exclude NULL cities
- Order by `user_count` DESC

**Difficulty Rating:** 4/5

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Anti-join patterns (LEFT JOIN with IS NULL, NOT EXISTS, NOT IN)
- Subqueries for filtering
- Average calculations with comparisons
- NULL handling strategies

## Tips

- Anti-join pattern: `LEFT JOIN ... WHERE right_table.id IS NULL`
- Alternative: `NOT EXISTS (SELECT 1 FROM ... WHERE ...)`
- Alternative: `WHERE column NOT IN (SELECT ...)`
- For averages: subquery or window function
- NULL handling: WHERE column IS NOT NULL

Good luck!
