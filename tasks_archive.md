
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

### Task Archive: 2025-12-11 (Week 2, Day 1)

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

SELECT
user_id,
id,
created_at,
amount,
SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS cum_sum
FROM transactions
WHERE user_id IS NOT NULL
AND amount IS NOT NULL


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


	SELECT
		COALESCE(pc.name, 'Total') AS category_name,
		SUM(p.price * op.quantity) AS total_revenue,
		COUNT(DISTINCT(op.order_id)) AS order_count
	FROM orders_products op
	JOIN orders o ON op.order_id = o.id
	JOIN products p ON op.product_id = p.id
	JOIN product_categories pc ON p.category_id = pc.id
	WHERE EXTRACT('YEAR' FROM o.created_at) = 2025
	GROUP BY ROLLUP (pc.name)


    As a note here - I didn't know how to sort these by total_revenue DESC and still keep the grand total last, as it quite doesn't make sense - If I order them DESC, the grand total will automatically become the FIRST value in order - so In the end I've left it unsorted, as it makes the total go to the last position, which is more important IMO - and it's all clear.

    Including the order count is quite weird here, as in Total row we'd expect to also get the total order count, but it doesn't match the total order_count for some reason.


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


WITH cities_counts AS (
SELECT
 city,
 COUNT(id) AS users_in_city
FROM users
GROUP BY city
),
cities_user_pairs AS (
SELECT 
	u1.id AS user_id_1,
	u2.id AS user_id_2,
	u1.city AS u1_city,
	u2.city AS u2_city
FROM users u1
CROSS JOIN users u2
WHERE u1.city = u2.city
AND u1.id < u2.id
AND u1.city IS NOT NULL
ORDER BY u1_city, user_id_1
)
SELECT 
	cu.user_id_1,
	cu.user_id_2,
	cu.u1_city,
	cu.u2_city,
	cc.users_in_city
FROM cities_user_pairs cu
JOIN cities_counts cc ON cu.u1_city = cc.city 

Your idea to calculate the number of users with the window function here is fallacious, as we wouldn't be able to do that properly - unless we'd be able to COUNT DISTINCT user_ids using the Window function, BUT WE CAN'T. So I had to use a different approach, and use CTEs and display the users in city count in the final step.


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

### Task Archive: 2025-12-12 (Week 2, Day 2)

# Daily SQL Practice Tasks

**Generated:** 2025-12-12
**Week 2, Day 2 Focus:** Advanced Date Functions, CASE Expressions, Multiple Aggregations

---

## Task 1: Monthly Active Users (MAU) Calculation

**Scenario:**
The analytics team needs to calculate Monthly Active Users (MAU) for each month. A user is "active" in a month if they have at least one session with count_sessions > 0 during that month.

**Expected Output Columns:**
- `year` (integer) — extracted from date
- `month` (integer) — extracted from date
- `active_users` (bigint) — count of distinct users with sessions > 0 in that month
- `total_sessions` (numeric) — sum of all count_sessions for that month

**Requirements:**
- Use `user_sessions_daily` table
- Extract year and month from date column
- Count distinct users with count_sessions > 0 per month
- Calculate total sessions per month
- Order by `year` ASC, `month` ASC

**Difficulty Rating:** 3/5


WITH users_sessions_all_dates AS (
SELECT
d.date,
EXTRACT('YEAR' FROM d.date) AS year_,
EXTRACT('Month' FROM d.date) AS month_,
COALESCE(usd.id, 0) AS usd_id,
COALESCE(usd.user_id, 0) AS user_id,
COALESCE(usd.count_sessions, 0) AS count_sessions
FROM dates d 
LEFT JOIN user_sessions_daily usd ON d.date = usd."date"
ORDER BY d.date
)
SELECT 
year_,
month_,
SUM(count_sessions) AS total_sessions,
COUNT(DISTINCT(user_id)) AS active_users
FROM users_sessions_all_dates
WHERE count_sessions > 0
AND user_id IN (SELECT id FROM users)
GROUP BY year_, month_
ORDER BY year_, month_

First I used LEFT JOIN so that I can include all dates, including the inactive ones.
Later I had to filter out data with the count_sessions > 0 condition, but after that I also realized that there's more active users than the actual number of users in the database, so I had to filter out users who are not in the database - used a simple subquery for that, as it didn't make sense to write a CTE for such a simple SELECT.

Anyway, I achieved the goal and fulfilled your requirements.


---

## Task 2: Transaction Type Distribution with CASE

**Scenario:**
The finance team wants to analyze transaction types and categorize them into "Income" (deposit, transfer incoming) vs "Expense" (withdrawal, payment, purchase). For each user, show counts and totals for each category.

