
### Task Archive: 2025-12-07 (Week 1, Day 4)

# Daily SQL Practice Tasks

**Generated:** 2025-12-07
**Week 2, Day 1 Focus:** NTILE, Percentiles, String Functions, Complex Filtering

---

## Task 1: Customer Segmentation with NTILE

**Scenario:**
The marketing team wants to segment users into 5 equal groups (quintiles) based on their total order spending. Calculate each user's total spend and assign them to a quintile (1-5, where 1 = lowest spenders, 5 = highest spenders).

**Expected Output Columns:**
- `user_id` (integer)
- `total_spent` (numeric) — sum of all order amounts for this user
- `spending_quintile` (bigint) — quintile ranking (1-5)
- `users_in_quintile` (bigint) — count of users in the same quintile

**Requirements:**
- Use `orders` table
- Apply NTILE(5) window function to create quintiles
- Calculate total spending per user
- Only include users with at least 1 order
- Group to count users per quintile
- Order by `spending_quintile` ASC

**Difficulty Rating:** 3/5

WITH users_total_spendings AS (
	SELECT 
		user_id,
		SUM(amount) AS total_spent
	FROM orders
	GROUP BY user_id
	HAVING COUNT(*) >= 1
	),
users_spendings_quintiles AS (
	SELECT 
		user_id,
		total_spent,
		NTILE(5) OVER (ORDER BY total_spent) AS spending_quintile
	FROM users_total_spendings
	)
SELECT 
	user_id,
	total_spent,
	spending_quintile,
	COUNT(user_id) OVER (PARTITION BY spending_quintile) AS users_in_quintile
FROM users_spendings_quintiles
ORDER BY spending_quintile


---

## Task 2: Products Never Purchased Together

**Scenario:**
The e-commerce team wants to identify products that have NEVER been purchased together in the same order. For a specific product (e.g., product_id = 1), find all other products that have never appeared in any order containing that product.

**Expected Output Columns:**
- `product_id` (integer) — ID of product never bought with product 1
- `product_name` (varchar)
- `times_sold` (bigint) — how many times this product has been sold (in any order)

**Requirements:**
- Use `orders_products` and `products` tables
- Use anti-join pattern to find products never in same orders as product_id = 1
- Exclude product_id = 1 itself from results
- Calculate how many times each product has been sold
- Order by `times_sold` DESC

**Difficulty Rating:** 4/5

WITH orders_with_p1 AS (
	SELECT 
	order_id FROM orders_products
	WHERE product_id = 1
	),
other_products AS (
	SELECT 
		product_id
		FROM orders_products op
	JOIN orders_with_p1 owp ON op.order_id = owp.order_id
	WHERE op.product_id != 1
	)
SELECT
	op.product_id,
	p.name AS product_name,
	COUNT(*) AS times_sold
FROM orders_products op 
JOIN products p ON op.product_id  = p.id
WHERE op.product_id NOT IN
(SELECT * FROM other_products)
GROUP BY op.product_id, p.name
ORDER BY times_sold DESC

Please also note, that I DID NOT count the actual quantity of the products sold, but rather the number of times (orders) each of these products appeared in. I was able to do that easily, but since you didn't ask me to count their quantity, I simply followed your requirements.


---

## Task 3: Median Session Count Per User

**Scenario:**
The analytics team wants to find the median number of sessions per user across all users. Calculate the median of total sessions (sum of count_sessions) for each user.

**Expected Output Columns:**
- `median_sessions` (numeric) — the median total session count across all users

**Requirements:**
- Use `user_sessions_daily` table
- Calculate total sessions per user (SUM of count_sessions)
- Use PERCENTILE_CONT(0.5) to find the median
- Return a single row with the median value

**Difficulty Rating:** 3/5

WITH users_total_sessions AS (
SELECT 
	user_id,
	SUM(count_sessions) AS total_sessions
FROM user_sessions_daily
GROUP BY user_id
)
SELECT 
	PERCENTILE_CONT(0.5) WITHIN GROUP(ORDER BY total_sessions)
FROM users_total_sessions

Very easy task, but also worth doing, due to the different/unusual syntax of this window function.

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Logic correctness and query structure
- Window function usage
- Efficiency considerations
- Alternative approaches

## Tips

- NTILE(n) divides rows into n equal buckets based on ordering
- PERCENTILE_CONT(0.5) calculates the median (50th percentile)
- For "never together" queries, use NOT EXISTS or anti-join patterns
- Remember to aggregate before applying NTILE

Good luck!

### Task Archive: 2025-12-10 (Week 1, Day 5)

# Daily SQL Practice Tasks

**Generated:** 2025-12-10
**Week 1, Day 5 Focus:** String Functions, Complex Date Logic, Multiple Window Functions

---

## Task 1: Email Domain Analysis with String Functions

**Scenario:**
The marketing team wants to analyze user email domains to understand which email providers are most common. Extract the domain from each user's email address, count users per domain, and calculate what percentage of total users each domain represents.

