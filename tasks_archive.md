
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

### Task Archive: 2025-12-13 (Week 2, Day 3)

# Daily SQL Practice Tasks

**Generated:** 2025-12-13
**Week 2, Day 3 Focus:** Complex Window Frames, HAVING with Aggregations, Subqueries in SELECT

---

## Task 1: Product Sales Trend — Moving Average

**Scenario:**
The sales team wants to smooth out daily fluctuations in product sales. For each product and date, calculate a 7-day moving average of quantity sold.

**Expected Output Columns:**
- `product_id` (integer)
- `date` (date)
- `daily_quantity` (numeric) — total quantity sold on this date
- `moving_avg_7day` (numeric) — average quantity over current day + 6 preceding days, rounded to 2 decimals

**Requirements:**
- Use `orders`, `orders_products` tables
- Join with `dates` table to ensure all dates included (even if no sales)
- Use window function with ROWS frame (ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
- Calculate daily totals per product first, then apply moving average
- Only include products that have been sold at least once
- Order by `product_id` ASC, `date` ASC

**Difficulty Rating:** 4/5


WITH dates_orders_products AS (
SELECT 
d.date,
COALESCE(o.id, 0) AS order_id,
COALESCE(o.user_id, 0) AS user_id,
COALESCE(o.amount, 0) AS amount,
COALESCE(op.product_id, 0) AS product_id,
COALESCE(op.quantity, 0) AS quantity,
COALESCE(p.price, 0) AS price
FROM dates d
LEFT JOIN orders o ON DATE(o.created_at ) = d."date"
LEFT JOIN orders_products op
LEFT JOIN products p ON op.product_id = p.id
ON o.id = op.order_id
ORDER BY d.date
),
products_daily_revenue_quantity AS (
SELECT
	date,
	product_id,
	SUM(price * quantity) AS daily_revenue,
	SUM(quantity) AS daily_quantity
FROM dates_orders_products
GROUP BY date, product_id
ORDER BY date, product_id
)
SELECT 
	*,
	AVG(daily_revenue) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS moving_avg_7day
FROM products_daily_revenue_quantity

Your requirement to calculate the MOVING AVERAGE from the last 7 days for separate products is weird and really difficult to comply to. Either we are screwed by the days, as having multiple products separately doesn't allow us to GROUP BY the results by date realistically (as if we had 3 products during the past day, we wouldn't be able to reach the last 7 days with ROWS BETWEEN 6 PRECEDING AND CURRENT ROW). At the same time, if we partition the window by product_id, we obviously won't be able to count the empty dates (as they don't have the corresponding product_id there), which is a big issue, BUT there is a chance I miss something or my thinking process is wrong. Not a fan of this query to be honest, and I'd say it could be a bit unrealistic, but I might be wrong - calculating moving averages/window averages is definitely a skill I want to master. 


---

## Task 2: High-Value Customers — HAVING Filter

**Scenario:**
The marketing team wants to identify "high-value" customers who have spent more than $5000 total AND placed more than 10 orders. Show their spending summary.

**Expected Output Columns:**
- `user_id` (integer)
- `total_spent` (numeric) — sum of all order amounts
- `order_count` (bigint) — count of orders
- `avg_order_value` (numeric) — average order amount, rounded to 2 decimals
- `first_order_date` (timestamp) — date of first order
- `last_order_date` (timestamp) — date of most recent order

**Requirements:**
- Use `orders` table
- Use HAVING clause to filter for total_spent > 5000 AND order_count > 10
- Calculate all metrics in one query with GROUP BY
- Order by `total_spent` DESC

**Difficulty Rating:** 3/5


SELECT 
	user_id,
	SUM(amount) AS total_spent,
	COUNT(id) AS order_count,
	AVG(amount) AS avg_order_value,
	MIN(created_at) AS first_order_date,
	MAX(created_at) AS last_order_date
FROM orders
GROUP BY user_id
HAVING SUM(amount) > 5000 AND COUNT(id) > 10
ORDER BY total_spent DESC


---

## Task 3: Category Market Share — Subquery in SELECT

**Scenario:**
The product team wants to see each category's revenue as a percentage of total company revenue. Use a subquery in the SELECT clause to calculate the grand total.

**Expected Output Columns:**
- `category_id` (integer)
- `category_name` (varchar)
- `category_revenue` (numeric) — sum of (quantity * price) for this category
- `total_company_revenue` (numeric) — sum of all revenue (calculated via subquery)
- `market_share_pct` (numeric) — (category_revenue / total) * 100, rounded to 2 decimals

**Requirements:**
- Use `product_categories`, `products`, `orders_products` tables
- Use scalar subquery in SELECT to get total company revenue
- Calculate market share percentage
- Only include categories with at least one sale
- Order by `market_share_pct` DESC

**Difficulty Rating:** 4/5

WITH categories_revenues AS (
SELECT
	p.category_id,
	pc.name,
	ROUND(SUM(op.quantity * p.price), 3) AS category_revenue,
	(SELECT SUM(amount)::NUMERIC FROM orders) AS total_company_revenue
	FROM orders_products op
JOIN products p ON op.product_id = p.id 
JOIN product_categories pc ON p.category_id = pc.id 
GROUP BY p.category_id, pc.name
)
SELECT
	*,
	ROUND(category_revenue::NUMERIC / total_company_revenue::NUMERIC * 100, 2) AS market_share_prct
FROM categories_revenues
ORDER BY market_share_prct DESC



---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Logic correctness and query structure
- Window frame usage (ROWS vs RANGE)
- HAVING clause effectiveness
- Subquery patterns
- Alternative approaches

## Tips

