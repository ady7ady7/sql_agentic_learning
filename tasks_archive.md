
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
### Task Archive: 2025-12-31 (Week 4, Day 2)

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


SELECT 
	id AS user_id,
	first_name,
	last_name,
	email,
	EXTRACT(DAY FROM NOW() - created_at) AS days_since_registration
FROM users
WHERE id NOT IN (SELECT user_id FROM orders)
AND email IS NOT NULL
ORDER BY days_since_registration DESC



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

THERE ARE NO SUCH PRODUCTS!

SELECT id
FROM products
WHERE id NOT IN (SELECT product_id FROM orders_products)

I've checked it and there's no point in going further.
And the task was too easy for that reason.

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


WITH cities_counts AS (
	SELECT 
		city,
		COUNT(*) AS user_count
	FROM users
	WHERE city IS NOT NULL
	GROUP BY city
	),
	cities_counts_avgs AS (
	SELECT 
		*,
		ROUND((SELECT AVG(user_count) FROM cities_counts), 2) AS avg_users_per_city
	FROM cities_counts
	)
SELECT
	DENSE_RANK() OVER (ORDER BY user_count DESC) AS user_count_rank,
	*,
	user_count - avg_users_per_city AS users_above_avg
FROM cities_counts_avgs

I decided to also use DENSE_RANK here in the end to review this window function, but everything else is just as you wanted + DENSE_RANK automatically sorts our rows in desired way without using separate ORDER BY

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

### Task Archive: 2025-12-31 (Week 4, Day 3)

# Daily SQL Practice Tasks

**Generated:** 2025-12-31
**Week 4, Day 3 Focus:** Complex Aggregations, Date Ranges, Business Logic

---

## Task 1: Monthly Active Users (MAU) Trend

**Scenario:**
The growth team wants to track Monthly Active Users over time. For each month in the data, count how many distinct users placed at least one order during that month.

**Expected Output Columns:**
- `year` (integer) — year from order date
- `month` (integer) — month from order date (1-12)
- `monthly_active_users` (bigint) — distinct count of users who placed orders in this month
- `month_over_month_change` (bigint) — change in MAU compared to previous month (can be negative)

**Requirements:**
- Use `orders` table
- Count distinct users per month/year
- Calculate change from previous month
- Order by year ASC, month ASC

**Difficulty Rating:** 4/5

WITH orders_mon_year AS (
	SELECT 
		*,
		EXTRACT('YEAR' FROM created_at) AS year_,
		EXTRACT('Month' FROM created_at) AS month_
	FROM orders
	ORDER BY year_, month_
	),
orders_mau AS (
SELECT
	year_,
	month_,
	COUNT(DISTINCT(user_id)) AS monthly_active_users,
	LAG(COUNT(DISTINCT(user_id))) OVER (ORDER BY year_, month_) AS prev_mau
FROM orders_mon_year
GROUP BY year_, month_
)
SELECT
	year_,
	month_,
	monthly_active_users,
	COALESCE(monthly_active_users - prev_mau, monthly_active_users) AS month_over_month_change
FROM orders_mau


---

## Task 2: High-Value vs Low-Value Customer Segmentation

**Scenario:**
The marketing team wants to segment customers into high-value (total lifetime spending > $1000) and low-value (total lifetime spending <= $1000) groups. Show counts and average metrics for each segment.

**Expected Output Columns:**
- `segment` (text) — 'High-Value' or 'Low-Value'
- `customer_count` (bigint) — number of customers in this segment
- `avg_lifetime_value` (numeric) — average total spending per customer in segment, rounded to 2 decimals
- `avg_order_count` (numeric) — average number of orders per customer in segment, rounded to 2 decimals

**Requirements:**
- Use `orders` table
- Calculate total spending per user
- Segment users based on $1000 threshold
- Aggregate metrics per segment
- Order by segment DESC (High-Value first)

**Difficulty Rating:** 4/5

WITH orders_spendings AS (
	SELECT 
		user_id,
		SUM(amount) AS user_spending,
		COUNT(*) AS order_count
	FROM orders
	GROUP BY user_id
	),
users_orders_segments AS (
	SELECT
		*,
		CASE
			WHEN user_spending >= 1000 THEN 'high-value' ELSE 'low-value'
		END AS segment
	FROM orders_spendings
	)
SELECT
	segment,
	COUNT(user_id) AS customer_count,
	ROUND(AVG(user_spending::NUMERIC), 2) AS avg_lifetime_value,
	ROUND(AVG(order_count), 2) AS avg_order_count
FROM users_orders_segments
GROUP BY segment
ORDER BY segment DESC


---

## Task 3: Products Ordered Together Analysis

**Scenario:**
The product team wants to understand which products are frequently purchased together in the same order. Find the top 10 product pairs that appear together most often.

**Expected Output Columns:**
- `product_1_name` (varchar) — name of first product (alphabetically earlier)
- `product_2_name` (varchar) — name of second product (alphabetically later)
- `times_ordered_together` (bigint) — number of orders containing both products