**Expected Output Columns:**
- `email_domain` (text) — the domain extracted from email (e.g., 'gmail.com')
- `user_count` (bigint) — number of users with this domain
- `percentage_of_total` (numeric) — percentage of all users with email addresses

**Requirements:**
- Use `users` table
- Extract domain using string functions (SUBSTRING, POSITION, or SPLIT_PART)
- Exclude users with NULL emails
- Calculate percentage rounded to 2 decimal places
- Order by `user_count` DESC

**Difficulty Rating:** 3/5

WITH users_w_domains AS (
	SELECT 
		*,
		SPLIT_PART(email, '@', 2) AS email_domain
	FROM users
	WHERE email IS NOT NULL
	),
domains_counts AS (
	SELECT 
		email_domain,
		COUNT(id) AS user_count
	FROM users_w_domains 
	GROUP BY email_domain
	),
total_user_count AS (
	SELECT
	COUNT(*) AS users_total
	FROM users
	)
SELECT 
	dc.email_domain,
	dc.user_count,
	ROUND(dc.user_count::NUMERIC / tuc.users_total * 100, 2) AS percentage_of_total
FROM domains_counts dc
CROSS JOIN total_user_count tuc


---

## Task 2: Transaction Streaks — Consecutive Days

**Scenario:**
The analytics team wants to identify users who made transactions on consecutive days and find the longest streak for each user. A "streak" is a sequence of consecutive calendar days with at least one transaction.

**Expected Output Columns:**
- `user_id` (integer)
- `longest_streak` (integer) — maximum number of consecutive days with transactions
- `streak_start_date` (date) — first day of their longest streak
- `streak_end_date` (date) — last day of their longest streak

**Requirements:**
- Use `transactions` table
- Extract date from created_at timestamp
- Use window functions and CTEs to identify streaks
- Only include users with at least one streak of 3+ consecutive days
- Order by `longest_streak` DESC, then `user_id` ASC

**Difficulty Rating:** 5/5

WITH users_transaction_days AS (
	SELECT 
		DISTINCT user_id, (DATE(created_at)) AS transaction_day
	FROM transactions
	ORDER BY user_id
	),
users_t_days_next AS (
SELECT 
	user_id,
	transaction_day,
	LEAD(transaction_day) OVER (PARTITION BY user_id ORDER BY transaction_day) AS next_t_day
FROM users_transaction_days
)
SELECT *,
next_t_day - transaction_day AS days_diff,
RANK() OVER (PARTITION BY user_id ORDER BY transaction_day) AS streak_duration
FROM users_t_days_next
WHERE next_t_day IS NOT NULL
AND next_t_day - transaction_day = 1

With this code I've found out that THERE WERE NO streaks LONGER THAN 1 day, which simply made me stop trying to go further, as it doesn't make sense. You sohuld understand that.


---

## Task 3: Product Performance — Multiple Rankings

**Scenario:**
The product team wants to see products ranked by three different metrics simultaneously: total quantity sold, total revenue generated, and number of distinct orders. Create a comprehensive view showing all three rankings side by side.

**Expected Output Columns:**
- `product_id` (integer)
- `total_quantity` (numeric) — sum of quantity sold
- `total_revenue` (numeric) — sum of (quantity * price)
- `distinct_orders` (bigint) — count of distinct orders containing this product
- `rank_by_quantity` (bigint) — rank by total quantity
- `rank_by_revenue` (bigint) — rank by total revenue
- `rank_by_orders` (bigint) — rank by distinct orders

**Requirements:**
- Use `products`, `orders_products` tables
- Calculate all three metrics per product
- Apply RANK() three times with different ORDER BY clauses
- Include all products that have been sold at least once
- Order by `rank_by_revenue` ASC

**Difficulty Rating:** 4/5


WITH orders_products_summary AS (
SELECT 
p.id,
SUM(op.quantity) AS total_quantity,
SUM(op.quantity * p.price) AS total_revenue,
COUNT(op.order_id) AS distinct_orders
FROM orders_products op
JOIN products p ON op.product_id = p.id
GROUP BY p.id
HAVING COUNT(op.order_id) >= 1
)
SELECT 
	*,
	RANK() OVER (ORDER BY total_quantity DESC) AS rank_by_quantity,
	RANK() OVER (ORDER BY total_revenue DESC) AS rank_by_revenue,
	RANK() OVER (ORDER BY distinct_orders DESC) AS rank_by_orders
FROM orders_products_summary
ORDER BY rank_by_revenue

You didn't mention it here, ALTHOUGH YOU SHOULD - that we should ORDER these ranks by DESC values (from top total_quantity to low total_quantity, revenue, distinct_orders etc.). I did it, but please specify it next time. If i didn't do it, you shouldn't also take away points from me later.

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Logic correctness and query structure
- String function usage and efficiency
- Window function mastery
- Alternative approaches

## Tips

- For email domains: `SPLIT_PART(email, '@', 2)` extracts everything after '@'
- For consecutive days: Consider using LAG() to compare dates, then use a "gaps and islands" pattern
- For percentage: `(count * 100.0 / total)` ensures decimal division
- You can apply multiple RANK() functions with different ORDER BY in the same SELECT

Good luck!