- Moving average frame: `AVG(quantity) OVER (PARTITION BY product_id ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)`
- HAVING filters aggregated results: `GROUP BY user_id HAVING SUM(amount) > 5000 AND COUNT(*) > 10`
- Scalar subquery: `(SELECT SUM(revenue) FROM ...) AS total_revenue`
- For dates with no sales, use LEFT JOIN and COALESCE to show 0

Good luck!

### Task Archive: 2025-12-14 (Week 2, Day 4)

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

WITH users_countries_spend AS (
SELECT 
	o.user_id,
	u.country,
	SUM(o.amount) AS user_total_spent
FROM orders o
JOIN users u ON o.user_id = u.id 
WHERE u.country IS NOT NULL
GROUP BY o.user_id, u.country
),
users_countries_avg_spend AS (
	SELECT 
		*,
		ROUND(AVG(user_total_spent::NUMERIC) OVER (PARTITION BY country), 2) AS country_avg_spent
	FROM users_countries_spend
	ORDER BY user_total_spent DESC
)
SELECT 
	*
FROM users_countries_avg_spend
WHERE user_total_spent > country_avg_spent



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

WITH users_last_avg_transactions AS (
SELECT 
	user_id,
	amount,
	FIRST_VALUE(created_at) OVER (PARTITION BY user_id ORDER BY created_at DESC) AS last_transaction_date,
	FIRST_VALUE(amount) OVER (PARTITION BY user_id ORDER BY created_at DESC) AS last_transaction_amount,
	AVG(amount) OVER (PARTITION BY user_id) AS avg_transaction_amount
FROM transactions
)
SELECT 
	DISTINCT(user_id),
	last_transaction_date,
	last_transaction_amount,
	avg_transaction_amount,
	GREATEST(last_transaction_amount, avg_transaction_amount) AS max_of_last_and_avg
FROM users_last_avg_transactions
ORDER BY user_id

Please note that there was no need to exclude orders with null id as order id is the primary key in our orders table.
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

SELECT 
	op1.product_id AS product_id1,
	op2.product_id AS product_id2,
	p1.name AS product_name1,
	p2.name AS product_name2,
	COUNT(*) AS times_bought_together
FROM orders_products op1
JOIN orders_products op2 ON op1.order_id = op2.order_id
JOIN products p1 ON op1.product_id = p1.id
JOIN products p2 ON op2.product_id = p2.id
WHERE op1.product_id > op2.product_id
GROUP BY op1.product_id, op2.product_id, p1.name, p2.name
HAVING COUNT(*) > 2
ORDER BY times_bought_together DESC

I could filter pairs for appearing in at least 5 orders, but it's pointless, as the max times_bought_together value was 3, so I filtered out all orders below 2 times bought together.


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

### Task Archive: 2025-12-15 (Week 2, Day 5)

# Daily SQL Practice Tasks

**Generated:** 2025-12-15
**Week 2, Day 5 Focus:** UNION/INTERSECT/EXCEPT, Complex CASE Expressions, JSON Functions

---

## Task 1: Combined User Activity — UNION ALL

**Scenario:**
The analytics team wants a unified view of all user activity. Combine data from orders and transactions tables to show all user financial activity in chronological order.

**Expected Output Columns:**
- `user_id` (integer)
- `activity_date` (timestamp)
- `activity_type` (varchar) — 'order' or 'transaction'
- `amount` (numeric)
- `source_table` (varchar) — 'orders' or 'transactions'

**Requirements:**
- Use UNION ALL to combine orders and transactions
- Extract created_at as activity_date from both tables
- Label each row with its source table and activity type
- Exclude rows with NULL user_id or NULL amount
- Order by `user_id` ASC, `activity_date` ASC

**Difficulty Rating:** 3/5

SELECT 
	user_id,
	created_at AS activity_date,
	amount,
	'orders' AS source_table
FROM orders
UNION ALL
SELECT 
	user_id,
	created_at AS activity_date,
	amount,
	'transactions' AS source_table
FROM transactions
GROUP BY user_id, created_at, amount
ORDER BY user_id, activity_date


SELECT 
	user_id,
	created_at AS activity_date,
	amount,
	'orders' AS source_table
FROM orders
WHERE amount IS NOT NULL AND user_id IS NOT NULL
UNION ALL
SELECT 
	user_id,
	created_at AS activity_date,
	amount,
	'transactions' AS source_table
FROM transactions
WHERE amount IS NOT NULL AND user_id IS NOT NULL
GROUP BY user_id, created_at, amount
ORDER BY user_id, activity_date


---

## Task 2: Tiered Pricing with Complex CASE

**Scenario:**
Create a tiered discount system for products based on their price. Calculate the discount percentage and final price after discount using a complex CASE expression.

**Expected Output Columns:**
- `product_id` (integer)
- `product_name` (varchar)
- `original_price` (numeric)
- `discount_pct` (integer) — percentage: 0%, 5%, 10%, 15%, or 20%
- `final_price` (numeric) — price after discount, rounded to 2 decimals

**Requirements:**
- Use `products` table
- CASE expression for discount tiers:
  - price >= 100: 20% discount
  - price >= 75: 15% discount
  - price >= 50: 10% discount
  - price >= 25: 5% discount
  - price < 25: 0% discount
- Calculate final_price = original_price * (1 - discount_pct/100)
- Order by `original_price` DESC

**Difficulty Rating:** 3/5

WITH products_discounts AS (
SELECT 
	*,
	CASE 
		WHEN price >= 100 THEN 0.2
		WHEN price >= 75 THEN 0.15
		WHEN price >= 50 THEN 0.1
		WHEN price >= 20 THEN 0.05
		WHEN price < 25 THEN 0
	END AS discount_rate
FROM products
)
SELECT
	id AS product_id,
	name AS product_name,
	price AS original_price,
	discount_rate,
	ROUND(price - (price * discount_rate), 2) AS final_price