**Requirements:**
- Use `products`, `orders_products` tables
- Find product pairs that appear in the same order_id
- Avoid duplicates (if A-B exists, don't show B-A)
- Ensure product_1_name comes before product_2_name alphabetically
- Show top 10 pairs by frequency
- Order by `times_ordered_together` DESC

**Difficulty Rating:** 5/5

SELECT 
p1.name AS product_1_name,
p2.name AS product_2_name,
COUNT(*) AS times_ordered_together
FROM orders_products op1
JOIN orders_products op2 ON op1.order_id = op2.order_id
JOIN products p1 ON op1.product_id = p1.id
JOIN products p2 ON op2.product_id = p2.id
WHERE p1.id > p2.id
GROUP BY p1.name, p2.name
ORDER BY times_ordered_together DESC

I'm not sure how to make sure that the product_1_name comes before product_2_name alpabetically, when we already deduplicate them with p1.id > p2.id.

However, the rest seems to be fine.

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- LAG for time-series comparisons
- Conditional logic for segmentation
- Self-joins for finding pairs
- Deduplication strategies

## Tips

- LAG() OVER (ORDER BY year, month) for previous month's value
- CASE WHEN for conditional segmentation
- Self-join on same table with order_id match for pairs
- Use comparison operators to avoid duplicate pairs (e.g., p1.name < p2.name)

Good luck!

---

### Task Archive: 2026-01-08 (Week 4, Day 4)
# Daily SQL Practice Tasks

**Generated:** 2026-01-01
**Week 4, Day 4 Focus:** Date Ranges, Session Analysis, Advanced NULL Handling

---

## Task 1: Users Active in Last 30 Days

**Scenario:**
The engagement team wants to identify recently active users. Find all users who have placed at least one order in the last 30 days from the current date.

**Expected Output Columns:**
- `user_id` (integer)
- `first_name` (varchar)
- `last_name` (varchar)
- `most_recent_order_date` (date) — date of their most recent order
- `days_since_last_order` (integer) — days between most recent order and current date
- `total_orders_last_30_days` (bigint) — count of orders in last 30 days

**Requirements:**
- Use `users` and `orders` tables
- Filter to orders within last 30 days from CURRENT_DATE
- Calculate most recent order date per user
- Count orders in the 30-day window
- Order by `days_since_last_order` ASC

**Difficulty Rating:** 3/5

WITH users_last_order_date AS (
	SELECT
		user_id,
		DATE(MAX(created_at)) AS most_recent_order_date
	FROM orders o
	GROUP BY user_id
	)
	SELECT
		ulo.user_id,
		u.first_name,
		u.last_name,
		most_recent_order_date,
		EXTRACT('Day' FROM NOW() - most_recent_order_date) AS days_since_last_order,
		(SELECT COUNT(*) FROM orders WHERE created_at > (NOW() - INTERVAL '30' DAY)) AS total_orders_last_30_days
	FROM users_last_order_date ulo
	JOIN users u ON ulo.user_id = u.id
	ORDER BY days_since_last_order

While it's an easy task as it is, I've struggled a lot to also calculate and show the total order for the last 30 days. I've discovered pretty early that NONE of the users had orders in the last 30 days, so the task's idea was difficult in that matter.

I wanted however to count specifically that 0 for each user with COALESCE etc., but I entered several issues with that, and as I've started with dates, the user_ids were null if there were no orders, and in the effect I wasn't really able to get sufficient calculation that satisfied me. After nearly 40 minutes of trying, I gave up as I realized that I'm fully aware that there are no orders and perhaps there's no point in further continuation of this particular part.

The final effect is the same as real data.

---

## Task 2: Daily Session Patterns

**Scenario:**
The product team wants to understand session activity patterns. For each date, show total sessions, average sessions per active user, and identify dates with unusually high activity (>10 average sessions per user).

**Expected Output Columns:**
- `date` (date)
- `total_sessions` (numeric) — sum of all count_sessions for this date
- `active_users` (bigint) — count of users who had at least 1 session on this date
- `avg_sessions_per_user` (numeric) — average sessions per active user, rounded to 2 decimals
- `high_activity_day` (boolean) — TRUE if avg_sessions_per_user > 10

**Requirements:**
- Use `user_sessions_daily` table
- Calculate total sessions per date
- Count users who had sessions (count_sessions > 0)
- Calculate average sessions per active user
- Flag high activity days
- Order by `date` DESC

**Difficulty Rating:** 4/5

WITH dates_sessions_users AS (
SELECT 
	d.date,
	COALESCE(SUM(usd.count_sessions), 0) AS total_sessions,
	COUNT(DISTINCT(usd.user_id)) AS active_users
FROM dates d
LEFT JOIN user_sessions_daily usd ON d.date = usd."date"
GROUP BY d.date
ORDER BY d.date
)
SELECT 
	*,
	ROUND(total_sessions::NUMERIC / active_users::NUMERIC, 2) AS avg_sessions_per_user,
	ROUND(total_sessions::NUMERIC / active_users::NUMERIC, 2) > 4 AS high_activity_day
FROM dates_sessions_users
WHERE total_sessions > 0
ORDER BY date DESC

A note - I've decided to set 4 as the threshold for high_Activity_day, as by looking at the data, there weren't any days where avg_sessions_per_user exceeded 5, and 4 seems to be quite a high number.


---

## Task 3: Transaction Amount Outliers

**Scenario:**
The fraud team wants to identify unusually large transactions. Find transactions where the amount is more than 3 times the average transaction amount for that user.

**Expected Output Columns:**
- `transaction_id` (integer)
- `user_id` (integer)
- `amount` (numeric)
- `user_avg_transaction` (numeric) — average transaction amount for this user, rounded to 2 decimals
- `times_above_avg` (numeric) — how many times above their average this transaction is, rounded to 2 decimals

**Requirements:**
- Use `transactions` table
- Calculate average transaction amount per user
- Find transactions > 3x their user's average
- Only include transactions with non-null amounts
- Order by `times_above_avg` DESC

**Difficulty Rating:** 4/5


WITH users_transactions AS (
SELECT 
	id AS transaction_id,
	user_id,
	amount,
	ROUND(AVG(amount) OVER (PARTITION BY user_id), 2) AS avg_transaction_amount
FROM transactions
)
SELECT 
	*,
	amount > avg_transaction_amount AS higher_than_avg_transaction,
	ROUND(amount / avg_transaction_amount, 2) AS times_above_avg
FROM users_transactions
ORDER BY times_above_avg DESC

There were absolutely no transactions above 3 mark, but there were some above the 2 mark.
I've decided to leave them like that iwthout further sorting, as it's perfectly clear and we can see transactions that were deviated the most from the avg_transaction of a particular user.



---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Date range filtering (INTERVAL, date arithmetic)
- Aggregation with conditional logic
- Window functions vs subqueries for averages
- Boolean flag creation

## Tips

- Date filtering: WHERE date >= CURRENT_DATE - INTERVAL '30 days'
- Or: WHERE date >= CURRENT_DATE - 30
- CASE WHEN for boolean flags
- Window functions can calculate user averages: AVG(amount) OVER (PARTITION BY user_id)
- Or use subquery/CTE for user averages

Good luck!

---

### Task Archive: 2026-01-09 (Week 4, Day 5)
# Daily SQL Practice Tasks

**Generated:** 2026-01-08
**Week 4, Day 5 Focus:** Complex Aggregations, Multi-Level Grouping, Advanced Filtering

---

## Task 1: Products with Declining Sales

**Scenario:**
The sales team wants to identify products with declining performance. For each product, calculate revenue for the last 3 months and the 3 months before that, then find products where recent revenue is lower than previous revenue.

**Expected Output Columns:**
- `product_id` (integer)
- `product_name` (varchar)
- `recent_3_month_revenue` (numeric) — total revenue from last 3 complete months
- `previous_3_month_revenue` (numeric) — total revenue from months 4-6 ago
- `revenue_decline` (numeric) — difference (recent - previous), should be negative
- `decline_percentage` (numeric) — percentage decline, rounded to 2 decimals

**Requirements:**
- Use `orders`, `orders_products`, and `products` tables
- Define "last 3 complete months" as the 3 full calendar months before the current month
- Calculate revenue as SUM(quantity * price)
- Only include products where recent_3_month_revenue < previous_3_month_revenue
- Order by `decline_percentage` ASC (most declining first)

**Difficulty Rating:** 5/5

WITH products_orders AS (
	SELECT 
		op.product_id,
		p.name AS product_name,
		o.created_at,
		p.price,
		op.quantity
	FROM orders_products op 
	JOIN orders o ON op.order_id = o.id 
	JOIN products p ON op.product_id = p.id
),
products_last_order_dates AS (
SELECT 
	product_id,
	product_name,
    EXTRACT('Month' FROM MAX(date(created_at))) AS last_order_month,
    EXTRACT('Year' FROM MAX(date(created_at))) AS last_order_year
FROM products_orders
GROUP BY product_id, product_name
),
revenues_recent_3_months AS (
SELECT
	plo.product_id,
	plo.product_name,
	SUM(po.price * po.quantity) AS recent_3_month_revenue
FROM products_last_order_dates plo
JOIN products_orders po ON plo.product_id = po.product_id
WHERE EXTRACT('Month' FROM created_at) >= plo.last_order_month - 3
GROUP BY plo.product_id, plo.product_name
),
revenues_previous_3_months AS (
SELECT
	plo.product_id,
	plo.product_name,
	SUM(po.price * po.quantity) AS previous_3_month_revenue
FROM products_last_order_dates plo
JOIN products_orders po ON plo.product_id = po.product_id
WHERE EXTRACT('Month' FROM created_at) <= plo.last_order_month - 3
AND EXTRACT('Month' FROM created_at) >= plo.last_order_month - 6
GROUP BY plo.product_id, plo.product_name
)
SELECT 
	rp3.product_id,
	rp3.product_name,
	rr3.recent_3_month_revenue,
	rp3.previous_3_month_revenue,
	rr3.recent_3_month_revenue - rp3.previous_3_month_revenue AS revenue_decline,
	ROUND((rr3.recent_3_month_revenue - rp3.previous_3_month_revenue) / rp3.previous_3_month_revenue * 100, 2) AS decline_percentage
FROM revenues_previous_3_months rp3
JOIN revenues_recent_3_months rr3 ON rp3.product_id = rr3.product_id
WHERE rr3.recent_3_month_revenue < rp3.previous_3_month_revenue

That's was quite complex query, not gonna lie.

---

## Task 2: User Engagement Tiers

**Scenario:**
The product team wants to classify users based on multiple engagement metrics. Create engagement tiers combining order frequency, total spend, and recency.

**Expected Output Columns:**
- `user_id` (integer)
- `total_orders` (bigint)
- `total_spent` (numeric)
- `days_since_last_order` (integer)
- `engagement_tier` (text) — classification based on combined criteria
- `tier_rank` (integer) — rank within their tier

**Requirements:**
- Use `users` and `orders` tables
- Calculate total orders, total spending, and days since last order per user
- Classify into tiers:
  - "Champion": 5+ orders AND spent > $500 AND last order within 30 days
  - "Loyal": 5+ orders AND spent > $300
  - "Recent": Last order within 30 days but doesn't meet Champion criteria
  - "At Risk": Last order 31-90 days ago
  - "Churned": Last order > 90 days ago
- Rank users within their tier by total_spent DESC
- Order by engagement_tier, then tier_rank

**Difficulty Rating:** 4/5

WITH users_orders_metrics AS (
	SELECT
		*,
		MAX(created_at) OVER (PARTITION BY user_id) AS last_order_date,
		SUM(amount) OVER (PARTITION BY user_id) AS total_spent,
		COUNT(id) OVER (PARTITION BY user_id) AS total_orders
	FROM orders
	),
users_metrics_final AS (
	SELECT
		DISTINCT user_id,
		total_orders,
		total_spent,
		EXTRACT('Day' FROM NOW() - last_order_date) AS days_since_last_order
	FROM users_orders_metrics
	ORDER BY user_id
	),
users_engagement_tiers AS (
SELECT 
	*,
		CASE
		WHEN total_orders > 5 AND days_since_last_order <= 30 THEN 'champion'
		WHEN total_orders > 5 AND total_spent > 300 THEN 'loyal'
		WHEN days_since_last_order <= 30 THEN 'recent'
		WHEN days_since_last_order > 30 AND days_since_last_order < 90 THEN 'at risk'
		WHEN days_since_last_order > 90 THEN 'churned'
	END AS engagement_tier
FROM users_metrics_final
)
SELECT 
	*,
	RANK() OVER (PARTITION BY engagement_tier ORDER BY total_spent DESC) AS tier_rank
FROM users_engagement_tiers
ORDER BY engagement_tier, tier_rank


---

## Task 3: Category Cross-Sell Analysis

**Scenario:**
The marketing team wants to understand which product categories are frequently purchased together in the same order. Find category pairs that appear together in orders.

**Expected Output Columns:**
- `category_1_name` (varchar) — first category
- `category_2_name` (varchar) — second category
- `orders_together` (bigint) — count of orders containing both categories
- `avg_combined_revenue` (numeric) — average total revenue when both categories in same order, rounded to 2 decimals

**Requirements:**
- Use `orders`, `orders_products`, `products`, and `product_categories` tables
- Find orders containing products from at least 2 different categories
- Calculate how often each category pair appears together
- Calculate average revenue for orders containing both categories
- Avoid duplicate pairs (category A + B should not also appear as B + A)
- Exclude self-pairs (same category appearing twice)
- Only include pairs appearing together in at least 5 orders
- Order by `orders_together` DESC

**Difficulty Rating:** 5/5

WITH categories_orders_count_together AS (
	SELECT
		pc1.name AS category_1_name,
		pc2.name AS category_2_name,
		COUNT(*) AS orders_together
	FROM orders_products op1
	JOIN products p1 ON op1.product_id = p1.id
	JOIN product_categories pc1 ON p1.category_id = pc1.id
	JOIN orders_products op2 ON op1.order_id = op2.order_id
	JOIN products p2 ON op2.product_id = p2.id
	JOIN product_categories pc2 ON p2.category_id = pc2.id
	WHERE pc1.name > pc2.name
	GROUP BY pc1.name, pc2.name
	),
categories_total_revenue AS (
	SELECT
		pc1.name AS category_1_name_x,
		pc2.name AS category_2_name_x,
		SUM(op1.quantity * p1.price + op2.quantity * p2.price) AS total_revenue
	FROM orders_products op1
	JOIN products p1 ON op1.product_id = p1.id
	JOIN product_categories pc1 ON p1.category_id = pc1.id
	JOIN orders_products op2 ON op1.order_id = op2.order_id
	JOIN products p2 ON op2.product_id = p2.id
	JOIN product_categories pc2 ON p2.category_id = pc2.id
	WHERE pc1.name > pc2.name
	GROUP BY pc1.name, pc2.name
	)
SELECT
	cc1.category_1_name,
	cc1.category_2_name,
	cc1.orders_together,
	ROUND(cc2.total_revenue / cc1.orders_together, 2) AS avg_combined_revenue
FROM categories_orders_count_together cc1
JOIN categories_total_revenue cc2 ON cc1.category_1_name = cc2.category_1_name_x AND cc1.category_2_name = cc2.category_2_name_x
ORDER BY cc1.orders_together DESC



---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Complex date range filtering and calculations
- Multi-criteria classification logic
- Self-joins for finding pairs
- Advanced aggregation patterns

## Tips

- Date ranges: Use EXTRACT to identify complete months
- Multi-criteria CASE: Can nest CASE WHEN conditions
- Self-joins: Remember to deduplicate with comparison operators
- Window functions: Can use RANK() OVER (PARTITION BY tier) for tier rankings

Good luck!

---

### Task Archive: 2026-01-10 (Week 5, Day 1)
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

WITH dates_orders AS (
	SELECT 
		d.date,
		COALESCE(COUNT(o.id), 0) AS order_count
	FROM dates d
	LEFT JOIN orders o ON d.date = date(o.created_at)
	GROUP BY d.date
	ORDER BY d.date
	),
dates_cumulative_orders AS (
SELECT
	date,
	order_count,
	SUM(order_count) OVER (ORDER BY date) AS cumulative_orders
FROM dates_orders
)
	SELECT 
		*,
		date - (SELECT date FROM dates_cumulative_orders WHERE cumulative_orders = 1) AS first_order_date
	FROM dates_cumulative_orders




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


WITH users_email_check AS (
	SELECT
	id AS user_id,
	SPLIT_PART(email, '@', 1) AS email_first_part,
	SPLIT_PART(email, '@', 2) AS domain
	FROM users
	)
SELECT
	uec.user_id,
	u.email,
	uec."domain",
	CASE
		WHEN domain IN ('gmail.com', 'yahoo.com', 'hotmail.com', 'onet.pl', 'wp.pl', 'o2.pl', 'gazeta.pl', 'buziaczek.pl', 'protonmail.com', 'onet.eu') THEN 'Free'
		WHEN domain IN ('edu.pl') THEN 'Educational'
		ELSE 'Business'
	END AS domain_category,
	True AS email_valid_format
FROM users_email_check uec
JOIN users u ON uec.user_id = u.id
WHERE uec.domain LIKE ('%.%') AND u.email LIKE ('%@%.%')
ORDER BY domain_category, domain


Definitely not a task that I liked - the thing is we have to state all the domains explicitly here, and it's simply ineffective with CASE WHEN, and as for Educational emails, we'd have to also state all 'edu' domains for all the different countries, if we had people from multiple countries, which is just VERY, VERY BAD, and wildcards don't work in case when as LIKE operator doesn't work there.

So yeah, I fulfilled your requirements and also practiced SPLIT_PART (Which is actually cool, and I definitely want to practice more operations on text), but I didn't like the task.


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


WITH users_transactions AS (
	SELECT 
		user_id,
		id AS transaction_id,
		created_at,
		LAG(created_at) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_transaction_time
	FROM transactions
	),
users_transaction_gaps AS (
SELECT 
	*,
	EXTRACT('Minute' FROM created_at - prev_transaction_time) AS minutes_since_prev
FROM users_transactions
WHERE prev_transaction_time IS NOT NULL
),
users_avg_gaps AS (
SELECT 
	*,
	ROUND(AVG(minutes_since_prev) OVER (PARTITION BY user_id), 2) AS user_avg_gap
FROM users_transaction_gaps
)
SELECT 
	user_id,
	transaction_id,
	created_at,
	prev_transaction_time,
	minutes_since_prev,
	user_avg_gap,
	ROUND(ABS(minutes_since_prev - user_avg_gap), 2) AS deviation_from_avg,
	ABS(minutes_since_prev - user_avg_gap) > user_avg_gap * 2 AS unusual_pattern
FROM users_avg_gaps


A few things:

- Please note that the user_id, order_date is KEPT and maintained early on with the first window function
- I'VE DELIBERATELY used created_at and prev_transaction_time and minutes_since_prev, AFTER I SAW HOW DATA ACTUALLY LOOKS LIKE. There were absolutely no gaps that span across days, ONLY gaps that were measured in minutes, so it was the ONLY option that made sense. It was a purely data driven decision and although it's a bit different than your requirement, I DEMAND YOU SHALL NOT TAKE AWAY POINTS FROM me for this, as I've simply adapted the requirements based on the actual data. It works, it's clean and understandable for everyone.


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

---

### Task Archive: 2026-01-11 (Week 5, Day 2)
# Daily SQL Practice Tasks

**Generated:** 2026-01-10
**Week 5, Day 2 Focus:** JSON Operations, Advanced Window Frames, Complex Subqueries

---

## Task 1: First and Last Value in Same Row

**Scenario:**
The analytics team wants to see each user's transaction history with both their first and most recent transaction details in the same row for comparison.

**Expected Output Columns:**
- `user_id` (integer)
- `current_transaction_id` (integer)
- `current_amount` (numeric)
- `current_date` (date)
- `first_transaction_amount` (numeric) — amount of user's very first transaction
- `first_transaction_date` (date) — date of user's very first transaction
- `last_transaction_amount` (numeric) — amount of user's most recent transaction
- `last_transaction_date` (date) — date of user's most recent transaction

**Requirements:**
- Use `transactions` table
- Use FIRST_VALUE and LAST_VALUE window functions with proper frames
- Or use FIRST_VALUE with reversed ORDER BY for last values
- Include all transactions (every row should show first/last context)
- Order by user_id, current_date

**Difficulty Rating:** 3/5
SELECT 
	user_id,
	id AS current_order_id,
	created_at AS current_date,
	amount AS current_amount,
	FIRST_VALUE(id) OVER (PARTITION BY user_id ORDER BY created_at DESC) AS last_transaction_id,
	FIRST_VALUE(amount) OVER (PARTITION BY user_id ORDER BY created_at DESC) AS last_transaction_amount,
	FIRST_VALUE(created_at) OVER (PARTITION BY user_id ORDER BY created_at DESC) AS last_transaction_date,
	FIRST_VALUE(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS first_transaction_amount,
	FIRST_VALUE(created_at) OVER (PARTITION BY user_id ORDER BY created_at) AS first_transaction_date
FROM transactions


This task's requirements are very weird and unclear as for expected columns.
What do you mean by 'current' in this context?

I only sense it makes sense to get the first/last transaction and that last could be interpreted as 'current', but then why would you distinguish last and current? I get that we might also want to get the current date for whatever reason, but it doesn't make sense.



---

## Task 2: Running Total with Reset

**Scenario:**
The finance team wants to see daily order amounts with a running total that resets each month. Calculate cumulative revenue within each month.

**Expected Output Columns:**
- `order_date` (date)
- `daily_revenue` (numeric) — total revenue for this date
- `month` (integer) — month number
- `year` (integer) — year number
- `monthly_running_total` (numeric) — cumulative sum within the month, reset at each new month

**Requirements:**
- Use `orders` table
- Extract year and month from order dates
- Calculate daily revenue sum
- Use window function with PARTITION BY year, month for monthly reset
- Order by year, month, order_date

**Difficulty Rating:** 3/5

SELECT 
	d."date",
	EXTRACT('Month' FROM d.date) AS month_,
	EXTRACT('Year' FROM d.date) AS year_,
	SUM(amount) OVER (PARTITION BY d.date) AS daily_revenue,
	SUM(amount) OVER (PARTITION BY EXTRACT('Month' FROM d.date) ORDER BY d.date) AS monthly_running_total
FROM orders o 
JOIN dates d ON date(o.created_at) = d."date" 

---

## Task 3: Comparing Current Row to Next Row

**Scenario:**
The operations team wants to identify order amount increases. For each order, show the next order amount for that user to identify when spending increased.

**Expected Output Columns:**
- `order_id` (integer)
- `user_id` (integer)
- `amount` (numeric)
- `order_date` (date)
- `next_order_amount` (numeric) — amount of the next order for this user
- `next_order_date` (date) — date of the next order for this user
- `amount_increase` (numeric) — difference (next_order_amount - amount)
- `spending_increased` (boolean) — TRUE if next order amount is higher

**Requirements:**
- Use `orders` table
- Use LEAD window function to get next order details
- Calculate amount difference
- Include all orders (last order per user will have NULL next values)
- Order by user_id, order_date

**Difficulty Rating:** 3/5

WITH users_orders_next AS (
SELECT 
	id AS order_id,
	user_id,
	amount,
	created_at AS order_time,
	LEAD(created_at) OVER (PARTITION BY user_id ORDER BY created_at) AS next_order_date,
	LEAD(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS next_order_amount,
	(LEAD(amount) OVER (PARTITION BY user_id ORDER BY created_at)) - amount AS amount_increase
FROM orders
)
SELECT 
	*,
	next_order_amount > amount AS spending_increased
FROM users_orders_next
WHERE next_order_date IS NOT NULL

The ordering is also set as per your instructions, so everything works properly

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Window function frames (ROWS vs RANGE)
- PARTITION BY for grouping within window functions
- LEAD/LAG for row-to-row comparisons
- Proper NULL handling

## Tips

- FIRST_VALUE: `FIRST_VALUE(column) OVER (PARTITION BY group ORDER BY sort_col)`
- LAST_VALUE needs explicit frame: `LAST_VALUE(column) OVER (... ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)`
- Or use FIRST_VALUE with reversed ORDER BY: `FIRST_VALUE(column) OVER (PARTITION BY group ORDER BY sort_col DESC)`
- Running totals: `SUM(column) OVER (PARTITION BY group ORDER BY sort_col)`
- LEAD: `LEAD(column, 1) OVER (PARTITION BY group ORDER BY sort_col)` gets next row

Good luck!

---

### Task Archive: 2026-01-11 (Week 5, Day 3)

**Focus:** RANK vs ROW_NUMBER, Conditional Aggregation, Subquery Performance

## Task 1: Dense Ranking with Ties

**Scenario:**
The sales team wants to rank products by revenue, but when multiple products have the same revenue, they should share the same rank (with no gaps in the sequence).

**Expected Output Columns:**
- `product_id` (integer)
- `product_name` (varchar)
- `total_revenue` (numeric) — sum of all revenue for this product
- `revenue_rank` (bigint) — dense ranking (1,2,2,3 not 1,2,2,4)

**Requirements:**
- Use `orders_products` and `products` tables
- Calculate total revenue per product (quantity * price)
- Use DENSE_RANK window function
- Order by revenue_rank ASC

**Difficulty Rating:** 2/5

**Student Solution:**
```sql
WITH products_revenues AS (
SELECT 
	op.product_id,
	p.name AS product_name,
	SUM(op.quantity * p.price) AS total_revenue
FROM orders_products op 
JOIN products p ON op.product_id = p.id
GROUP BY op.product_id, p.name
)
SELECT 
	*,
	DENSE_RANK() OVER (ORDER BY total_revenue DESC)
FROM products_revenues
```

**Score: 10/10** - Perfect implementation of DENSE_RANK with clean CTE structure.

---

## Task 2: Conditional Aggregation with Multiple Conditions

**Scenario:**
The finance team wants a single-row summary showing order counts and totals broken down by different criteria.

**Expected Output Columns:**
- `total_orders` (bigint) — count of all orders
- `high_value_orders` (bigint) — count where amount > 100
- `high_value_revenue` (numeric) — sum of amounts where amount > 100
- `low_value_orders` (bigint) — count where amount <= 100
- `low_value_revenue` (numeric) — sum of amounts where amount <= 100
- `avg_order_amount` (numeric) — average of all order amounts, rounded to 2 decimals

**Requirements:**
- Use `orders` table
- Return a single row with all aggregations
- Use CASE WHEN or FILTER clause for conditional counts/sums
- Round averages to 2 decimals

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
SELECT 
	(SELECT COUNT(*) FROM orders) AS total_orders,
	(SELECT COUNT(*) FROM orders WHERE amount > 100) AS high_value_orders,
	(SELECT ROUND(SUM(amount::NUMERIC), 2) FROM orders WHERE amount > 100) AS high_value_revenue,
	(SELECT COUNT(*) FROM orders WHERE amount <= 100) AS low_value_orders,
	(SELECT ROUND(SUM(amount::NUMERIC), 2) FROM orders WHERE amount <= 100) AS low_value_revenue,
	(SELECT ROUND(AVG(amount::NUMERIC), 2) FROM orders) AS avg_order_amount
FROM orders
```

**Student Note:** "I used subqueries today and I thought it's a good way to solve this exercise with one single CTE. It also works perfectly here."

**Score: 8/10** - Subqueries work but create duplicate rows (one row per order with same values). The FILTER/CASE WHEN pattern is more efficient and produces exactly one row.

---

## Task 3: Correlated Subquery - Users Above Category Average

**Scenario:**
Find users whose total spending in each product category exceeds the average spending in that category across all users.

**Expected Output Columns:**
- `user_id` (integer)
- `category_name` (varchar)
- `user_category_spending` (numeric) — total spent by this user in this category
- `category_avg_spending` (numeric) — average spent per user in this category, rounded to 2 decimals
- `amount_above_avg` (numeric) — difference between user spending and average

**Requirements:**
- Use `orders`, `orders_products`, `products`, and `product_categories` tables
- Calculate user spending per category
- Calculate average spending per category (across all users)
- Only show users exceeding the category average
- Order by category_name, amount_above_avg DESC

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH users_category_spendings AS (
SELECT 
	o.user_id,
	pc.name AS category_name,
	SUM(op.quantity * p.price) AS user_category_spending
FROM orders_products op
JOIN orders o ON op.order_id = o.id
JOIN products p ON op.product_id = p.id 
JOIN product_categories pc ON pc.id = p.category_id
GROUP BY o.user_id, pc.name
),
categories_avgs AS (
SELECT
	pc.name AS category_name,
	ROUND(AVG(op.quantity * p.price), 2) AS category_avg_spending
FROM orders_products op 
JOIN orders o ON op.order_id = o.id
JOIN products p ON op.product_id = p.id 
JOIN product_categories pc ON pc.id = p.category_id
GROUP BY pc.name
)
SELECT
	ucs.user_id,
	ucs.category_name,
	ucs.user_category_spending,
	ca.category_avg_spending,
	ucs.user_category_spending - ca.category_avg_spending AS amount_above_avg
FROM users_category_spendings ucs
JOIN categories_avgs ca ON ucs.category_name = ca.category_name
WHERE ucs.user_category_spending > ca.category_avg_spending 
ORDER BY category_name, amount_above_avg DESC
```

**Score: 9/10** - Excellent CTE decomposition. Minor semantic difference: categories_avgs calculates average per line item rather than average per user, but overall approach is sound.

---

**Day 3 Overall Score: 9/10**


---

### Task Archive: 2026-01-14 (Week 5, Day 4)

**Focus:** Advanced Window Frames, Percentile Rankings, Gap Analysis

## Task 1: Rolling Window with Custom Frame

**Scenario:**
The analytics team wants to compare each user's daily session count to their own 3-day moving average (current day plus the 2 previous days). Flag days where the user's session count is more than 50% above their moving average as "spike" days.

**Expected Output Columns:**
- `user_id` (integer)
- `date` (date)
- `count_sessions` (integer)
- `moving_avg_3day` (numeric) — 3-day moving average rounded to 2 decimals
- `is_spike` (boolean) — TRUE if count_sessions > moving_avg_3day * 1.5

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH users_session_avgs AS (
	SELECT 
		date,
		user_id,
		count_sessions,
		ROUND(AVG(count_sessions) OVER (PARTITION BY user_id ORDER BY date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS moving_avg_3d,
		RANK() OVER (PARTITION BY user_id ORDER BY date) AS day_count
		FROM user_sessions_daily usd 
		ORDER BY user_id
	)
SELECT 
	*,
	count_sessions > moving_avg_3d * 1.5 AS is_spike
FROM users_session_avgs
WHERE day_count > 2
```

**Student Note:** Discovered data sparsity (sessions not consecutive), adapted to use last 3 session records per user rather than consecutive calendar days.

**Score: 9/10** - Correct ROWS BETWEEN frame, smart incomplete window filtering.

---

## Task 2: Percentile Ranking with Category Context

**Scenario:**
For each product, calculate its revenue percentile rank within its category and across all products. Identify products that are top performers in their category (top 25%) but underperformers overall (bottom 50%).

**Expected Output Columns:**
- `product_id` (integer)
- `category_name` (varchar)
- `total_revenue` (numeric)
- `category_percentile` (numeric)
- `global_percentile` (numeric)
- `category_star_global_underperformer` (boolean)

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH products_revenues AS (
SELECT 
	op.product_id,
	p."name" AS product_name,
	pc."name" AS category_name,
	SUM(p.price * op.quantity) AS product_revenue
FROM orders_products op
JOIN products p ON op.product_id = p.id
JOIN product_categories pc ON p.category_id = pc.id
GROUP BY op.product_id, p.name, pc."name"
ORDER BY product_revenue DESC
),
products_revenues_ranks AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY category_name ORDER BY product_revenue DESC) AS category_revenue_rank,
	ROUND(PERCENT_RANK() OVER (PARTITION BY category_name ORDER BY product_revenue)::NUMERIC, 2) AS category_percentile_rank,
	ROUND(PERCENT_RANK() OVER (ORDER BY product_revenue)::NUMERIC, 2) AS overall_percentile_rank,
	RANK() OVER (ORDER BY product_revenue DESC) AS overall_revenue_rank
FROM products_revenues
ORDER BY overall_revenue_rank, category_revenue_rank
)
SELECT 
	*,
	CASE WHEN category_percentile_rank >= 0.75 AND overall_percentile_rank < 0.50 THEN TRUE ELSE FALSE
	END AS category_star_global_underperformer
FROM products_revenues_ranks
```

**Student Note:** First time using PERCENT_RANK — loved learning it. Added extra RANK() columns for practice.

**Score: 10/10** - Perfect dual PERCENT_RANK implementation with different partitions.

---

## Task 3: Gap and Island Detection - User Order Gaps

**Scenario:**
Identify significant gaps in user ordering behavior. For each user, find periods where they went more than 30 days between consecutive orders.

**Expected Output Columns:**
- `user_id` (integer)
- `previous_order_date` (timestamp)
- `next_order_date` (timestamp)
- `gap_days` (integer)
- `user_avg_order_amount` (numeric)
- `estimated_missed_orders` (numeric)
- `potential_lost_revenue` (numeric)

**Difficulty Rating:** 5/5

**Student Solution:**
```sql
WITH users_orders AS (
	SELECT 
		*,
		LAG(created_at) OVER (PARTITION BY user_id ORDER BY created_at) AS previous_order_time,
		LEAD(created_at) OVER (PARTITION BY user_id ORDER BY created_at) AS next_order_time,
		AVG(amount) OVER (PARTITION BY user_id) AS user_avg_order_amount,
		COUNT(id) OVER (PARTITION BY user_id) AS user_orders_count
	FROM orders
	),
users_gaps AS (
	SELECT 
		user_id,
		created_at,
		COALESCE(DATE(created_at) - date(previous_order_time), 0) AS gap_days
	FROM users_orders
	WHERE user_orders_count > 2
	),
users_orders_avg_gaps_amounts AS (
	SELECT 
		*,
		uo.user_id AS id_user,
		AVG(ug.gap_days) OVER (PARTITION BY ug.user_id) AS user_avg_gap_days
	FROM users_orders uo
	JOIN users_gaps ug ON uo.user_id = ug.user_id AND uo.created_at = ug.created_at
	),
users_missed_orders AS (
	SELECT 
		id_user,
		previous_order_time,
		next_order_time,
		gap_days,
		ROUND(user_avg_order_amount::NUMERIC, 2) AS user_avg_order_amount,
		CASE WHEN gap_days != 0 THEN ROUND(gap_days / user_avg_gap_days, 2) END AS estimated_missed_orders
	FROM users_orders_avg_gaps_amounts
	WHERE gap_days != 0
	)
SELECT 
	*,
	ROUND(estimated_missed_orders * user_avg_order_amount, 2) AS potential_lost_revenue
FROM users_missed_orders
ORDER BY potential_lost_revenue DESC
```

**Student Note:** Complex but satisfying task. Wants more like this.

**Score: 9/10** - Strong multi-CTE decomposition, correct LAG usage, opportunity cost calculation.

---

**Day 4 Overall Score: 9.3/10**


---

### Task Archive: 2026-01-15 (Week 5, Day 5)

**Focus:** Cumulative Distribution, Running Comparisons, Multi-Partition Analytics

## Task 1: NTILE vs PERCENT_RANK — Customer Spending Quartiles

**Scenario:**
The marketing team wants to segment customers into spending quartiles for targeted campaigns.

**Expected Output Columns:**
- `user_id` (integer)
- `total_spending` (numeric)
- `spending_quartile` (integer) — NTILE(4) bucket
- `spending_percentile` (numeric) — PERCENT_RANK (0 to 1)
- `quartile_label` (text) — Bronze/Silver/Gold/Platinum

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH users_orders AS (
	SELECT 
		user_id,
		SUM(amount) AS total_spending,
		COUNT(*) AS orders_total
	FROM orders
	GROUP BY user_id
	),
orders_quartiles AS (
	SELECT 
		*,
		NTILE(4) OVER (ORDER BY total_spending) AS spending_quartile,
		ROUND(PERCENT_RANK() OVER (ORDER BY total_spending)::NUMERIC, 2) AS spending_percentile,
		NTILE(4) OVER (ORDER BY orders_total) AS orders_quartile
	FROM users_orders
	)
SELECT 
	*,
	CASE 
		WHEN spending_quartile = 1 THEN 'Bronze'
		WHEN spending_quartile = 2 THEN 'Silver'
		WHEN spending_quartile = 3 THEN 'Gold'
		WHEN spending_quartile = 4 THEN 'Platinum'
	END AS spending_quartile_label
FROM orders_quartiles
ORDER BY total_spending DESC
```

**Student Note:** Added orders_quartile for extra analysis.

**Score: 10/10** - Perfect NTILE and PERCENT_RANK implementation with bonus analysis.

---

## Task 2: Running Difference — Month-over-Month Revenue Change

**Scenario:**
Finance needs a monthly revenue report showing totals and change from previous month.

**Expected Output Columns:**
- `month` (date)
- `monthly_revenue` (numeric)
- `previous_month_revenue` (numeric)
- `revenue_change` (numeric)
- `change_percent` (numeric)
- `trend` (text) — UP/DOWN/FLAT

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH orders_y_m AS (
SELECT 
	*,
	EXTRACT('Year' FROM created_at) AS year_,
	EXTRACT('Month' FROM created_at) AS month_
FROM orders
),
monthly_revenues AS (
SELECT 
	year_, 
	month_,
	SUM(amount) AS monthly_revenue
FROM orders_y_m
GROUP BY year_, month_
ORDER BY year_, month_
),
monthly_revs_complete AS (
SELECT 
	*,
	COALESCE(LAG(monthly_revenue) OVER (ORDER BY year_, month_), 0) AS previous_month_revenue
FROM monthly_revenues
)
SELECT
	*,
	CASE
		WHEN previous_month_revenue != 0 THEN ROUND(monthly_revenue::NUMERIC - previous_month_revenue::NUMERIC, 2) ELSE 0
	END AS revenue_change,
	CASE 
		WHEN previous_month_revenue != 0 THEN ROUND(monthly_revenue::NUMERIC / previous_month_revenue::NUMERIC * 100, 1) ELSE 0
	END AS change_percent,
	CASE 
		WHEN monthly_revenue - previous_month_revenue = monthly_revenue THEN 'FLAT'
		WHEN monthly_revenue > previous_month_revenue THEN 'UP'
		WHEN monthly_revenue < previous_month_revenue THEN 'DOWN' 
	END AS trend
FROM monthly_revs_complete
```

**Student Note:** Used EXTRACT instead of DATE_TRUNC. Used COALESCE with 0 for practical calculations.

**Score: 9/10** - Correct LAG implementation. EXTRACT works, DATE_TRUNC is more concise.

---

## Task 3: Multi-Level Ranking — Product Performance

**Scenario:**
Identify products that rank top 3 in category but outside top 10 overall.

**Expected Output Columns:**
- `product_id` (integer)
- `product_name` (varchar)
- `category_name` (varchar)
- `units_sold` (bigint)
- `category_rank` (bigint)
- `overall_rank` (bigint)
- `is_category_champion_needs_boost` (boolean)

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH products_units_sold AS (
SELECT 
	p.id AS product_id,
	p.name AS product_name,
	pc.name AS category_name,
	SUM(op.quantity) AS units_sold
FROM orders_products op
JOIN products p ON op.product_id = p.id
JOIN product_categories pc ON pc.id = p.category_id
GROUP BY p.id, p.name, pc.name
),
products_ranks AS (
SELECT 
	product_id,
	product_name,
	category_name,
	units_sold,
	RANK() OVER (ORDER BY units_sold DESC) AS overall_rank,
	RANK() OVER (PARTITION BY category_name ORDER BY units_sold DESC) AS category_rank
FROM products_units_sold
)
SELECT 
	*,
	CASE 
		WHEN category_rank <= 3 AND overall_rank > 10 THEN TRUE ELSE FALSE
	END AS is_category_champion_needs_boost
FROM products_ranks
ORDER BY category_name, category_rank
```

**Score: 10/10** - Perfect dual-partition RANK implementation.

---

**Day 5 Overall Score: 9.7/10**

**Week 5 Complete - Average: 9.2/10**


---

### Task Archive: 2026-01-16 (Week 6, Day 1)

**Focus:** Recursive CTEs, Hierarchical Data, Date Series Generation

## Task 1: Recursive CTE — Generate Date Series

**Scenario:**
Generate a complete date series for all months in 2025, then LEFT JOIN to show monthly order counts.

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH months_trunc AS (
SELECT
	DISTINCT DATE_TRUNC('Month', date) AS month_date
FROM dates d 
WHERE EXTRACT('Year' FROM date) = 2025
)
SELECT 
	mt.month_date AS month_start,
	COALESCE(COUNT(o.id), 0) AS order_count,
	COALESCE(SUM(o.amount), 0) AS total_revenue
FROM months_trunc mt
LEFT JOIN orders o ON mt.month_date = DATE_TRUNC('Month', o.created_at)
GROUP BY mt.month_date
```

**Student Note:** Used dates table instead of recursive CTE. "Why would I use RECURSIVE?"

**Score: 8/10** - Correct output but missed recursive CTE requirement. Practical solution but learning objective bypassed.

---

## Task 2: Self-Referential Hierarchy — User Referral Chain

**Scenario:**
Find users whose first transaction was 1-7 days before another user (potential referrer).

**Difficulty Rating:** 5/5

**Student Solution:**
```sql
WITH users_first_transactions AS (
SELECT 
	DISTINCT user_id,
	FIRST_VALUE(created_at) OVER (PARTITION BY user_id) AS first_transaction_date
FROM transactions
),
referral_windows AS (
SELECT 
	*,
	first_transaction_date - INTERVAL '7' Day AS possibly_referred
FROM users_first_transactions
),
potential_referral_pairs AS (
SELECT 
	DISTINCT rw1.user_id AS uid1,
	rw2.user_id AS uid2,
	rw1.first_transaction_date AS first_transaction_1,
	rw2.first_transaction_date AS first_transaction_2,
	rw2.possibly_referred AS back_window2
FROM referral_windows rw1
JOIN referral_windows rw2 ON rw1.first_transaction_date > rw2.possibly_referred
WHERE rw1.user_id != rw2.user_id
AND rw1.first_transaction_date >= rw2.first_transaction_date
AND rw1.first_transaction_date <= rw2.first_transaction_date + INTERVAL '7' DAY
ORDER BY rw1.user_id
),
closest_referred AS (
SELECT
	uid1 AS user_id,
	MIN(first_transaction_2) AS referred_first_transaction
FROM potential_referral_pairs
GROUP BY uid1
)
SELECT
	cr.user_id,
	pfp.first_transaction_1 AS first_transaction_date,
	pfp.uid2 AS potentially_referred_id,
	pfp.first_transaction_2 AS referred_first_transaction,
	EXTRACT('Days' FROM pfp.first_transaction_1 - pfp.first_transaction_2) AS days_apart
FROM closest_referred cr
JOIN potential_referral_pairs pfp ON cr.user_id = pfp.uid1 AND cr.referred_first_transaction = pfp.first_transaction_2
ORDER BY first_transaction_date
```

**Student Note:** Complex task, inverted relationship direction (shows who user referred rather than who referred them). "Pain in the ass, need a breather tomorrow."

**Score: 9/10** - Excellent self-join logic on genuinely difficult problem.

---

## Task 3: Running Balance Simulation

**Scenario:**
Simulate running account balance for user_id = 1, starting at 1000.

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH first_transaction AS (
SELECT 
	*,
	FIRST_VALUE(created_at) OVER (PARTITION BY user_id ORDER BY t.created_at) AS first_transaction_time
FROM transactions t
WHERE user_id = 1
GROUP BY t.user_id, t.id
),
starting_balance AS (
SELECT *,
	CASE WHEN created_at = first_transaction_time THEN 1000 
	END AS running_balance
FROM first_transaction
),
cash_flows AS (
SELECT 
	*,
	CASE WHEN TYPE IN ('withdrawal', 'transfer', 'purchase') THEN COALESCE(running_balance, 0) - amount ELSE COALESCE(running_balance, 0) + amount
	END AS cashflow
FROM starting_balance
)
SELECT 
	id AS transaction_id,
	created_at,
	TYPE,
	amount,
	cashflow AS balance_change,
	SUM(cashflow) OVER (ORDER BY created_at) AS running_balance
FROM cash_flows
```

**Score: 9/10** - Correct running total with conditional logic. Slightly over-engineered but works.

---

**Day 1 Overall Score: 8.7/10**

**Action Items:**
- Week 6 to focus on recursive CTEs with scaffolded learning
- Lighter Monday sessions (1 hard + 2 moderate)
- Include rationale and examples for recursive CTE concept


---

### Task Archive: 2026-01-17 (Week 6, Day 2)

**Focus:** Recursive CTEs — Foundations & Building Blocks

## Task 1: Generate a Number Sequence (Warm-up)

**Scenario:**
Generate numbers from 1 to 10 using a recursive CTE.

**Difficulty Rating:** 2/5

**Student Solution:**
```sql
WITH RECURSIVE counter AS (
	SELECT 1 AS n
	UNION ALL
	SELECT n + 1
	FROM counter
	WHERE n < 11
)
SELECT * FROM counter
```

**Score: 10/10** - Perfect pattern execution.

---

## Task 2: Generate a Date Series (Apply the Pattern)

**Scenario:**
Generate all dates in January 2025 using a recursive CTE.

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH RECURSIVE january_dates AS (
	SELECT DATE_TRUNC('Day', '2025-01-01'::DATE) AS DAY
	UNION ALL
	SELECT day + INTERVAL '1' Day
	FROM january_dates
	WHERE day < '2025-01-31'::DATE
)
SELECT * FROM january_dates
```

**Score: 10/10** - Clean application of pattern to dates.

---

## Task 3: Monthly Revenue Report with Generated Dates

**Scenario:**
Generate all 12 months of 2025 using recursive CTE, LEFT JOIN to orders for revenue.

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH RECURSIVE months_2025 AS (
SELECT DATE_TRUNC('Month', '2025-01-01'::DATE) AS month_
UNION ALL
SELECT month_ + INTERVAL '1' MONTH
FROM months_2025
WHERE month_ < '2025-12-1'::DATE
)
SELECT 
	m_2025.month_ AS month_start,
	COUNT(o.id) AS order_count,
	COALESCE(ROUND(SUM(o.amount)::NUMERIC, 2), 0) AS total_revenue
FROM months_2025 m_2025
LEFT JOIN orders o ON m_2025.month_ = DATE_TRUNC('Month', o.created_at) 
GROUP BY m_2025.month_
```

**Score: 10/10** - Exactly the correct approach for Day 1's task. Full understanding demonstrated.

---

**Day 2 Overall Score: 10/10**

**Notes:** Scaffolded approach highly effective. Completed in 30 minutes including notes.