**Expected Output Columns:**
- `user_id` (integer)
- `income_count` (bigint) — count of deposit/transfer transactions
- `income_total` (numeric) — sum of amounts for deposit/transfer
- `expense_count` (bigint) — count of withdrawal/payment/purchase transactions
- `expense_total` (numeric) — sum of amounts for withdrawal/payment/purchase
- `net_balance` (numeric) — income_total - expense_total

**Requirements:**
- Use `transactions` table
- Use CASE expressions to categorize transaction types
- Consider: "deposit" and "transfer" as income, others as expense
- Exclude transactions with NULL user_id or NULL amount
- Only include users with at least one transaction
- Order by `net_balance` DESC

**Difficulty Rating:** 4/5

WITH transactions_flow AS (
	SELECT
		*,
		CASE
			WHEN TYPE IN ('deposit', 'transfer') THEN 'income' ELSE 'expense'
		END AS flow_type
	FROM transactions
	),
expenses_summary AS (
	SELECT
		user_id,
		COUNT(*) AS expense_count,
		SUM(amount) AS expense_total
	FROM transactions_flow
	WHERE flow_type = 'expense'
	GROUP BY user_id
	ORDER BY user_id
	),
incomes_summary AS (
	SELECT
		user_id,
		COUNT(*) AS income_count,
		SUM(amount) AS income_total
	FROM transactions_flow
	WHERE flow_type = 'income'
	GROUP BY user_id
	ORDER BY user_id
	)
SELECT 
	ins.user_id,
	ins.income_count,
	ins.income_total,
	exs.expense_count,
	exs.expense_total,
	ins.income_total - exs.expense_total AS net_balance
FROM incomes_summary ins
JOIN expenses_summary exs ON ins.user_id = exs.user_id 
ORDER BY net_balance DESC



---

## Task 3: Support Ticket Response Time Analysis

**Scenario:**
The support team wants to analyze response times. For each ticket, calculate the time difference between ticket creation and the first message, then find the average response time per priority level.

**Expected Output Columns:**
- `priority` (varchar) — ticket priority
- `ticket_count` (bigint) — number of tickets with this priority
- `avg_response_minutes` (numeric) — average minutes between ticket creation and first message, rounded to 2 decimals
- `median_response_minutes` (numeric) — median response time in minutes

**Requirements:**
- Use `chat_tickets` and `chat_messages` tables
- Calculate time difference in minutes using EXTRACT(EPOCH FROM (timestamp1 - timestamp2))/60
- Use window function FIRST_VALUE to get first message per ticket
- Use PERCENTILE_CONT(0.5) for median
- Group by priority
- Order by `avg_response_minutes` ASC

**Difficulty Rating:** 4/5

WITH tickets_first_messages AS (
SELECT
DISTINCT(ticket_id),
FIRST_VALUE(created_at) OVER (PARTITION BY ticket_id) AS first_message_time
FROM chat_messages
),
tickets_responses AS (
SELECT 
	DISTINCT(ticket_id),
	FIRST_VALUE(created_at) OVER (PARTITION BY ticket_id) AS response_time
FROM chat_messages
WHERE author_id IS NOT NULL
AND message_type = 'text'
),
tickets_priorities_response_times AS (
	SELECT 
		ct.id,
		ct.priority,
		tr.response_time - tf.first_message_time AS response_time,
		AVG(tr.response_time - tf.first_message_time) OVER (PARTITION BY priority) AS average_response_time
	FROM chat_tickets ct
	JOIN tickets_first_messages tf ON ct.id = tf.ticket_id
	JOIN tickets_responses tr ON ct.id = tr.ticket_id
	)
SELECT
	priority,
	average_response_time,
	COUNT(id) AS ticket_count,
	PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY response_time) AS median_response_times
FROM tickets_priorities_response_times
GROUP BY priority, average_response_time
ORDER BY average_response_time

Note, that the creation time of the ticket was the same as the first_message_time in every single time, so there was no point in calculating that. Also, I didn't use EPOCH as it would only cause more disruption here than it would actually help - the output is perfectly clear.

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Logic correctness and query structure
- Date extraction and manipulation
- CASE expression usage
- Aggregation patterns
- Alternative approaches

## Tips

- EXTRACT(YEAR FROM date_column) and EXTRACT(MONTH FROM date_column) extract date parts
- CASE expressions in aggregations: SUM(CASE WHEN type = 'deposit' THEN amount ELSE 0 END)
- Time differences: EXTRACT(EPOCH FROM (timestamp1 - timestamp2))/60 gives minutes
- PERCENTILE_CONT requires WITHIN GROUP(ORDER BY column)

Good luck!