FROM products_discounts
ORDER BY original_price DESC


Please note that I've used discount_rate instead of discount_percent, as it's simply easier for me and more intuitive. It doesn't change the final output, but I named it discount_rate, as discount_prct would suggest that these rates are percents, which they're not (0.2 would suggest a 0.2%, but we know it's actually 20%, so I wanted to make it clear)


---

## Task 3: Users Active in Both Orders and Sessions

**Scenario:**
Find users who are active in BOTH orders (placed at least 1 order) AND sessions (had at least 1 session with count_sessions > 0). Use INTERSECT or an alternative approach.

**Expected Output Columns:**
- `user_id` (integer)
- `order_count` (bigint) — count of orders
- `total_sessions` (numeric) — sum of count_sessions
- `first_order_date` (timestamp) — date of first order
- `last_session_date` (date) — date of most recent session

**Requirements:**
- Use `orders` and `user_sessions_daily` tables
- Find users present in both datasets (INTERSECT or INNER JOIN approach)
- Calculate metrics for matched users
- Order by `order_count` DESC

**Difficulty Rating:** 4/5

WITH users_order_cnt AS (
SELECT 
	user_id,
	COUNT(*) AS order_count,
	MIN(created_at) AS first_order_date
FROM orders
GROUP BY user_id
),
users_sessions_cnt AS (
SELECT
	user_id,
	SUM(count_sessions) AS total_sessions,
	MAX(date) AS last_session_date
FROM user_sessions_daily usd
GROUP BY user_id
)
SELECT
	uoc.user_id,
	uoc.order_count,
	usc.total_sessions,
	uoc.first_order_date,
	usc.last_session_date
FROM users_order_cnt uoc
JOIN users_sessions_cnt usc ON uoc.user_id = usc.user_id
ORDER BY order_count DESC

I used inner join, as honestly it doesn't make sense to use INTERSECT here - I would use it if we were to give only the list of user_ids, but if we need more info, I'd have to use another CTE to extract all the necessary information for the list of user_ids extracted with the INTERSECT.

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Logic correctness and query structure
- UNION/INTERSECT usage
- Complex CASE expression implementation
- Set operation alternatives
- Alternative approaches

## Tips

- UNION ALL includes duplicates, UNION removes them
- INTERSECT finds common elements between two sets
- Complex CASE: `CASE WHEN condition1 THEN value1 WHEN condition2 THEN value2 ELSE default END`
- INTERSECT alternative: `WHERE user_id IN (SELECT user_id FROM other_table)`

Good luck!

### Task Archive: 2025-12-16 (Week 3, Day 1)

# Daily SQL Practice Tasks

**Generated:** 2025-12-16
**Week 3, Day 1 Focus:** Advanced Window Frames, FILTER Clause, Complex Aggregations

---

## Task 1: Revenue by Month with Month-over-Month Growth

**Scenario:**
The finance team wants to see monthly revenue with month-over-month growth comparison. For each month, show the revenue and compare it to the previous month.

**Expected Output Columns:**
- `year` (integer)
- `month` (integer) — 1-12
- `monthly_revenue` (numeric) — total revenue for this month
- `prev_month_revenue` (numeric) — revenue from previous month
- `mom_growth_pct` (numeric) — month-over-month growth percentage, rounded to 2 decimals

**Requirements:**
- Use `orders` table
- Extract year and month from created_at
- Use LAG window function ordered by year, month
- Calculate MoM growth: ((current - previous) / previous) * 100
- Order by `year` ASC, `month` ASC

**Difficulty Rating:** 4/5

WITH orders_y_m AS (
	SELECT
		*,
		EXTRACT('Month' FROM created_at) AS month_,
		EXTRACT('Year' FROM created_at) AS year_
	FROM orders
	),
monthly_rev_y_m AS (
	SELECT
		year_,
		month_,
		SUM(amount) AS monthly_revenue
	FROM orders_y_m
	GROUP BY year_, month_
	ORDER BY year_, month_
	),
prev_m_monthly_rev_y_m AS (
	SELECT 
		*,
		LAG(monthly_revenue) OVER (ORDER BY year_, month_) AS prev_month_revenue
		FROM monthly_rev_y_m
	)
SELECT *,
ROUND((monthly_revenue::NUMERIC - prev_month_revenue::NUMERIC) / prev_month_revenue::NUMERIC * 100, 2) AS mom_growth_pct
FROM prev_m_monthly_rev_y_m

---

## Task 2: Filtered Aggregations — Active vs Inactive Users

**Scenario:**
Compare order statistics between active and inactive users in a single query. Use aggregate functions with FILTER clause or CASE expressions to separate the metrics.

**Expected Output Columns:**
- `active_user_count` (bigint) — count of distinct users where is_active = true who placed orders
- `active_total_revenue` (numeric) — sum of order amounts from active users
- `active_avg_order` (numeric) — average order amount from active users
- `inactive_user_count` (bigint) — count of distinct users where is_active = false who placed orders
- `inactive_total_revenue` (numeric) — sum of order amounts from inactive users
- `inactive_avg_order` (numeric) — average order amount from inactive users

**Requirements:**
- Use `orders` and `users` tables
- Use FILTER (WHERE ...) clause with aggregations OR CASE expressions inside aggregations
- Return a single row with all metrics
- Round averages to 2 decimals

**Difficulty Rating:** 4/5


WITH active_users_data AS (
	SELECT
		COUNT(DISTINCT(user_id)) AS active_user_count,
		ROUND(SUM(amount::NUMERIC), 2) AS active_total_revenue,
		ROUND(AVG(amount::NUMERIC), 2) AS active_avg_order
	FROM users u
	JOIN orders o ON u.id = o.user_id
	WHERE u.is_active = TRUE
	),
inactive_users_data AS (
SELECT
	COUNT(DISTINCT(user_id)) AS inactive_user_count,
	ROUND(SUM(amount::NUMERIC), 2) AS inactive_total_revenue,
	ROUND(AVG(amount::NUMERIC), 2) AS inactive_avg_order
FROM users u
JOIN orders o ON u.id = user_id
WHERE u.is_active = FALSE
)
SELECT * FROM active_users_data
CROSS JOIN inactive_users_data




---

## Task 3: Gap Analysis — Days Between Transactions

**Scenario:**
For each user, find their transaction frequency patterns. Calculate the average gap (in days) between consecutive transactions and identify the longest gap for each user.

**Expected Output Columns:**
- `user_id` (integer)
- `transaction_count` (bigint) — total number of transactions
- `avg_gap_days` (numeric) — average days between consecutive transactions, rounded to 2 decimals
- `max_gap_days` (integer) — maximum days between any two consecutive transactions
- `min_gap_days` (integer) — minimum days between any two consecutive transactions

**Requirements:**
- Use `transactions` table
- Use LAG to get previous transaction date per user
- Calculate date differences for consecutive transactions
- Aggregate: AVG, MAX, MIN of gaps
- Only include users with at least 2 transactions
- Order by `avg_gap_days` DESC

**Difficulty Rating:** 4/5



WITH transactions_prev_and_count AS (
	SELECT 
			*,
			COUNT(id) OVER (PARTITION BY user_id) AS transaction_count,
			LAG(created_at) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_transaction_date
	FROM transactions
	),
users_gap_days AS (
	SELECT *,
		DATE(created_at) - DATE(prev_transaction_date) AS gap_days
	FROM transactions_prev_and_count
	WHERE prev_transaction_date IS NOT NULL
	AND transaction_count >= 2
	)
SELECT
	user_id,
	transaction_count,
	MAX(gap_days) AS max_gap_days,
	MIN(gap_days) AS min_gap_days,
	ROUND(AVG(gap_days), 2) AS avg_gap_days
FROM users_gap_days
GROUP BY user_id, transaction_count
ORDER BY avg_gap_days DESC


---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Logic correctness and query structure
- Window function usage with complex PARTITION BY
- FILTER clause or CASE aggregation patterns
- Date arithmetic and gap calculations
- Alternative approaches

## Tips

- LAG with specific partitioning: `LAG(revenue) OVER (PARTITION BY quarter ORDER BY year)`
- FILTER clause: `COUNT(*) FILTER (WHERE is_active = true)` or use `SUM(CASE WHEN is_active THEN 1 ELSE 0 END)`
- Date difference in days: `DATE_PART('day', date1 - date2)` or `(date1 - date2)`
- For gaps, exclude NULL results from LAG (first transaction has no previous)

Good luck!

### Task Archive: 2025-12-22 (Week 3, Day 2)

# Daily SQL Practice Tasks

**Generated:** 2025-12-22
**Week 3, Day 2 Focus:** Advanced Ranking, Running Totals with Frames, Multiple Window Functions

---

## Task 1: Top 3 Products per Category by Revenue

**Scenario:**
The product team wants to identify the top 3 best-selling products in each category based on total revenue. Show products ranked within their category, but only include the top 3 from each category.

**Expected Output Columns:**
- `category_name` (varchar) — category name from product_categories
- `product_name` (varchar) — product name
- `total_revenue` (numeric) — total revenue for this product (price × quantity across all orders), rounded to 2 decimals
- `category_rank` (bigint) — rank within category (1 = highest revenue in category)

**Requirements:**
- Use `products`, `product_categories`, `orders_products` tables
- Calculate revenue as price × quantity, then SUM for each product
- Use DENSE_RANK() OVER (PARTITION BY category_id ORDER BY total_revenue DESC)
- Filter to only include ranks 1, 2, 3
- Order by `category_name` ASC, `category_rank` ASC

**Difficulty Rating:** 4/5

WITH product_categories_rank AS (
	SELECT 
		pc.name AS category_name,
		p.name AS product_name,
		SUM(p.price * op.quantity) AS total_revenue,
		DENSE_RANK() OVER (PARTITION BY pc.name ORDER BY SUM(p.price * op.quantity) DESC) AS category_rank
	FROM orders_products op
	JOIN products p ON op.product_id = p.id
	JOIN product_categories pc ON p.category_id = pc.id
	GROUP BY pc.name, p.name
	)
SELECT 
	*
FROM product_categories_rank
WHERE category_rank IN (1, 2, 3)
ORDER BY category_name, category_rank



---

## Task 2: Running Total of Daily Revenue with Month Reset

**Scenario:**
Finance wants to see a running total of daily revenue that resets at the start of each month. For each day, show the cumulative revenue within that month up to and including that day.

**Expected Output Columns:**
- `order_date` (date) — the date orders were created
- `daily_revenue` (numeric) — total revenue for that specific day, rounded to 2 decimals
- `running_monthly_total` (numeric) — cumulative revenue within the month up to this day, rounded to 2 decimals
- `year` (integer) — year from order_date
- `month` (integer) — month from order_date

**Requirements:**
- Use `orders` table
- Extract date from created_at timestamp
- Calculate daily revenue: SUM of amount per date
- Use window function with PARTITION BY year, month and ORDER BY date
- Use appropriate frame: ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
- Order by `order_date` ASC

**Difficulty Rating:** 4/5


WITH orders_dates AS (
	SELECT 
		*,
		DATE(created_at) AS order_date,
		EXTRACT('Month' FROM created_at) AS month_,
		EXTRACT('Year' FROM created_at) AS year_
	FROM orders o
	)
SELECT
	amount,
	order_date,
	year_,
	month_,
	ROUND(SUM(amount::NUMERIC) OVER (PARTITION BY order_date ORDER BY order_date), 2) AS daily_revenue,
	ROUND(SUM(amount::NUMERIC) OVER (PARTITION BY year_, month_ ORDER BY order_date), 2) AS running_monthly_total
FROM orders_dates

I don't know why you wanted me to use ROWS BETWEEN X AND CURRENT ROW, but it's absolutely not necessary.
I confirm that dates are ordered from the lowest to the highest date, and the running_monthly_total is properly counted.



---

## Task 3: User Quartiles by Transaction Amount

**Scenario:**
The analytics team wants to segment users into quartiles (4 equal groups) based on their total transaction amount. Assign each user to a quartile (1 = lowest 25%, 4 = highest 25%) and show summary statistics.

**Expected Output Columns:**
- `user_id` (integer)
- `total_transaction_amount` (numeric) — sum of all transaction amounts for this user, rounded to 2 decimals
- `transaction_count` (bigint) — number of transactions for this user
- `user_quartile` (integer) — quartile assignment (1, 2, 3, or 4)

**Requirements:**
- Use `transactions` table
- Calculate total_transaction_amount: SUM(amount) per user
- Calculate transaction_count: COUNT(*) per user
- Use NTILE(4) OVER (ORDER BY total_transaction_amount DESC) for quartile assignment
- Only include users who have at least 1 transaction with non-null amount
- Order by `user_quartile` ASC, `total_transaction_amount` DESC

**Difficulty Rating:** 3/5

WITH users_transactions AS (
SELECT 
	user_id,
	SUM(amount) AS total_transaction_amount,
	COUNT(*) AS transaction_count
FROM transactions
WHERE amount IS NOT NULL
GROUP BY user_id
)
SELECT 
	*,
	NTILE(4) OVER (ORDER BY total_transaction_amount) AS user_quartile
FROM users_transactions
ORDER BY user_quartile, total_transaction_amount DESC

Please note that all rows in transactions table equate to one transaction, so there are no null amounts, but still I've added IS NOT NULl condition to test it.




---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- DENSE_RANK vs RANK vs ROW_NUMBER usage
- Window frame specifications (ROWS vs RANGE)
- NTILE for bucketing/segmentation
- PARTITION BY with multiple columns
- Filtering ranked results (WHERE vs HAVING vs subquery)

## Tips

- DENSE_RANK: No gaps in ranking when ties exist (1, 2, 2, 3)
- RANK: Gaps after ties (1, 2, 2, 4)
- ROW_NUMBER: Always unique (1, 2, 3, 4)
- Frame clause: `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` for running totals
- NTILE(n): Divides rows into n roughly equal buckets
- Filtering ranked results: Use a subquery/CTE, then filter in outer query WHERE clause

Good luck!

### Task Archive: 2025-12-29 (Week 3, Day 3)

# Daily SQL Practice Tasks

**Generated:** 2025-12-22
**Week 3, Day 3 Focus:** Complex Window Functions, Self-Joins, Advanced Aggregations

---

## Task 1: Revenue Percentile Analysis

**Scenario:**
The finance team wants to understand the distribution of order values. For each order, calculate what percentile it falls into compared to all other orders (e.g., an order at the 75th percentile is larger than 75% of all orders).

**Expected Output Columns:**
- `order_id` (integer)
- `user_id` (integer)
- `amount` (double precision) — order amount
- `revenue_percentile` (double precision) — percentile rank (0.0 to 1.0, rounded to 4 decimals)

**Requirements:**
- Use `orders` table
- Calculate the percentile rank for each order based on its amount
- Higher amounts should have higher percentile values
- Only include orders with non-null amounts
- Order by `revenue_percentile` DESC, `order_id` ASC

**Difficulty Rating:** 3/5

WITH orders_percentile_ranks AS (
	SELECT 
		id,
		user_id,
		amount,
		ROUND(PERCENT_RANK()OVER (ORDER BY amount)::NUMERIC, 4) AS percent_rank
	FROM orders
	WHERE amount IS NOT NULL
	ORDER BY percent_rank DESC, id
	)
SELECT 
	*,
	percent_rank > 0.75 AS higher_than_75th_percentile
FROM orders_percentile_ranks


---

## Task 2: Customer Retention — Users with Orders in Consecutive Months

**Scenario:**
The product team wants to identify users who made purchases in consecutive months (e.g., ordered in January and February, or March and April). Find all users who have made orders in at least one pair of consecutive months.

**Expected Output Columns:**
- `user_id` (integer)
- `first_month` (integer) — the earlier month of the consecutive pair (1-12)
- `second_month` (integer) — the later month of the consecutive pair (1-12)
- `year` (integer) — year when this happened
- `orders_in_first_month` (bigint) — count of orders in the first month
- `orders_in_second_month` (bigint) — count of orders in the second month

**Requirements:**
- Use `orders` table
- Find users with orders in consecutive calendar months within the same year
- A user may have multiple consecutive month pairs (e.g., Jan-Feb AND Feb-Mar)
- Order by `user_id` ASC, `year` ASC, `first_month` ASC

**Difficulty Rating:** 5/5


WITH orders_month_year AS (
	SELECT 
		user_id,
		EXTRACT('Month' FROM created_at) AS month_,
		EXTRACT('Year' FROM created_at) AS year_
	FROM orders
	ORDER BY user_id
	),
orders_months AS (
	SELECT
		*,
		FIRST_VALUE(month_) OVER (PARTITION BY user_id ORDER BY month_, year_) AS first_month,
		FIRST_VALUE(month_) OVER (PARTITION BY user_id ORDER BY month_ DESC, year_ DESC) AS last_month,
		LAG(year_) OVER (PARTITION BY user_id ORDER BY month_, year_) AS previous_year
	FROM orders_month_year
	),
eligible_users_orders AS (
	SELECT 
		*
	FROM orders_months
	WHERE last_month - first_month = 1
	),
orders_first_month_cnt AS (
	SELECT
		user_id,
		COUNT(*) AS orders_in_first_month
	FROM eligible_users_orders
	WHERE month_ = first_month
	GROUP BY user_id
	),
orders_second_month_cnt AS (
	SELECT
		user_id,
		COUNT(*) AS orders_in_second_month
	FROM eligible_users_orders
	WHERE month_ = last_month
	GROUP BY user_id
	)
SELECT
	DISTINCT(ofm.user_id),
	euo.first_month,
	euo.last_month AS second_month,
	euo.year_,
	ofm.orders_in_first_month,
	osm.orders_in_second_month
FROM orders_first_month_cnt ofm
JOIN orders_second_month_cnt osm ON ofm.user_id = osm.user_id
JOIN eligible_users_orders euo ON ofm.user_id = euo.user_id
ORDER BY ofm.user_id, euo.year_, euo.first_month


---

## Task 3: Support Ticket Response Time Analysis

**Scenario:**
The support team wants to analyze how quickly they respond to tickets. Calculate the time between ticket creation and the first message sent by support (where `author_id` IS NOT NULL, indicating a support agent message, not a user message).

**Expected Output Columns:**
- `ticket_id` (bigint)
- `ticket_created_at` (timestamp with time zone) — when ticket was created
- `first_response_at` (timestamp with time zone) — timestamp of first support message
- `response_time_minutes` (numeric) — time difference in minutes, rounded to 2 decimals

**Requirements:**
- Use `chat_tickets` and `chat_messages` tables
- Find the first message where `author_id IS NOT NULL` for each ticket
- Calculate time difference in minutes between ticket creation and first response
- Only include tickets that have received at least one support response
- Order by `response_time_minutes` DESC

**Difficulty Rating:** 4/5


WITH tickets_creation_times AS (
	SELECT
		DISTINCT(cm.ticket_id),
		FIRST_VALUE(cm.created_at) OVER (PARTITION BY cm.ticket_id) AS ticket_created_at
	FROM chat_messages cm
	),
ticket_responses AS (
	SELECT 
		ticket_id,
		FIRST_VALUE(created_at) OVER (PARTITION BY ticket_id) AS ticket_response_time
	FROM chat_messages
	WHERE message_type = 'text'
	AND author_id IS NOT NULL
	)
SELECT
	tct.ticket_id,
	tct.ticket_created_at,
	tr.ticket_response_time AS first_response_at,
	tr.ticket_response_time - tct.ticket_created_at AS response_time_minutes
FROM tickets_creation_times tct
JOIN ticket_responses tr ON tct.ticket_id = tr.ticket_id
ORDER BY response_time_minutes DESC


---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Percentile and ranking functions (PERCENT_RANK, CUME_DIST, etc.)
- Self-joins and date arithmetic for consecutive period detection
- Window functions with filtering (FIRST_VALUE, MIN, etc.)
- Time difference calculations (EXTRACT EPOCH, date subtraction)

## Tips

- PERCENT_RANK() returns values from 0 to 1 showing relative position
- CUME_DIST() returns cumulative distribution (percentage of values <= current)
- For consecutive months, consider date arithmetic and comparisons
- FIRST_VALUE with proper ordering can find earliest/latest values
- EXTRACT(EPOCH FROM interval) converts intervals to seconds

Good luck!

### Task Archive: 2025-12-29 (Week 3, Day 4)

# Daily SQL Practice Tasks

**Generated:** 2025-12-29
**Week 3, Day 4 Focus:** Recursive CTEs, Advanced String Functions, Complex Filtering

---

## Task 1: Transaction Sequence Analysis with Self-Join

**Scenario:**
The fraud detection team wants to identify suspicious rapid-fire transactions. Find all instances where the same user made two or more transactions within 5 minutes of each other. Show both transactions in the pair and the time difference between them.

**Expected Output Columns:**
- `user_id` (integer)
- `first_transaction_id` (integer) — ID of the earlier transaction
- `second_transaction_id` (integer) — ID of the later transaction
- `first_transaction_time` (timestamp) — timestamp of earlier transaction
- `second_transaction_time` (timestamp) — timestamp of later transaction
- `time_diff_seconds` (numeric) — time difference in seconds between the two transactions, rounded to 2 decimals

**Requirements:**
- Use `transactions` table
- Find transaction pairs from the same user where the second transaction occurs within 5 minutes (300 seconds) after the first
- Calculate time difference in seconds between the pair
- Avoid duplicate pairs (if transaction A→B is shown, don't show B→A)
- Only include transactions with non-null created_at timestamps
- Order by `user_id` ASC, `first_transaction_time` ASC

**Difficulty Rating:** 4/5


WITH users_transactions AS (
	SELECT
		t1.id AS transaction_id_1,
		t2.id AS transaction_id_2,
		t1.user_id AS u_id,
		t1.amount AS transaction_amount_1,
		t2.amount AS transaction_amount_2,
		t1.created_at AS transaction_time_1,
		t2.created_at AS transaction_time_2
	FROM transactions t1
	JOIN transactions t2 ON t1.user_id = t2.user_id
	WHERE t1.id > t2.id
	ORDER BY u_id, t1.created_at
	),
transactions_differences_in_seconds AS (
SELECT 
	u_id,
	transaction_id_1,
	transaction_id_2,
	transaction_time_1,
	transaction_time_2,
	transaction_time_2 - transaction_time_1 AS time_diff,
	EXTRACT(EPOCH FROM (transaction_time_2::TIMESTAMP - transaction_time_1::TIMESTAMP)) AS time_diff_seconds
FROM users_transactions
)
SELECT 
	u_id AS user_id,
	transaction_id_1 AS first_transaction_id,
	transaction_id_2 AS second_transaction_id,
	transaction_time_1 AS first_transaction_time,
	transaction_time_2 AS second_transaction_time,
	time_diff_seconds
FROM transactions_differences_in_seconds
WHERE time_diff_seconds <= 300 AND time_diff_seconds > 0



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


WITH users_email_domains AS (
	SELECT 
		id,
		SPLIT_PART(email, '@', 2) AS email_domain
	FROM users
	)
SELECT
	email_domain,
	COUNT(*) AS user_count,
	ROUND(COUNT(*)::NUMERIC / (SELECT COUNT(*)::NUMERIC FROM users) * 100, 2) AS percentage
FROM users_email_domains
GROUP BY email_domain
ORDER BY percentage DESC


This was not that difficult yet a GOOD question, as I feel like questions like this may become real in real-life analysis scenarios, and it allows to practice a few important skills + SPLIT_PART is not used that often, and I feel like this is useful to know.


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

WITH users_orders_cnt AS (
	SELECT 
		user_id,
		COUNT(*) AS order_count
	FROM orders
	GROUP BY user_id
	),
users_orders_cnt_comparison AS (
	SELECT 
		user_id,
		order_count,
		(SELECT AVG(order_count) FROM users_orders_cnt) AS avg_order_count,
		order_count - (SELECT AVG(order_count) FROM users_orders_cnt) AS orders_above_avg
	FROM users_orders_cnt
	)
SELECT 
	* 
FROM users_orders_cnt_comparison
WHERE orders_above_avg > 0
ORDER BY order_count DESC

Also a pretty good task
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

### Task Archive: 2025-12-30 (Week 3, Day 5)

# Daily SQL Practice Tasks

**Generated:** 2025-12-30
**Week 3, Day 5 Focus:** Advanced Date Arithmetic, Complex Filtering, Multi-Table Analysis

---

## Task 1: Order Streaks — Users with Consecutive Day Ordering

**Scenario:**
The marketing team wants to identify highly engaged users who made purchases on consecutive days (not just consecutive months, but actual back-to-back days). Find users who have ordered on at least 3 consecutive days at some point in their history.

**Expected Output Columns:**
- `user_id` (integer)
- `streak_start_date` (date) — the first day of the consecutive streak
- `streak_end_date` (date) — the last day of the consecutive streak
- `streak_length` (integer) — number of consecutive days in the streak
- `total_orders_in_streak` (bigint) — total number of orders during the streak period

**Requirements:**
- Use `orders` table
- Identify sequences where a user ordered on consecutive calendar days
- Only include streaks of 3 or more consecutive days
- If a user has multiple streaks, show all of them
- Order by `streak_length` DESC, `user_id` ASC

**Difficulty Rating:** 5/5

WITH users_orders_dates AS (
	SELECT
		user_id,
		created_at,
		DATE(created_at) AS date,
		LAG(DATE(created_at)) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_order_date
	FROM orders
	),
orders_days_diffs AS (
SELECT
	*,
	date - prev_order_date AS days_diff,
	CASE 
	WHEN prev_order_date IS NULL OR date - prev_order_date != 1 THEN 0 ELSE 1
	END AS streak_continuation
FROM users_orders_dates
),
users_streaks AS (
	SELECT *,
	RANK() OVER (PARTITION BY user_id ORDER BY created_at) AS streak
	FROM orders_days_diffs
	WHERE streak_continuation = 1
),
users_streak_dates AS (
	SELECT 
		*,
		streak + 1 AS streak_length,
		CASE
			WHEN streak = 1 THEN prev_order_date ELSE LAG(prev_order_date) OVER (PARTITION BY user_id)
		END AS streak_start_date,
		FIRST_VALUE(date) OVER (PARTITION BY user_id ORDER BY date DESC) AS streak_end_date
	FROM users_streaks
	ORDER BY user_id, date
	),
users_streak_finals AS (
SELECT
	*,
	COALESCE(streak_start_date, LAG(streak_start_date) OVER (PARTITION BY user_id)) AS streak_start
FROM users_streak_dates
),
users_streaks_display AS (
	SELECT
		usf.user_id,
		usf.streak_start AS streak_start_date,
		usf.streak_end_date,
		usf.streak_length,
		MAX(usf.streak_length) OVER (PARTITION BY usf.user_id, usf.streak_end_date) AS total_orders_in_streak
	FROM users_streak_finals usf
)
SELECT 
	*
FROM users_streaks_display
WHERE streak_length > 2
ORDER BY streak_length DESC, user_id


That was truly a hated task for me, I really struggled with it and I don't think I've got it 100% correct IN THE END. It was super difficult and perhaps even maybe a bit retarded, as expecting me to get all of these things correctly with SQL is really difficult, or maybe I don't get how to do it yet. If that's doable, let me know.
---

## Task 2: Product Category Performance Comparison

**Scenario:**
The product team wants to compare category performance. For each product category, show total revenue, average order value, and how it compares to the overall average across all categories.

**Expected Output Columns:**
- `category_name` (varchar)
- `total_revenue` (numeric) — total revenue for this category, rounded to 2 decimals
- `order_count` (bigint) — number of orders containing products from this category
- `avg_order_value` (numeric) — average revenue per order for this category, rounded to 2 decimals
- `overall_avg_order_value` (numeric) — average order value across all categories, rounded to 2 decimals
- `performance_vs_avg` (numeric) — difference between category avg and overall avg, rounded to 2 decimals

**Requirements:**
- Use `products`, `product_categories`, `orders_products` tables
- Calculate revenue as price × quantity
- Calculate average order value per category
- Compare each category's performance to the overall average
- Order by `total_revenue` DESC

**Difficulty Rating:** 4/5

WITH categories_revenues AS (
	SELECT 
		pc."name" AS category_name,
		COUNT(*) AS order_count,
		ROUND(SUM(op.quantity::NUMERIC * p.price::NUMERIC), 2) AS total_revenue,
		ROUND(AVG(op.quantity::NUMERIC * p.price::NUMERIC), 2) AS avg_order_value
	FROM orders_products op
	JOIN products p ON op.product_id = p.id
	JOIN product_categories pc ON p.category_id = pc.id
	GROUP BY pc."name"
	),
overall_avg_order AS (
	SELECT 
		ROUND(AVG(o.amount::NUMERIC), 2) AS overall_avg_order_value
	FROM orders o
	)
SELECT
	*,
	ROUND((cr.avg_order_value - oao.overall_avg_order_value) / oao.overall_avg_order_value  * 100, 2) AS performance_vs_avg
FROM categories_revenues cr
CROSS JOIN overall_avg_order oao
ORDER BY total_revenue DESC


---

## Task 3: User Purchase Recency Analysis

**Scenario:**
The marketing team wants to segment users based on their purchasing behavior. For each user who has made orders, calculate when they last purchased, their total order count, and their lifetime spending value.

**Expected Output Columns:**
- `user_id` (integer)
- `most_recent_order_date` (date) — date of their most recent order
- `days_since_last_order` (integer) — days between their most recent order and the current date
- `total_orders` (bigint) — total number of orders this user has made
- `total_lifetime_value` (numeric) — sum of all order amounts for this user, rounded to 2 decimals
- `avg_order_value` (numeric) — average order amount for this user, rounded to 2 decimals

**Requirements:**
- Use `orders` table
- Calculate days since last order using CURRENT_DATE - most_recent_order_date
- Include all users who have placed at least one order
- Handle NULL amounts appropriately
- Order by `days_since_last_order` ASC (most recent purchasers first)

**Difficulty Rating:** 3/5

WITH users_orders AS (
SELECT *,
FIRST_VALUE(DATE(created_at)) OVER (PARTITION BY user_id ORDER BY created_at DESC) AS most_recent_order_date
FROM orders
)
SELECT 
	user_id,
	EXTRACT('Day' FROM (NOW() - most_recent_order_date)) AS days_since_last_order,
	COUNT(*) AS total_orders,
	ROUND(SUM(amount::NUMERIC), 2) AS total_lifetime_value,
	ROUND(AVG(amount::NUMERIC), 2) AS avg_order_value
FROM users_orders
GROUP BY user_id, most_recent_order_date
ORDER BY days_since_last_order, user_id


---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Date arithmetic and consecutive sequence detection
- Multi-table aggregations and comparisons
- Timestamp differences and time unit conversions
- Grouping and filtering strategies

## Tips

- Consecutive day detection often requires LAG or complex date arithmetic
- Multi-table revenue calculations need careful JOIN conditions
- Time differences: EXTRACT(EPOCH FROM interval) gives seconds, divide by 3600 for hours
- Consider using CTEs to break complex problems into manageable steps

Good luck!
### Task Archive: 2025-12-30 (Week 4, Day 1)

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

SELECT 
	p.name AS product_name,
	pc."name" AS category_name,
	SUM(op.quantity * p.price) AS total_quantity_sold,
	COUNT(*) AS order_count,
	ROUND(AVG(op.quantity), 2) AS avg_quantity_per_order
FROM orders_products op
JOIN products p ON op.product_id = p.id
JOIN product_categories pc ON pc.id = p.category_id
GROUP BY p.name, pc.name
ORDER BY total_quantity_sold DESC

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

WITH users_reg_year_month AS (
	SELECT
		*,
		EXTRACT('Year' FROM created_at) AS year_,
		EXTRACT('Month' FROM created_at) AS month_
	FROM users
	),
users_cohorts AS (
	SELECT
		year_,
		month_,
		COUNT(id) AS total_users_in_cohort
	FROM users_reg_year_month
	GROUP BY year_, month_
	),
active_cohorts AS (
	SELECT
		year_,
		month_,
		COUNT(id) AS active_users_in_cohort
	FROM users_reg_year_month
	WHERE is_active = True
	GROUP BY year_, month_
	)
SELECT
ac.month_ AS cohort_month,
ac.year_ AS cohort_year,
ac.active_users_in_cohort,
uc.total_users_in_cohort,
ROUND(ac.active_users_in_cohort::NUMERIC / uc.total_users_in_cohort::NUMERIC * 100, 2) AS retention_rate_prct
FROM active_cohorts ac
JOIN users_cohorts uc ON ac.year_ = uc.year_ AND ac.month_ = uc.month_
ORDER BY cohort_year DESC, cohort_month DESC


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


WITH transactions_labels AS (
SELECT 
	*,
	CASE WHEN TYPE = 'transfer' THEN 1 ELSE 0 END AS transfer_count,
	CASE WHEN TYPE = 'payment' THEN 1 ELSE 0 END AS payment_count,
	CASE WHEN TYPE = 'deposit' THEN 1 ELSE 0 END AS deposit_count,
	CASE WHEN TYPE = 'withdrawal' THEN 1 ELSE 0 END AS withdrawal_count,
	CASE WHEN TYPE = 'purchase' THEN 1 ELSE 0 END AS purchase_count
FROM transactions
)
SELECT
	user_id,
	COUNT(*) AS total_transactions,
	SUM(transfer_count) AS transfer_count,
	SUM(payment_count) AS payment_count,
	SUM(deposit_count) AS deposit_count,
	SUM(withdrawal_count) AS withdrawal_count,
	SUM(purchase_count) AS purhcase_count
FROM transactions_labels
GROUP BY user_id
ORDER BY total_transactions DESC




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