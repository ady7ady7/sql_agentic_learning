# Daily SQL Practice Tasks

**Generated:** 2025-12-29
**Week 3, Day 4 Focus:** Recursive CTEs, Advanced String Functions, Complex Filtering

---

## Task 1: Category Hierarchy Traversal

**Scenario:**
Your product_categories table has been extended with a `parent_id` column (self-referencing FK to create category hierarchies). The marketing team wants to see the full category path for each category (e.g., "Electronics > Computers > Laptops").

**Note:** For this exercise, assume the schema has been updated with:
- `product_categories.parent_id` (integer, nullable, FK to product_categories.id)

**Expected Output Columns:**
- `category_id` (integer) — the category ID
- `category_name` (varchar) — the category name
- `category_path` (text) — full path from root to this category, separated by " > "
- `depth` (integer) — depth level in hierarchy (0 = root, 1 = first level child, etc.)

**Requirements:**
- Build a recursive CTE to traverse the category hierarchy
- Start from root categories (parent_id IS NULL)
- Concatenate category names to build the full path
- Track the depth of each category in the tree
- Order by category_path ASC

**Difficulty Rating:** 5/5

---

## Task 2: Email Domain Analysis

**Scenario:**
The analytics team wants to understand email provider distribution among users. Extract the domain from each user's email address and count how many users belong to each domain.

**Expected Output Columns:**
- `email_domain` (text) — the domain part of the email (everything after @)
- `user_count` (bigint) — number of users with this domain
- `percentage` (numeric) — percentage of total users, rounded to 2 decimals

**Requirements:**
- Use `users` table
- Extract domain from email addresses (text after @)
- Calculate count of users per domain
- Calculate percentage of total users
- Only include users with non-null email addresses
- Order by `user_count` DESC

**Difficulty Rating:** 3/5

---

## Task 3: Users with Above-Average Order Frequency

**Scenario:**
The product team wants to identify power users: those who place orders more frequently than the average user. Find all users whose total order count exceeds the overall average order count per user.

**Expected Output Columns:**
- `user_id` (integer)
- `order_count` (bigint) — total orders for this user
- `avg_order_count` (numeric) — the average order count across all users, rounded to 2 decimals
- `orders_above_avg` (numeric) — how many orders above average this user has, rounded to 2 decimals

**Requirements:**
- Use `orders` table
- Calculate total orders per user
- Calculate the average order count across all users
- Filter to only users with above-average order counts
- Show how many orders above average each user has
- Order by `order_count` DESC

**Difficulty Rating:** 4/5

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Recursive CTE structure and termination conditions
- String manipulation functions (SPLIT_PART, SUBSTRING, POSITION, etc.)
- Subqueries vs window functions for average calculations
- Percentage calculations and rounding

## Tips

- Recursive CTEs have two parts: anchor query (base case) and recursive query (joins to itself)
- For string splitting, PostgreSQL offers SPLIT_PART(string, delimiter, position)
- SUBSTRING and POSITION are useful for string extraction
- Average calculations can use window functions or subqueries
- Always consider NULL handling in string operations

Good luck!
