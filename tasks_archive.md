
### Task Archive: 2026-06-05 (Week 24, Day 4) — Score: 18/20

**Task 1 (RTH-RANGE-002):** DISTINCT ON fix + weekday summary. 9/10 — correct pattern, NULLIF denominator clean.
**Task 2 (RTH-CLOSE-001):** Day-over-day close change by weekday. 9/10 — LAG correct, FILTER syntax clean, minor: >= 0 includes flat days.

---

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


---

### Task Archive: 2026-01-18 (Week 6, Day 3)

**Focus:** Recursive CTEs — Reinforcing the Basics

## Task 1: Generate Even Numbers

**Scenario:**
Generate all even numbers from 2 to 20.

**Difficulty Rating:** 2/5

**Student Solution:**
```sql
WITH RECURSIVE even_number AS (
	SELECT 2 AS n
	UNION ALL
	SELECT n + 2
	FROM even_number
	WHERE n < 20
)
SELECT * FROM even_number
```

**Score: 10/10** - Perfect pattern execution.

---

## Task 2: Generate Week Dates with Day Names

**Scenario:**
Generate all 7 days of a specific week with weekday names.

**Difficulty Rating:** 2/5 (Student felt: 4/5)

**Student Solution:**
```sql
WITH RECURSIVE dates AS (
SELECT 
	'2025-01-06'::DATE AS day_date,
	TRIM(TO_CHAR('2025-01-06'::DATE, 'Day')) AS date_name
	UNION ALL 
	SELECT (day_date::DATE + INTERVAL '1' DAY)::DATE,
	TRIM(TO_CHAR(day_date + 1, 'Day')) AS date_name
	FROM dates
	WHERE day_date::DATE < '2025-01-12'::DATE
)
SELECT * FROM dates
```

**Student Note:** "It wasn't a 2/5 difficulty task for me, but rather 4/5 - I need to practice this MORE and MORE"

**Score: 9/10** - Correct output. Type casting was challenging.

---

## Task 3: Quarterly Revenue Report

**Scenario:**
Generate 4 quarters of 2025 using recursive CTE, LEFT JOIN to orders for revenue.

**Difficulty Rating:** 3/5 (Student felt: 5/5)

**Student Solution:**
```sql
WITH RECURSIVE quarters AS (
SELECT
	'2025-01-01'::DATE AS quarter_start
	UNION ALL
	SELECT (quarter_start + INTERVAL '3' MONTH)::DATE
	FROM quarters
	WHERE quarter_start < '2025-10-01'
)
SELECT 
	q.quarter_start,
	'Q' || EXTRACT(QUARTER FROM q.quarter_start) AS quarter_label,
	COUNT(o.id) AS orders_count,
	COALESCE(SUM(o.amount), 0) AS total_revenue
FROM quarters q
LEFT JOIN orders o ON o.created_at >= q.quarter_start AND o.created_at < q.quarter_start + INTERVAL '3' MONTH
GROUP BY q.quarter_start
ORDER BY q.quarter_start
```

**Student Note:** "I needed your help for this... difficult elements: 1. Setting the quarter starts properly 2. Setting the quarter label 3. Figuring out the logic for actually joining the orders."

**Score: 10/10** - Needed help but understood and implemented correctly.

---

**Day 3 Overall Score: 9.7/10**

**Session Notes:** 
- Original tasks (hierarchical CTEs) were too complex — regenerated simpler ones
- Student requested continued scaffolded approach with smaller steps


---

### Task Archive: 2026-01-19 (Week 6, Day 4)

**Focus:** Recursive CTEs — Applying the Pattern to Real Scenarios

## Task 1: Generate Bi-Weekly Pay Dates for 2025

**Scenario:**
Generate all bi-weekly pay dates for 2025 starting January 10th.

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH RECURSIVE pay_dates AS (
SELECT 
	('2025-01-10')::DATE AS pay_date,
	1 AS pay_period_number
UNION ALL
SELECT 
	(pay_date + INTERVAL '14' Day)::DATE,
	pay_period_number + 1
FROM pay_dates
WHERE pay_date < ('2025-12-26')::DATE
)
SELECT * FROM pay_dates
```

**Score: 10/10** - Perfect execution with two columns tracked.

---

## Task 2: Hourly Slots for a Business Day

**Scenario:**
Generate hourly booking slots for 9 AM to 5 PM with labels.

**Difficulty Rating:** 3/5 (Student felt: 5/5)

**Student Solution:**
```sql
WITH RECURSIVE time_slots AS (
SELECT 
	TIME '09:00:00' AS slot_start,
	TIME '10:00:00' AS slot_end,
	'9 AM' AS slot_label
UNION ALL
SELECT 
	slot_start + INTERVAL '1' HOUR,
	slot_end + INTERVAL '1' HOUR,
	((split_part(slot_label, ' ', 1))::INTEGER + 1)::TEXT || CASE WHEN EXTRACT(HOUR FROM slot_start) < 12 THEN ' AM' ELSE ' PM' END
FROM time_slots
WHERE slot_start < TIME '17:00:00'
)
SELECT * FROM time_slots
```

**Student Note:** "This is A VERY DIFFICULT EXAMPLE - a 5/5 for the type casting and multiple type changing"

**Score: 9/10** - Correct TIME usage. Complex string manipulation handled.

---

## Task 3: Weekly Date Ranges for a Month

**Scenario:**
Generate all calendar weeks in January 2025 with start/end dates.

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH RECURSIVE weeks_2025 AS (
SELECT
	1 AS week_number,
	'2024-12-30'::DATE AS week_start,
	('2024-12-30'::DATE + INTERVAL '6' DAY)::DATE AS week_end_sunday
UNION ALL
SELECT
	week_number + 1,
	(week_start::DATE + INTERVAL '7' DAY)::DATE,
	(week_end_sunday::DATE + INTERVAL '7' DAY)::DATE
FROM weeks_2025
WHERE week_start < '2025-01-27'::DATE
)
SELECT * FROM weeks_2025
```

**Student Note:** "This task wasn't that difficult, as we didn't have to do complex type casting and/or string concatenation"

**Score: 10/10** - Correct boundary handling with three columns.

---

**Day 4 Overall Score: 9.7/10**


---

### Task Archive: 2026-01-20 (Week 6, Day 5)

**Focus:** Recursive CTEs — Consolidation & Simple Combinations

## Task 1: Generate Year-End Countdown

**Scenario:**
Generate countdown showing last 10 days of 2025 with days until new year counter.

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH RECURSIVE year_end_countdown AS (
SELECT 
	'2025-12-22'::DATE AS date_,
	('2026-01-01'::DATE - '2025-12-22'::DATE) AS days_until_new_year
UNION ALL
SELECT
	(date_ + INTERVAL '1' DAY)::DATE,
	days_until_new_year - 1
FROM year_end_countdown
WHERE days_until_new_year > 1
)
SELECT * FROM year_end_countdown
```

**Score: 10/10** - Clever dynamic calculation of days_until instead of hardcoding.

---

## Task 2: Generate Fiscal Quarters (April Start)

**Scenario:**
Generate 4 fiscal quarters for FY2025 (April 2025 - March 2026).

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH RECURSIVE fiscal_quarters_2025 AS (
SELECT
	'FY25-Q' || 1 AS fiscal_quarter,
	'2025-04-01'::DATE AS quarter_start,
	('2025-04-01'::DATE + INTERVAL '3' MONTH)::DATE AS quarter_end,
	1 AS quarter_num
UNION ALL
SELECT
	'FY25-Q' || quarter_num + 1,
	(quarter_start::DATE + INTERVAL '3' MONTH)::DATE,
	(quarter_end::DATE + INTERVAL '3' MONTH - INTERVAL '1' DAY)::DATE,
	quarter_num + 1
FROM fiscal_quarters_2025
WHERE quarter_end < '2026-03-28'
)
SELECT * FROM fiscal_quarters_2025
```

**Score: 8/10** - Good structure. Anchor quarter_end off by 1 day (should be June 30, not July 1).

---

## Task 3: Daily Transaction Summary with Generated Dates

**Scenario:**
Generate all days in December 2025 with transaction counts and totals.

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH RECURSIVE december_transactions AS (
SELECT 
	'2025-12-01'::DATE AS day_
UNION ALL
SELECT
	(day_ + INTERVAL '1' DAY)::DATE
	FROM december_transactions
	WHERE day_ < '2025-12-31'::DATE
)
SELECT 
	dt.day_,
	COUNT(t.id) AS transaction_count,
	COALESCE(ROUND(SUM(t.amount), 2), 0) AS daily_total
FROM december_transactions dt
LEFT JOIN transactions t ON dt.day_ = DATE(t.created_at)
GROUP BY dt.day_
ORDER BY dt.day_
```

**Score: 10/10** - Perfect recursive CTE + LEFT JOIN pattern.

---

**Day 5 Overall Score: 9.3/10**

**Week 6 Complete - Average: 9.5/10**


---

### Task Archive: 2026-01-21 (Week 7, Day 1)

**Focus:** Recursive CTEs — Continued Practice + Carrying Forward Values

## Task 1: Generate a Simple Number Pyramid

**Scenario:**
Generate numbers 1 through 5 with cumulative sum at each step.

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH RECURSIVE snp AS (
SELECT
	1 AS n,
	1::NUMERIC AS cumulative_sum
UNION ALL
SELECT
	n + 1,
	SUM(cumulative_sum::NUMERIC) OVER (ORDER BY n::NUMERIC) + n + 1 AS cumulative_sum
	FROM snp
	WHERE n < 5
	)
SELECT * FROM snp
```

**Score: 7/10** - Overcomplicated with window function. Simpler: `cumulative_sum + (n + 1)`.

---

## Task 2: Weekly Order Summary for Q1 2025

**Scenario:**
Generate weeks in Q1 2025, show order count and revenue per week.

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH RECURSIVE weeks_q1 AS (
SELECT
	'2024-12-30'::DATE AS week_start,
	1 AS week_number
UNION ALL
SELECT
	(week_start + INTERVAL '7' DAY)::DATE,
	week_number + 1
FROM weeks_q1
WHERE week_start::DATE < '2025-03-31'::DATE
)
SELECT
	wq1.week_start,
	wq1.week_number,
	COALESCE(COUNT(o.id), 0) AS order_count,
	COALESCE(SUM(o.amount), 0) AS weekly_revenue
FROM weeks_q1 wq1
LEFT JOIN orders o ON wq1.week_start < DATE(o.created_at) AND DATE(o.created_at) < (wq1.week_start + INTERVAL '7' DAY)::DATE
GROUP BY wq1.week_start, wq1.week_number
```

**Score: 9/10** - Good pattern. Minor boundary issue: `<` should be `>=` for week_start.

---

## Task 3: Power of 2 Sequence

**Scenario:**
Generate powers of 2 from 2^0 to 2^10.

**Difficulty Rating:** 2/5

**Student Solution:**
```sql
WITH RECURSIVE powers AS (
SELECT
	0 AS exponent,
	1::NUMERIC
	AS power_of_2
UNION ALL
SELECT 
	exponent + 1,
	POWER(2, exponent)::NUMERIC
FROM powers
WHERE exponent < 10
)
SELECT * FROM powers
```

**Score: 7/10** - Used POWER() instead of carrying forward. Results off by one. Simpler: `power_of_2 * 2`.

---

**Day 1 Overall Score: 7.7/10**

**Key Learning:** Recursive CTEs carry values forward naturally — no need for window functions or POWER().


---

### Task Archive: 2026-01-22 (Week 7, Day 2)

**Focus:** Recursive CTEs — The "Carry Forward" Pattern

## Task 1: Factorial Sequence

**Scenario:**
Generate factorials from 1! to 7!.

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH RECURSIVE factorials AS (
SELECT
	1 AS n,
	1 AS factorial
UNION ALL
SELECT 
	n + 1,
	factorial * (n + 1)
FROM factorials
WHERE n < 7
)
SELECT * FROM factorials
```

**Student Note:** "I get that pattern already, there's no need to reiterate OVER SUCH SIMPLE TASKS ANYMORE"

**Score: 10/10**

---

## Task 2: Compound Interest Growth

**Scenario:**
$1000 at 10% annual interest for 5 years.

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH RECURSIVE interest_results AS (
SELECT
	ROUND(1000, 2) AS balance,
	0 AS year_,
	10 AS interest_rate
UNION ALL
SELECT 
	ROUND(balance + interest_rate * balance / 100, 2),
	year_ + 1,
	interest_rate
FROM interest_results
WHERE year_ < 5
)
SELECT 
	balance,
	year_,
	interest_rate || '%' AS interest_rate
FROM interest_results
```

**Student Note:** Added % formatting. Observed that numeric values must be kept during recursion, formatted only at display.

**Score: 10/10**

---

## Task 3: Fibonacci Sequence

**Scenario:**
Generate first 10 Fibonacci numbers.

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH RECURSIVE fibo_numbers AS (
SELECT
	1 AS position_,
	1 AS fib_value,
	0 AS prev_value
UNION ALL
SELECT 
	position_ + 1,
	fib_value + prev_value,
	fib_value
FROM fibo_numbers
WHERE fib_value < 55
)
SELECT * FROM fibo_numbers
```

**Student Note:** "I think I've mastered it, so we can move further with more complex tasks now."

**Score: 10/10**

---

**Day 2 Overall Score: 10/10**

**Note:** Student requests increased complexity — basic recursive CTE patterns mastered.


---

### Task Archive: 2026-01-23 (Week 7, Day 3)

**Focus:** Recursive CTEs — Moderate Step Up (Adjusted from original 5/5 difficulty)

## Task 1: Generate Date Range from Actual Data

**Scenario:**
Generate all dates between earliest and latest order, count orders per day.

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH RECURSIVE min_max_date AS (
SELECT 
	MIN(DATE(created_at)) AS min_date,
	MAX(DATE(created_at)) AS max_date
FROM orders
),
chronological_dates AS (
SELECT 
	mmd.min_date::DATE AS date_,
	mmd.max_date::DATE AS max_date
FROM min_max_date mmd
UNION ALL
SELECT
	(date_ + INTERVAL '1' DAY)::DATE,
	max_date::DATE
FROM chronological_dates
WHERE date_ < max_date
)
SELECT 
	cd.date_,
	COALESCE(COUNT(o.id), 0) AS order_count
FROM chronological_dates cd
LEFT JOIN orders o ON cd.date_ = DATE(o.created_at)
GROUP BY cd.date_
ORDER BY cd.date_
```

**Score: 10/10** - Perfect execution with dynamic date bounds.

---

## Task 2: Countdown with Conditional Message

**Scenario:**
New Year countdown Dec 25-31 with conditional messages.

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH RECURSIVE new_year_countdown AS (
SELECT
	'2025-12-25'::DATE AS date_,
	'2025-12-31'::DATE - '2025-12-25'::DATE AS days_left,
	'Almost there' AS message
UNION ALL
SELECT
	(date_ + INTERVAL '1' DAY)::DATE,
	'2025-12-31'::DATE - date_::DATE AS days_left,
	CASE 
		WHEN days_left > 4 THEN 'Almost there'	
		WHEN days_left > 0 AND days_left < 5 THEN 'So close!'
		WHEN days_left = 0 THEN 'Happy New Year!'
	END
FROM new_year_countdown
WHERE date_ < '2026-01-01'::DATE
)
SELECT * FROM new_year_countdown
```

**Score: 8/10** - Good structure, timing issue with old vs new values in CASE WHEN.

---

## Task 3: Running Total of Daily Orders

**Scenario:**
Daily order counts with cumulative running total.

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH daily_order_counts AS (
SELECT 
	DATE(created_at) AS day_,
	COUNT(*) AS order_count
FROM orders
GROUP BY DATE(created_at)
ORDER BY DATE(created_at)
)
SELECT 
	doc.day_,
	doc.order_count,
	SUM(amount) OVER (ORDER BY o.created_at) AS running_total
FROM daily_order_counts doc
JOIN orders o ON doc.day_ = DATE(o.created_at)
```

**Score: 6/10** - Wrong column (amount vs count), unnecessary JOIN created duplicates.

---

**Day 3 Overall Score: 8/10**

**Note:** Original tasks were too difficult (5/5) and caused frustration. Adjusted to 3-4/5 difficulty mid-session.


---

### Task Archive: 2026-01-24 (Week 7, Day 4)

**Focus:** Recursive CTEs + Window Functions Combined

## Task 1: Monthly Order Stats with Dynamic Date Range

**Scenario:**
Generate all months between first and last order, show order count, revenue, and running cumulative revenue.

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH RECURSIVE min_max_order_dates AS (
SELECT 
	DATE_TRUNC('Month', MIN(created_at)) AS first_order_month,
	DATE_TRUNC('Month', MAX(created_at)) AS last_order_month
FROM orders
),
month_range AS (
SELECT
	first_order_month AS month_date,
	last_order_month
	FROM min_max_order_dates
UNION ALL
SELECT
	month_date + INTERVAL '1' MONTH,
	last_order_month
FROM month_range
WHERE month_date::date < last_order_month::DATE
),
months_orders_revenue AS (
SELECT 
	mr.month_date,
	COALESCE(COUNT(o.id), 0) AS order_count,
	COALESCE(SUM(amount), 0) AS monthly_revenue
FROM month_range mr
LEFT JOIN orders o ON mr.month_date = DATE_TRUNC('Month', o.created_at)
GROUP BY mr.month_date
ORDER BY mr.month_date
)
SELECT 
	*,
	SUM(monthly_revenue) OVER (ORDER BY month_date) AS cumulative_revenue
FROM months_orders_revenue
```

**Score: 10/10** - Perfect combination of recursive CTE + LEFT JOIN + window function.

---

## Task 2: Depreciation Schedule

**Scenario:**
$10,000 asset depreciating 15% annually for 7 years.

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH RECURSIVE depreciation_schema AS (
SELECT 
	1 AS year_,
	10000::NUMERIC AS start_value,
	10000 * 0.15 AS depreciation,
	(10000 - (10000 * 0.15))::NUMERIC AS end_value
UNION ALL
SELECT
	year_ + 1,
	end_value AS start_value,
	ROUND(end_value * 0.15, 2) AS depreciation,
	ROUND(end_value - end_value * 0.15, 2)
FROM depreciation_schema
WHERE year_ < 7
)
SELECT * FROM depreciation_schema
```

**Score: 10/10** - Perfect carry-forward pattern.

---

## Task 3: User Registration by Week with Running Total

**Scenario:**
Weekly user registration counts with cumulative running total.

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH registration_count_weeks AS (
SELECT 
	DATE_TRUNC('Week', created_at) AS week_start,
	COUNT(*) AS new_users
FROM users
GROUP BY week_start
)
SELECT 
	*,
	SUM(new_users) OVER (ORDER BY week_start) AS total_users
FROM registration_count_weeks
```

**Score: 10/10** - Fixed Day 3's mistake. Clean window function on aggregated CTE.

---

**Day 4 Overall Score: 10/10**


---

### Task Archive: 2026-01-25 (Week 7, Day 5)

**Focus:** Recursive CTEs — Week Wrap-Up

---

## Task 1: Daily Revenue with 7-Day Moving Average

**Scenario:**
For each day with orders, show the daily revenue AND a 7-day trailing moving average.

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH orders_daily_revenue AS (
SELECT
	DATE(created_at) AS order_date,
	SUM(amount) AS daily_revenue
FROM orders
GROUP BY order_date
)
SELECT
	*,
	ROUND(AVG(daily_revenue) OVER (ORDER BY order_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)::NUMERIC, 2) AS moving_avg_7d
FROM orders_daily_revenue
```

**Score: 10/10** - Perfect window frame usage.

---

## Task 2: Loan Amortization Schedule

**Scenario:**
$5,000 loan at 1% monthly interest with $500 monthly payments. Generate amortization schedule.

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH RECURSIVE loan_repayment AS (
SELECT
	1 AS MONTH,
	5000::numeric AS balance,
	5000 / 100::NUMERIC AS interest,
	500::numeric AS payment,
	5000 + (0.01 * 5000) - 500 AS ending_balance
UNION ALL
SELECT
	MONTH + 1,
	ending_balance AS starting_balance,
	round(ending_balance / 100, 2),
	ROUND(CASE WHEN ending_balance + (0.01 * ending_balance) < 500 THEN ending_balance ELSE 500 END, 2) AS payment,
	ROUND(CASE WHEN ending_balance + (0.01 * ending_balance) < 500 THEN 0 ELSE ending_balance + (0.01 * ending_balance) - 500 END, 2) AS ending_balance
FROM loan_repayment
WHERE ending_balance > 0
)
SELECT * FROM loan_repayment
```

**Score: 9/10** - Excellent recursive logic. Minor: final payment should include interest (ending_balance + interest, not just ending_balance).

---

## Task 3: Product Sales Ranking by Month

**Scenario:**
Rank products by revenue per month, show top 3.

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH products_monthly_revenues AS (
SELECT
	DATE_TRUNC('Month', o.created_at) AS month_,
	p.name AS product_name,
	SUM(op.quantity * p.price) AS monthly_revenue
FROM orders_products op
JOIN orders o ON op.order_id = o.id
JOIN products p ON p.id = op.product_id
GROUP BY month_, product_name
),
monthly_revenues_ranks AS (
SELECT
	*,
	RANK() OVER (PARTITION BY month_ ORDER BY monthly_revenue DESC) AS rank
FROM products_monthly_revenues
)
SELECT * FROM monthly_revenues_ranks
WHERE RANK < 4
```

**Score: 10/10** - Perfect RANK() with PARTITION BY.

---

**Day 5 Overall Score: 29/30 (9.67/10)**

**Week 7 Overall: 9.07/10 average**


---

### Task Archive: 2026-01-30 (Week 8, Day 1)

**Focus:** Hierarchical Recursive CTEs — Introduction (2-level)

---

## Task 1: Department-Team Hierarchy (2 Levels)

**Scenario:**
Create 2-level hierarchy: Departments → Teams

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH RECURSIVE department_hierarchy(team, parent) AS (
VALUES
    ('Backend', 'Engineering'),
    ('Frontend', 'Engineering'),
    ('Inside Sales', 'Sales'),
    ('Enterprise', 'Sales'),
    ('Digital', 'Marketing'),
    ('Brand', 'Marketing')
),
hierarchy AS (
SELECT
    1 AS LEVEL,
    name,
    NULL::text AS parent_name
FROM (VALUES('Engineering'), ('Sales'), ('Marketing')) AS d(name)
UNION ALL
SELECT
    h.LEVEL + 1,
    dt.team,
    h.name
FROM HIERARCHY h
JOIN department_hierarchy dt ON dt.parent = h.name
WHERE h.LEVEL = 1
)
SELECT * FROM hierarchy
```

**Score: 10/10** - Correct pattern with mapping CTE. Required initial guidance.

---

## Task 2: Countries & Cities (Same Pattern)

**Scenario:**
Apply same pattern: Countries → Cities

**Difficulty Rating:** 2/5

**Student Solution:**
```sql
WITH RECURSIVE countries_hierarchy(country, city) AS (
VALUES
    ('USA', 'New York'),
    ('USA', 'Los Angeles'),
    ('Germany', 'Berlin'),
    ('Germany', 'Munich')
),
HIERARCHY AS (
SELECT
    1 AS level_,
    name,
    NULL::TEXT AS parent_name
FROM UNNEST(ARRAY['USA', 'Germany']) AS d(name)
UNION ALL
SELECT
    h.level_ + 1,
    ch.city,
    h.name
FROM HIERARCHY h
JOIN countries_hierarchy ch ON h.name = ch.country
)
SELECT * FROM hierarchy
```

**Score: 10/10** - Applied pattern independently. Used UNNEST alternative.

---

## Task 3: Conceptual Questions

**Score: 8/10** - Q1-Q3 correct. Q4 incorrect (WHERE stops recursion, not sorting).

---

**Day 1 Overall Score: 28/30**


---

### Task Archive: 2026-01-31 (Week 8, Day 2)

**Focus:** Hierarchical CTEs with Real Table Data + Window Functions

---

## Task 1: Categories → Products Hierarchy

**Scenario:**
Build 2-level hierarchy using product_categories and products tables.

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH RECURSIVE hierarchy AS (
SELECT
    1 AS level,
    id AS category_id,
    name::text AS name,
    NULL::TEXT AS parent_name
FROM product_categories
UNION ALL
SELECT
    h.LEVEL + 1,
    h.category_id,
    p.name,
    h.name
FROM HIERARCHY h JOIN products p ON h.category_id = p.category_id
WHERE h.LEVEL < 2
)
SELECT * FROM hierarchy
```

**Score: 10/10** - Correct: included category_id for JOIN.

---

## Task 2: Users → Orders Hierarchy

**Scenario:**
Build 2-level hierarchy: Users → Orders

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH RECURSIVE order_hierarchy AS (
SELECT
    1 AS LEVEL,
    id AS user_id,
    email::TEXT AS name,
    NULL::TEXT AS parent_id
FROM users
UNION ALL
SELECT
    oh.LEVEL + 1,
    oh.user_id,
    o.ID::TEXT,
    oh.name
FROM order_hierarchy oh JOIN orders o ON o.user_id = oh.user_id
WHERE oh.LEVEL < 2
)
SELECT LEVEL, USER_ID, name, parent_id
FROM order_hierarchy
WHERE user_id < 6
```

**Student Note:** "I had to include oh.user_id to be able to actually join orders based on that user_id"

**Score: 10/10** - Key insight: carry IDs through recursion for JOINs.

---

## Task 3: PERCENT_RANK Window Function

**Scenario:**
Calculate percentile ranking of users by total spending.

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH users_spent AS (
SELECT user_id, SUM(amount) AS total_spent
FROM orders
GROUP BY user_id
)
SELECT *, PERCENT_RANK() OVER (ORDER BY total_spent) AS percentile_rank
FROM users_spent
ORDER BY total_spent DESC
```

**Student Note:** "Very easy task for me."

**Score: 9/10** - Missing ROUND and filter, but core concept correct.

---

**Day 2 Overall Score: 29/30**


---

### Task Archive: 2026-02-01 (Week 8, Day 3)

**Focus:** 3-Level Hierarchies + Advanced Window Functions

---

## Task 1: 3-Level Hardcoded Hierarchy

**Scenario:**
Company → Department → Team (3 levels)

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH RECURSIVE mapping(child, parent) AS (
VALUES
    ('Engineering', 'TechCorp'),
    ('Sales', 'TechCorp'),
    ('Backend', 'Engineering'),
    ('Frontend', 'Engineering'),
    ('Inside Sales', 'Sales'),
    ('Enterprise', 'Sales')
),
HIERARCHY AS (
SELECT 1 AS LEVEL, 'TechCorp' AS name, NULL::TEXT AS parent_name
UNION ALL
SELECT h.LEVEL + 1, m.child, h.name
FROM HIERARCHY h JOIN MAPPING m ON m.parent = h.name
WHERE h.LEVEL < 3
)
SELECT * FROM hierarchy
```

**Student Note:** "NOT feeling confident... need to practice to get comfortable"

**Score: 8/10** - Works but needed heavy guidance.

---

## Task 2: 3-Level with Real Data

**Scenario:**
Categories → Products → Orders (3 levels with real tables)

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH RECURSIVE hierarchy AS (
SELECT 1 AS LEVEL, id AS category_id, NULL::INT AS product_id,
       NULL::TEXT AS name, NULL::TEXT AS parent_name
FROM product_categories WHERE name = 'travel'
UNION ALL
SELECT h.level + 1, h.category_id, COALESCE(p.id, h.product_id),
       COALESCE(p.name, op.order_id::TEXT), h.name
FROM HIERARCHY h
LEFT JOIN products p ON h.LEVEL = 1 AND p.category_id = p.category_id
LEFT JOIN orders_products op ON h.LEVEL = 2 AND op.product_id = h.product_id
WHERE h.LEVEL < 3
)
SELECT LEVEL, name, parent_name FROM hierarchy
```

**Student Note:** "still don't understand it... feels really steep/difficult"

**Score: 6/10** - Bug in JOIN condition. Task was too advanced.

---

## Task 3: LAG Running Difference

**Scenario:**
Show transactions with previous amount and difference.

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH users_transactions AS (
SELECT user_id, DATE(created_at) AS transaction_date, amount,
       LAG(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_amount
FROM transactions
)
SELECT *, CASE WHEN prev_amount IS NULL THEN NULL ELSE amount - prev_amount END AS amount_diff
FROM users_transactions WHERE user_id < 6
```

**Student Note:** "easy peasy, I didn't even think when solving it."

**Score: 10/10** - Perfect.

---

**Day 3 Overall Score: 24/30**

**Curriculum Note:** Task 2 was too steep. Days 4-5 revised for more gradual progression.


---

### Task Archive: 2026-02-02 (Week 8, Day 4)

**Focus:** 3-Level Hierarchy Practice + Path Building

---

## Task 1: 3-Level Geography Hierarchy

**Scenario:**
World → Continents → Countries (3 levels)

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH RECURSIVE mapping(child, parent) AS (
VALUES
('Europe', 'World'),
('Asia', 'World'),
('France', 'Europe'),
('Germany', 'Europe'),
('Japan', 'Asia'),
('China', 'Asia')
),
HIERARCHY AS (
SELECT 1 AS LEVEL, 'World' AS name, NULL::TEXT AS parent_name
UNION ALL
SELECT h.LEVEL + 1, m.child, h.name
FROM HIERARCHY h JOIN MAPPING m ON h.name = m.parent
WHERE LEVEL < 3
)
SELECT * FROM hierarchy
```

**Student Note:** "I honestly had to look it up, but it's getting clearer"

**Score: 9/10** - Correct, needed to look back.

---

## Task 2: Path Building

**Scenario:**
Add full path column to hierarchy.

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH RECURSIVE mapping(child, parent) AS (...),
HIERARCHY AS (
SELECT 1 AS LEVEL, 'World' AS name, NULL::TEXT AS parent_name, 'World' AS path
UNION ALL
SELECT h.LEVEL + 1, m.child, h.name, h.path || ' > ' || m.child
FROM HIERARCHY h JOIN MAPPING m ON h.name = m.parent
WHERE LEVEL < 3
)
SELECT * FROM hierarchy
```

**Score: 10/10** - Perfect path concatenation.

---

## Task 3: NTILE Quartile Bucketing

**Scenario:**
Divide users into quartiles by spending.

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH users_orders AS (
SELECT user_id, SUM(amount) AS total_spent
FROM orders GROUP BY user_id
)
SELECT *, NTILE(4) OVER (ORDER BY total_spent) AS quartile
FROM users_orders
ORDER BY quartile, total_spent DESC
```

**Score: 10/10** - Clean and correct.

---

**Day 4 Overall Score: 29/30**


---

### Task Archive: 2026-02-03 (Week 8, Day 5)

**Focus:** Week Consolidation — Hierarchy Review + Window Functions

---

## Task 1: 3-Level Org Hierarchy with Path

**Scenario:**
Acme Inc → HR/Finance/IT → Teams (3 levels with path)

**Difficulty Rating:** 3/5

**Score: 10/10** - Completed from memory.

---

## Task 2: Categories → Products with Path (Real Data)

**Scenario:**
2-level hierarchy with path using product_categories and products.

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH RECURSIVE HIERARCHY AS (
SELECT 1 AS LEVEL, id AS id, name::TEXT, NULL::TEXT AS parent_name, name::TEXT AS path
FROM product_categories
UNION ALL
SELECT h.LEVEL + 1, p.id, p.name, h.name, h.PATH || ' > ' || p.name
FROM HIERARCHY H JOIN products p ON h.id = p.category_id
WHERE h.LEVEL < 2
)
SELECT LEVEL, NAME, parent_name, path FROM hierarchy
```

**Score: 10/10** - Clean real data hierarchy with path.

---

## Task 3: DENSE_RANK Product Ranking

**Scenario:**
Rank products by total quantity sold.

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH products_quantities AS (
SELECT op.product_id, p.name AS product_name, SUM(op.quantity) AS total_quantity
FROM orders_products op
JOIN products p ON op.product_id = p.id
GROUP BY OP.product_id, p.name
)
SELECT *, DENSE_RANK() OVER (ORDER BY total_quantity DESC) AS sales_rank
FROM products_quantities
ORDER BY sales_rank, product_name
```

**Student Note:** "SUPER easy... I'd appreciate tasks that require multiple CTE"

**Score: 10/10** - Trivial for student at this level.

---

**Day 5 Overall Score: 30/30**

**Week 8 Overall: 28/30 average (9.33/10)**


---

### Task Archive: 2026-02-04 (Week 9, Day 1)

**Focus:** Hierarchy Practice + Multi-CTE Challenges

---

## Task 1: 3-Level Price Tier Hierarchy

**Scenario:**
All Products → Budget/Mid-Range/Premium → Price sub-tiers (3 levels with path)

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
WITH RECURSIVE mapping(child, parent) AS (
VALUES
('Budget (<$50)', 'All Products'),
('Mid-Range ($50 - $150)', 'All Products'),
('Premium (>$150)', 'All Products'),
('Under $20', 'Budget (<$50)'),
('$20-$50', 'Budget (<$50)'),
('$50-$100', 'Mid-Range ($50 - $150)'),
('$100-$150', 'Mid-Range ($50 - $150)'),
('$150-$300', 'Premium (>$150)'),
('$300+', 'Premium (>$150)')
),
HIERARCHY AS (
SELECT 1 AS LEVEL, 'All Products' AS name, NULL::TEXT AS parent_name, 'All Products' AS PATH
UNION ALL
SELECT h.LEVEL + 1, m.child, h.name, h.PATH || ' > ' || m.child
FROM HIERARCHY h JOIN MAPPING m ON h.name = m.parent
)
SELECT * FROM hierarchy
```

**Student Note:** "Written 100% from memory, weekend break helped consolidate"

**Score: 10/10** - From memory, pattern solidified.

---

## Task 2: Monthly Revenue Dashboard (Multi-CTE)

**Scenario:**
Monthly revenue with MoM change, percentage, 3-month moving avg, vs overall.

**Difficulty Rating:** 5/5

**Student Solution:**
```sql
WITH monthly_revenues AS (
SELECT DATE_TRUNC('Month', created_at) AS month_, SUM(amount) AS monthly_revenue
FROM orders GROUP BY DATE_TRUNC('Month', created_at)
),
prev_monthly_revs AS (
SELECT *, LAG(monthly_revenue) OVER (ORDER BY month_) AS prev_month_revenue
FROM monthly_revenues
),
revenues_changes_averages AS (
SELECT *,
    monthly_revenue - prev_month_revenue AS mom_change,
    ROUND((monthly_revenue - prev_month_revenue)::NUMERIC / prev_month_revenue::NUMERIC * 100, 1) AS mom_pct_change,
    ROUND(AVG(monthly_revenue::NUMERIC) OVER (ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS moving_avg_3d
FROM prev_monthly_revs
)
SELECT month_, monthly_revenue, prev_month_revenue, mom_change,
    mom_pct_change || '%' AS mom_pct_change, moving_avg_3d,
    CASE WHEN monthly_revenue > moving_avg_3d THEN 'Above Average' ELSE 'Below Average' END AS vs_overall
FROM revenues_changes_averages
```

**Student Note:** "I enjoyed this task... great way to review advanced queries"

**Score: 10/10** - Excellent multi-CTE layered approach.

---

## Task 3: Category Performance Comparison (Multi-CTE)

**Scenario:**
Category revenue, unique customers, avg order value, revenue share %, rank.

**Difficulty Rating:** 5/5

**Student Solution:**
```sql
WITH categories_revenues AS (
SELECT p.category_id, SUM(op.quantity * p.price) AS total_revenue
FROM orders_products op JOIN products p ON op.product_id = p.id
GROUP BY P.category_id
),
categories_ranked AS (
SELECT *, RANK() OVER (ORDER BY total_revenue DESC) AS revenue_rank,
    ROUND(total_revenue / (SELECT SUM(total_revenue) FROM categories_revenues) * 100, 1) || '%' AS revenue_share_pct
FROM categories_revenues
),
categories_customers_avg_order AS (
SELECT p.category_id, pc.name AS category_name,
    COUNT(DISTINCT(o.user_id)) AS unique_customers,
    ROUND(AVG(o.amount)::NUMERIC, 2) AS avg_order_value
FROM orders_products op
JOIN products p ON op.product_id = p.id
JOIN orders o ON op.order_id = o.id
JOIN product_categories pc ON p.category_id = pc.id
GROUP BY p.category_id, pc.name
)
SELECT cca.category_name, cr.total_revenue, cca.unique_customers,
    cca.avg_order_value, cr.revenue_share_pct, cr.revenue_rank
FROM categories_ranked cr
JOIN categories_customers_avg_order cca ON cr.category_id = cca.category_id
```

**Student Note:** "Everything works as intended, it was fun and I loved it!"

**Score: 10/10** - Smart split into separate aggregation CTEs.

---

**Day 1 Overall Score: 30/30**


---

### Task Archive: 2026-02-05 (Week 9, Day 2)

**Focus:** Hierarchy Practice + Complex Aggregation Challenges

---

## Task 1: 3-Level Real Data Hierarchy (Categories → Products → Delivery Status)

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH RECURSIVE HIERARCHY AS (
SELECT 1 AS LEVEL, id AS category_id, name::NAME AS name,
       NULL::TEXT AS parent_name, name::TEXT AS path
FROM product_categories pc WHERE name = 'travel'
UNION ALL
SELECT h.LEVEL + 1, COALESCE(p.id, op.order_id) AS id,
       COALESCE(p.name, d.status) AS name, h.name,
       h.PATH || ' > ' || COALESCE(p.name, d.status)
FROM HIERARCHY h
LEFT JOIN products p ON p.category_id = h.category_id AND h.LEVEL = 1
LEFT JOIN orders_products op ON h.category_id = op.product_id AND h.LEVEL = 2
LEFT JOIN deliveries d ON op.order_id = d.order_id
)
SELECT * FROM hierarchy
```

**Student Note:** "Kind of difficult, but I managed... without help"

**Score: 9/10** - Solved independently. Missing WHERE h.LEVEL < 3.

---

## Task 2: Customer Lifetime Value Segmentation

**Difficulty Rating:** 5/5

**Student Solution:**
```sql
WITH users_metrics_spending_orders AS (
SELECT user_id, ROUND(SUM(amount)::NUMERIC, 2) AS total_spent,
       COUNT(id) AS order_count, ROUND(AVG(amount)::NUMERIC, 2) AS avg_order_value,
       EXTRACT('Days' FROM MAX(created_at) - MIN(created_at)) AS days_as_customer
FROM ORDERS GROUP BY USER_id
),
users_orders_per_month_prct_rank AS (
SELECT *,
    CASE WHEN days_as_customer = 0 THEN order_count
    ELSE ROUND(order_count / (days_as_customer / 30.0), 2) END AS orders_per_month,
    PERCENT_RANK() OVER (ORDER BY total_spent)
FROM users_metrics_spending_orders
)
SELECT *,
    CASE WHEN percent_rank >= 0.9 THEN 'Platinum'
         WHEN percent_rank >= 0.75 THEN 'Gold'
         WHEN percent_rank >= 0.5 THEN 'Silver'
         WHEN percent_rank < 0.5 THEN 'Bronze' END AS value_tier
FROM users_orders_per_month_prct_rank ORDER BY total_spent DESC
```

**Score: 9/10** - Efficient CTE combination. Missing single-order user filter.

---

## Task 3: Order Gap Analysis with Trend

**Difficulty Rating:** 5/5

**Student Solution:**
```sql
WITH users_orders AS (
SELECT *, DATE_TRUNC('Day', created_at) AS day_,
       LAG(DATE_TRUNC('Day', created_at)) OVER (PARTITION BY user_id ORDER BY created_at) AS previous_order_day
FROM orders
),
users_order_gaps AS (
SELECT *, day_ - previous_order_day AS gap_days
FROM users_orders WHERE previous_order_day IS NOT NULL
),
users_last_gaps AS (
SELECT DISTINCT user_id,
       FIRST_VALUE(gap_days) OVER (PARTITION BY user_id ORDER BY created_at DESC) AS last_gap_days
FROM users_order_gaps
),
users_gap_metrics AS (
SELECT user_id, COUNT(id) AS order_count,
       AVG(gap_days) AS avg_gap_days, MAX(gap_days) AS max_gap_days
FROM users_order_gaps GROUP BY user_id
)
SELECT *,
    CASE WHEN ugm.avg_gap_days > ulg.last_gap_days THEN 'Accelerating'
         WHEN ugm.avg_gap_days < ulg.last_gap_days THEN 'Slowing Down'
         WHEN ugm.avg_gap_days = ulg.last_gap_days THEN 'Steady' END AS trend
FROM users_gap_metrics ugm
JOIN users_last_gaps ulg ON ulg.user_id = ugm.user_id
ORDER BY avg_gap_days DESC
```

**Score: 9/10** - Clean 4-CTE structure. Missing 3+ orders filter.

---

**Day 2 Overall Score: 27/30**


---

### Task Archive: 2026-02-06 (Week 9, Day 3)

**Focus:** Hierarchy + Subquery & CTE Combinations

---

## Task 1: 3-Level Hierarchy — Users → Order Count Tiers → Specific Users

**Scenario:**
Build a 3-level hierarchy with hardcoded tiers and real user data:
- Level 1: 'All Users'
- Level 2: 'Power Users' (10+ orders), 'Regular Users' (3-9 orders), 'New Users' (1-2 orders)
- Level 3: Actual user emails from the `orders` table, matched to their tier

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH RECURSIVE mapping(child, parent) AS (
VALUES
('Power Users', 'All Users'),
('Regular Users', 'All Users'),
('New Users', 'All Users')
),
users_order_counts AS (
SELECT user_id, COUNT(id) AS order_count
FROM orders GROUP BY user_id
),
users_tiers AS (
SELECT *, CASE WHEN uoc.order_count > 10 THEN 'Power Users'
    WHEN uoc.order_count >= 3 THEN 'Regular Users' ELSE 'New Users' END AS user_tier
FROM users_order_counts uoc JOIN users u ON uoc.user_id = u.id
),
users_limited_tiers AS (
SELECT *, ROW_NUMBER() OVER (PARTITION BY user_tier ORDER BY order_count DESC) AS tier_count_ranking
FROM users_tiers
),
HIERARCHY AS (
SELECT 1 AS LEVEL, NULL AS id, 'All Users'::TEXT AS name, NULL::TEXT AS parent_name, NULL::NUMERIC AS cnt
UNION ALL
SELECT h.LEVEL + 1, COALESCE(h.id, ut.user_id::TEXT), COALESCE(m.child, ut.email::TEXT), h.name,
    COALESCE(h.cnt::NUMERIC, ut.order_count::NUMERIC)
FROM HIERARCHY h
LEFT JOIN MAPPING m ON m.parent = h.name AND h.LEVEL = 1
LEFT JOIN users_limited_tiers ut ON h.name = ut.user_tier AND h.LEVEL = 2 AND ut.tier_count_ranking <= 3
WHERE h.LEVEL < 3
)
SELECT LEVEL, name, parent_name FROM HIERARCHY WHERE name IS NOT NULL
```

**Student Note:** "It was a loooong task, that took me around 30+ minutes and got me a hella long query + I've realized that limiting requirement in the end, but I've managed to do everything and it's quite satisfying."

**Score: 10/10** - Most complex hierarchy yet. 5 CTEs combining hardcoded mapping + real data with ROW_NUMBER limiting. Fully independent.

---

## Task 2: Revenue Anomaly Detection (Multi-CTE)

**Scenario:**
Find days where revenue was "anomalous" — significantly above or below the norm. Define anomalous as more than 1.5x the 7-day moving average, or less than 0.5x.

**Difficulty Rating:** 5/5

**Student Solution:**
```sql
WITH daily_order_revenues AS (
SELECT DATE_TRUNC('Day', created_at) AS day_, SUM(amount) AS daily_revenue
FROM orders GROUP BY DATE_TRUNC('Day', created_at) ORDER BY day_
),
daily_revs_moving_avg AS (
SELECT *, ROUND(AVG(daily_revenue) OVER (ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)::NUMERIC, 2) AS moving_avg_7d
FROM daily_order_revenues
)
SELECT *, ROUND(daily_revenue::NUMERIC / moving_avg_7d::numeric, 2) AS ratio_to_avg,
    CASE WHEN ROUND(daily_revenue::NUMERIC / moving_avg_7d::numeric, 2) > 1.5 THEN 'Spike'
         WHEN ROUND(daily_revenue::NUMERIC / moving_avg_7d::numeric, 2) < 0.5 THEN 'Drop' ELSE 'Normal'
    END AS anomaly_type
FROM daily_revs_moving_avg ORDER BY ratio_to_avg DESC
```

**Student Note:** "It was a playful task for me - didn't have to use NULLIF with this approach, but I simply skipped days without orders - reasonable approach."

**Score: 10/10** - Clean 2-CTE + final SELECT approach. Reasonable interpretation of "all days".

---

## Task 3: Cross-Category Buyers Analysis (Multi-CTE)

**Scenario:**
Find users who buy from multiple product categories. Show categories count, favorite category, spending breakdown, and percentage.

**Difficulty Rating:** 5/5

**Student Solution:**
```sql
WITH users_category_spending AS (
SELECT o.user_id, p.category_id, pc.name AS category_name, SUM(p.price * op.quantity) AS category_spending
FROM orders_products op JOIN products p ON op.product_id = p.id
JOIN product_categories pc ON p.category_id = pc.id JOIN orders o ON op.order_id = o.id
GROUP BY o.user_id, p.category_id, pc.name ORDER BY o.user_id
),
categories_total_spent AS (
SELECT user_id, COUNT(category_id) AS categories_count, SUM(category_spending) AS total_spent,
    MAX(category_spending) AS favorite_category_revenue
FROM users_category_spending GROUP BY user_id
)
SELECT cts.user_id, cts.categories_count, ucs.category_name, cts.favorite_category_revenue,
    cts.total_spent, ROUND(cts.favorite_category_revenue / cts.total_spent * 100, 1) || '%' AS favorite_pct
FROM categories_total_spent cts
JOIN users_category_spending ucs ON cts.user_id = ucs.user_id
WHERE cts.favorite_category_revenue = ucs.category_spending AND cts.categories_count >= 2
ORDER BY cts.categories_count DESC, cts.total_spent DESC
```

**Student Note:** "I enjoyed that task - it really prompted me to combine my SQL skills and I had to stop for a while - it offered some kind of a challenge and thinking."

**Score: 10/10** - Smart MAX + JOIN alternative to ROW_NUMBER for finding favorite category.

---

**Day 3 Overall Score: 30/30**


---

### Task Archive: 2026-02-11 (Week 9, Day 4)

**Note:** Weekly limits hit — student assessed independently using an outside model. Completed 3 random review tasks from prior curriculum with ease.

---

## Task 1: Active Users in Both Orders and Sessions

**Scenario:**
Find users who are active in BOTH orders (placed at least 1 order) AND sessions (had at least 1 session with count_sessions > 0).

**Student Solution:**
```sql
WITH users_with_orders AS (
SELECT user_id, COUNT(id) AS order_cnt
FROM orders GROUP BY user_id
)
SELECT usd.user_id, uwo.order_cnt, SUM(usd.count_sessions) AS total_sessions
FROM user_sessions_daily usd JOIN users_with_orders uwo ON uwo.user_id = usd.user_id
GROUP BY usd.user_id, uwo.order_cnt
```

---

## Task 2: Purchase Without Deposit Users

**Scenario:**
How many users did a purchase transaction but never did a deposit transaction?

**Student Solution:**
```sql
SELECT COUNT(DISTINCT(user_id))
FROM transactions
WHERE type = 'purchase' AND user_id NOT IN (SELECT user_id FROM transactions WHERE TYPE = 'deposit')
```

---

## Task 3: Customer Segmentation by Lifetime Spending

**Scenario:**
Segment customers into high-value (> $1000) and low-value (<= $1000) groups with counts and average metrics.

**Student Solution:**
```sql
WITH users_spending AS (
SELECT user_id, SUM(amount) AS total_spending
FROM orders GROUP BY user_id
),
users_segments AS (
SELECT *, CASE WHEN total_spending >= 1000 THEN 'High Value' ELSE 'Low Value' END AS customer_segment
FROM users_spending
)
SELECT customer_segment, ROUND(AVG(total_spending)::NUMERIC, 2) AS avg_segment_spending,
    COUNT(user_id) AS segment_user_counts
FROM users_segments GROUP BY customer_segment
```

**Day 4 Note:** Foundational SQL patterns review — all completed independently with ease.


---

### Task Archive: 2026-02-13 (Week 9, Day 5)

**Focus:** Hierarchy with Dynamic Level 2 + HackerRank-Style Multi-CTE Puzzles

---

## Task 1: 3-Level Hierarchy — Delivery Statuses from Real Data

**Scenario:**
3-level hierarchy where Level 2 is pulled dynamically from `deliveries.status` (no hardcoded VALUES). Level 3: top 3 orders per status by amount DESC.

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH RECURSIVE HIERARCHY AS (
SELECT 1 AS LEVEL, NULL::TEXT AS id, 'All Deliveries'::TEXT AS name,
       NULL::TEXT AS amount, NULL::TEXT AS parent_name, 'All Deliveries'::TEXT AS PATH
UNION ALL
SELECT h.LEVEL + 1, d.order_id::TEXT, COALESCE(d.status, o.id::TEXT)::TEXT,
       COALESCE(NULL, o.amount)::TEXT, h.name, h.PATH || ' > ' || h.name
FROM HIERARCHY h
LEFT JOIN deliveries d ON h.LEVEL = 1
LEFT JOIN orders o ON (o.id)::TEXT = h.id AND h.LEVEL = 2
)
SELECT LEVEL, name, parent_name, path FROM HIERARCHY
```

**Student Note:** "I tried to add amount ordering, but the queries take forever and I just can't wait anymore at some point and cancel it."

**Score: 6/10**
- Bug 1: Missing WHERE h.LEVEL < 3 — infinite recursion causing the performance freeze
- Bug 2: Path appends h.name (parent) instead of new node name
- Bug 3: Non-distinct statuses — joins all delivery rows, not DISTINCT statuses

---

## Task 2: Pareto Analysis — Revenue Concentration (80/20 Rule)

**Scenario:**
Identify users contributing to the top 80% of total revenue. Label as 'Key Account' vs 'Standard Account'.

**Difficulty Rating:** 5/5

**Student Solution:**
```sql
WITH users_revenues AS (
SELECT user_id, SUM(amount) AS total_user_revenue,
       (SELECT ROUND(SUM(amount)::NUMERIC, 2) FROM orders) AS total_revenue
FROM orders GROUP BY user_id
),
users_rev_rank AS (
SELECT *, RANK() OVER (ORDER BY total_user_revenue DESC) AS revenue_rank,
    ROUND(total_user_revenue::NUMERIC / total_revenue::NUMERIC * 100, 2) AS revenue_share_pct
FROM users_revenues
)
SELECT *,
    SUM(revenue_share_pct) OVER (ORDER BY revenue_share_pct DESC) AS cumulative_share_pct,
    CASE WHEN (SUM(revenue_share_pct) OVER (ORDER BY revenue_share_pct DESC)) < 80
         THEN 'Key Account' ELSE 'Standard Account' END AS account_type
FROM users_rev_rank
```

**Score: 10/10** - Correct cumulative window function and classification. ORDER BY via window function equivalent to revenue_rank ASC.

---

## Task 3: Support Ticket Complexity Scoring

**Scenario:**
Score tickets by message count × duration, rank within priority level.

**Difficulty Rating:** 5/5

**Student Solution:**
```sql
WITH tickets_priority_duration AS (
SELECT cm.ticket_id, ct.priority,
       MAX(cm.created_at) - MIN(cm.created_at) AS conversation_duration
FROM chat_tickets ct JOIN chat_messages cm ON ct.id = cm.ticket_id
GROUP BY cm.ticket_id, ct.priority
),
tickets_msg_cnt AS (
SELECT ticket_id, COUNT(id) AS messages_cnt
FROM chat_messages WHERE message_type = 'text'
GROUP BY ticket_id
)
SELECT tpd.ticket_id, tpd.priority, tpd.conversation_duration, tmc.messages_cnt,
       tmc.messages_cnt * EXTRACT('Minute' FROM tpd.conversation_duration) AS complexity_score,
       RANK() OVER (PARTITION BY tpd.priority ORDER BY (tmc.messages_cnt * EXTRACT('Minute' FROM tpd.conversation_duration)) DESC) AS rank
FROM tickets_priority_duration tpd JOIN tickets_msg_cnt tmc ON tpd.ticket_id = tmc.ticket_id
ORDER BY priority
```

**Student Note:** "I extracted minutes as almost all tickets were solved within one hour — data-driven decision."

**Score: 9/10** - Good logic and adaptation. Missing: ROUND on complexity_score, +1 in formula, HAVING messages_cnt >= 2.

---

**Day 5 Overall Score: 25/30**

**Week 9 Overall: 28/30 average (9.33/10)**


### Task Archive: 2026-02-17 (Week 10, Day 1)
# Daily SQL Practice Tasks

**Generated:** 2026-02-16
**Week 10, Day 1 Focus:** Dynamic Hierarchies + HackerRank Hard Puzzles

---

## Task 1: 3-Level Hierarchy — Transaction Types from Real Data

**Scenario:**
Build a 3-level hierarchy where Level 2 is pulled dynamically from real data:
- Level 1: 'All Transactions'
- Level 2: Distinct transaction types from the `transactions` table (no hardcoded VALUES)
- Level 3: Top 3 transactions per type by amount DESC (show transaction ID as text)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — type name or transaction ID
- `parent_name` (text)
- `path` (text) — full path from root

**Requirements:**
- Pre-aggregate distinct types and top-3-per-type before the recursive CTE
- Carry the type name through Level 2 to use as the join key at Level 3
- Include path column
- Do not forget the termination condition

**Difficulty Rating:** 4/5

WITH RECURSIVE transactions_rank AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY TYPE ORDER BY amount DESC) AS transaction_rank
FROM transactions
),
top_three AS (
SELECT * FROM transactions_rank
WHERE transaction_rank <= 3
),
distinct_types AS (
SELECT DISTINCT TYPE FROM top_three
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Transactions' AS name,
	NULL::TEXT AS parent_name,
	'All Transactions' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(dt.TYPE::TEXT, tt.id::TEXT),
	h.name,
	h.PATH || ' > ' || COALESCE(dt.TYPE, tt.id::TEXT)
FROM HIERARCHY h
LEFT JOIN distinct_types dt ON h.LEVEL = 1
LEFT JOIN top_three tt ON h.name = tt."type" AND h.LEVEL = 2
WHERE h.LEVEL < 3
)
SELECT * FROM hierarchy


Finally got it, but it was difficult!


---

## Task 2: Consecutive Month Buyers

**Scenario:**
The retention team wants to identify users with strong purchasing streaks. Find users who placed orders in at least 3 consecutive calendar months. For each qualifying user, show their longest streak.

**Expected Output Columns:**
- `user_id` (integer)
- `streak_start` (date) — first month of the longest streak (truncated to month)
- `streak_end` (date) — last month of the longest streak
- `streak_length` (bigint) — number of consecutive months in the streak
- `streak_revenue` (numeric) — total order revenue during the streak, rounded to 2 decimals

**Requirements:**
- Use `orders` table
- A streak is a sequence of months with no gaps (each month follows the previous by exactly 1 month)
- Only show users whose longest streak is 3+ months
- If a user has multiple streaks of equal length, show the most recent one
- Order by streak_length DESC, streak_revenue DESC

**Difficulty Rating:** 5/5

WITH users_orders_months AS (
SELECT 
	*,
	date_trunc('Month', created_at) AS month_
FROM orders
),
users_order_cnt AS (
SELECT 
	user_id,
	month_,
	COUNT(id) AS order_cnt
FROM users_orders_months
GROUP BY user_id, month_
ORDER BY user_id, month_
),
prev_month_check AS (
SELECT 
	*,
	month_ - INTERVAL '1' MONTH AS valid_month,
	LAG(month_) OVER (PARTITION BY user_id) AS prev_month
FROM users_order_cnt
)
SELECT
	*,
	CASE WHEN valid_month = prev_month THEN 1 ELSE 0 END AS valid_streak_
FROM prev_month_check 
WHERE prev_month IS NOT NULL


I STOPPED HERE - BECAUSE THERE ARE NO SUCH STREAKS.
At this point I was able to visually scan data and there were no users with streaks MORE THAN 2 MONTHS - literally nobody. That would be the time I'd report it to the business. Anyway, I didn't feel like I'm making the smoothest approach here, but I might be wrong!

---

## Task 3: User Cohort Activation Funnel

**Scenario:**
The growth team wants to understand how quickly new users make their first purchase after registering. Group users by their registration month (cohort), then classify them based on time to firstc order.

**Expected Output Columns:**
- `cohort_month` (date) — registration month, truncated to month
- `total_users` (bigint) — users registered in that cohort
- `activated_within_30d` (bigint) — users who placed their first order within 30 days of registration
- `activated_31_to_90d` (bigint) — first order between 31 and 90 days after registration
- `never_activated` (bigint) — users who never placed any order
- `activation_rate_pct` (numeric) — % of cohort who activated within 90 days, rounded to 1 decimal

**Requirements:**
- Use `users` and `orders` tables
- First order = earliest order by created_at per user
- Users with no orders count as never_activated
- Order by cohort_month ASC

**Difficulty Rating:** 5/5


WITH cohort_months AS (
SELECT 
DATE_TRUNC('Month', created_at) AS cohort_month,
COUNT(id) AS total_users
FROM users u
GROUP BY DATE_TRUNC('Month', created_at)
ORDER BY cohort_month
),
users_first_orders AS (
SELECT 
	DISTINCT
	o.user_id,
	u.created_at AS acc_creation_time,
	FIRST_VALUE(o.created_at) OVER (PARTITION BY o.user_id) AS first_order_time
FROM users u
JOIN orders o ON u.id = o.user_id
),
users_activation_time AS (
SELECT 
	u.id AS user_id,
	u.created_at,
	ufo.first_order_time,
	COALESCE(EXTRACT('Day' FROM ufo.first_order_time - ufo.acc_creation_time), 0) AS activation_days
FROM users u LEFT JOIN users_first_orders ufo ON u.id = ufo.user_id
),
users_cohorts_check AS (
SELECT
	date_trunc('Month', created_at) AS creation_month,
	CASE WHEN activation_days > 20 THEN 1 ELSE 0 END AS activated_after_20d,
	CASE WHEN activation_days >= 0 AND activation_days <= 20 AND firsT_order_time IS NOT NULL THEN 1 ELSE 0 END AS activated_within_20d,
	CASE WHEN activation_days = 0 THEN 1 ELSE 0 END AS never_activated
FROM users_activation_time
),
grouped_monthly_cohorts AS (
SELECT 
creation_month,
SUM(activated_after_20d) AS activated_after_20d,
SUM(activated_within_20d) AS activated_within_20d,
SUM(never_activated) AS never_activated
FROM users_cohorts_check
GROUP BY creation_month
)
SELECT 
	cm.cohort_month,
	cm.total_users,
	gmc.activated_after_20d,
	gmc.activated_within_20d,
	gmc.never_activated,
	ROUND(((gmc.activated_after_20d + gmc.activated_within_20d)::NUMERIC / cm.total_users::NUMERIC) * 100, 1) || '%' AS activation_rate_pct
FROM grouped_monthly_cohorts gmc 
JOIN cohort_months cm ON gmc.creation_month = cm.cohort_month
ORDER BY cohort_month

I must admit that this was A REALLY DEMANDING and time consuming task. I had to stop for a few times and get back e.g. to include users that didn't make any orders, to make 100% sure that users who made the order the very same day that they registered will not be taken for users without any orders etc.

But I'm sure I've done it properly in the end as I've verified and checked data in between the steps. Pretty satisfying.

---

## Submission Instructions

1. Task 1 — Dynamic hierarchy from transactions (4/5)
2. Task 2 — Consecutive month buyer streaks (5/5)
3. Task 3 — Cohort activation funnel (5/5)

### Task Archive: 2026-02-17 (Week 10, Day 2)
# Daily SQL Practice Tasks

**Generated:** 2026-02-17
**Week 10, Day 2 Focus:** Gaps-and-Islands, Session Engagement Analysis, Product Category Ranking

---

## Task 1: 3-Level Hierarchy — Delivery Statuses by Month

**Scenario:**
Build a 3-level hierarchy over delivery data:
- Level 1: `'All Deliveries'`
- Level 2: Distinct delivery statuses from the `deliveries` table (pulled dynamically, no hardcoded VALUES)
- Level 3: The 3 most recent deliveries per status (show delivery ID as text)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — status name at Level 2, delivery ID at Level 3
- `parent_name` (text)
- `path` (text) — full path from root

**Requirements:**
- Pre-aggregate distinct statuses and top-3-per-status before the recursive CTE
- Use `created_at DESC` to define "most recent"
- Do not forget the termination condition
- Do not hardcode status values

**Difficulty Rating:** 3/5

WITH RECURSIVE delivery_ranks AS (
SELECT 
	*,
	DENSE_RANK() OVER (PARTITION BY status ORDER BY created_at DESC)
FROM deliveries
),
three_recent_deliveries_by_status AS (
SELECT 
	*
FROM delivery_ranks
WHERE DENSE_RANK <= 3
),
distinct_statuses AS (
SELECT DISTINCT status FROM deliveries
),
HIERARCHY AS (
SELECT
	1 AS level,
	'All Deliveries' AS name,
	NULL::TEXT AS parent_name,
	'All Deliveries' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(ds.status, trd.id::TEXT),
	h.name,
	h.PATH || ' > ' || COALESCE(ds.status, trd.id::TEXT)
FROM HIERARCHY h
LEFT JOIN distinct_statuses ds ON h.LEVEL = 1
LEFT JOIN three_recent_deliveries_by_status trd ON trd.status = h.name
WHERE h.LEVEL < 3
)
SELECT * FROM HIERARCHY

---

## Task 2: User Session Streaks (Gaps-and-Islands)

**Scenario:**
The engagement team wants to identify "power users" — users with long streaks of consecutive days where they had at least one session (`count_sessions > 0`).

Find users whose longest active streak is at least 5 consecutive days. For each qualifying user, show their longest streak only.

**Expected Output Columns:**
- `user_id` (integer)
- `streak_start` (date)
- `streak_end` (date)
- `streak_length` (bigint) — number of consecutive active days
- `avg_daily_sessions` (numeric) — average `count_sessions` during the streak, rounded to 2 decimals

**Requirements:**
- Use `user_sessions_daily` table
- Active day = `count_sessions > 0`
- A streak is a sequence of consecutive calendar days with no gaps
- If a user has multiple streaks of equal max length, show the most recent one
- Order by `streak_length DESC`, `avg_daily_sessions DESC`

**Hint:** The gaps-and-islands pattern — subtract `ROW_NUMBER()` from the date to create a streak group key. Dates within the same streak will share the same group key.

**Difficulty Rating:** 5/5


WITH users_dates AS (
SELECT 
	user_id,
	date,
	LAG(date) OVER (PARTITION BY user_id ORDER BY date) AS prev_session_date,
	count_sessions
FROM user_sessions_daily usd
ORDER BY user_id
),
users_dates_rn AS (
SELECT 
	*,
	ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY date) AS rn
FROM users_dates
WHERE prev_session_date IS NOT NULL
),
users_streak_keys AS (
SELECT 
	*,
	date - rn * INTERVAL '1' DAY AS streak_key
FROM users_dates_rn
)
SELECT 
	user_id,
	streak_key,
	MIN(date) AS streak_start,
	MAX(date) AS streak_end,
	COUNT(*) AS streak_length,
	ROUND(AVG(count_sessions), 2) AS avg_daily_sessions
FROM users_streak_keys
GROUP BY user_id, streak_key
HAVING COUNT(*) >= 5
ORDER BY streak_length DESC, avg_daily_sessions DESC


Completed after a long struggle, but I definitely don't feel strong with this pattern AND I'M FEELING like it looks a bit weird e.g. streak_keys differ from the actual streak_starts and I wouldn't trust this with 100% trust score.

---

## Task 3: Category Revenue Ranking with Rolling Comparison

**Scenario:**
The product team wants a monthly revenue leaderboard by product category.

For each month and category, calculate:
- Total revenue from that category in that month (`quantity × price`)
- The category's rank that month (by revenue, highest first)
- Revenue from the same category in the previous month (LAG)
- Month-over-month revenue change (current − previous), NULL if no previous month

Show only months and categories where total revenue > 0.

**Expected Output Columns:**
- `month` (date) — truncated to month
- `category_name` (text)
- `monthly_revenue` (numeric) — total revenue, rounded to 2 decimals
- `revenue_rank` (bigint) — rank within that month (1 = highest revenue)
- `prev_month_revenue` (numeric) — previous month's revenue for same category, rounded to 2 decimals, NULL if none
- `mom_change` (numeric) — month-over-month change, rounded to 2 decimals, NULL if no previous month

**Requirements:**
- Use `orders`, `orders_products`, `products`, `product_categories`
- Revenue = `orders_products.quantity × products.price`
- Order by `month ASC`, `revenue_rank ASC`

**Difficulty Rating:** 4/5


WITH categories_monthly_revenues AS (
SELECT 
	DATE_TRUNC('Month', o.created_at) AS month_,
	pc."name" AS category_name,
	SUM(p.price * op.quantity) AS monthly_revenue
FROM orders_products op
JOIN orders o ON op.order_id = o.id
JOIN products p ON op.product_id = p.id
JOIN product_categories pc ON p.category_id = pc.id
GROUP BY DATE_TRUNC('Month', o.created_at), pc.name
)
SELECT 
	month_,
	category_name,
	monthly_revenue,
	RANK() OVER (PARTITION BY month_ ORDER BY monthly_revenue DESC) AS revenue_rank,
	COALESCE(LAG(monthly_revenue) OVER (PARTITION BY category_name ORDER BY month_), NULL) AS prev_month_revenue,
	monthly_revenue - COALESCE(LAG(monthly_revenue) OVER (PARTITION BY category_name ORDER BY month_), NULL) AS mom_change
FROM categories_monthly_revenues


Your requirements were met in this case and the order also matches as I've checked everything. I could add one more CTE for more clarity, but I decieded to make it the most efficient instead and did one-liner with mom_change.

It all works and definitely satisfies all the requirements

---

## Submission Instructions

1. Task 1 — 3-level delivery hierarchy (3/5)
2. Task 2 — User session streaks (5/5)
3. Task 3 — Category revenue ranking with rolling comparison (4/5)

### Task Archive: 2026-02-18 (Week 10, Day 3)
# Daily SQL Practice Tasks

**Generated:** 2026-02-17
**Week 10, Day 2 Focus:** Gaps-and-Islands, Session Engagement Analysis, Product Category Ranking

---

## Task 1: 3-Level Hierarchy — Delivery Statuses by Month

**Scenario:**
Build a 3-level hierarchy over delivery data:
- Level 1: `'All Deliveries'`
- Level 2: Distinct delivery statuses from the `deliveries` table (pulled dynamically, no hardcoded VALUES)
- Level 3: The 3 most recent deliveries per status (show delivery ID as text)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — status name at Level 2, delivery ID at Level 3
- `parent_name` (text)
- `path` (text) — full path from root

**Requirements:**
- Pre-aggregate distinct statuses and top-3-per-status before the recursive CTE
- Use `created_at DESC` to define "most recent"
- Do not forget the termination condition
- Do not hardcode status values

**Difficulty Rating:** 3/5

WITH RECURSIVE delivery_ranks AS (
SELECT 
	*,
	DENSE_RANK() OVER (PARTITION BY status ORDER BY created_at DESC)
FROM deliveries
),
three_recent_deliveries_by_status AS (
SELECT 
	*
FROM delivery_ranks
WHERE DENSE_RANK <= 3
),
distinct_statuses AS (
SELECT DISTINCT status FROM deliveries
),
HIERARCHY AS (
SELECT
	1 AS level,
	'All Deliveries' AS name,
	NULL::TEXT AS parent_name,
	'All Deliveries' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(ds.status, trd.id::TEXT),
	h.name,
	h.PATH || ' > ' || COALESCE(ds.status, trd.id::TEXT)
FROM HIERARCHY h
LEFT JOIN distinct_statuses ds ON h.LEVEL = 1
LEFT JOIN three_recent_deliveries_by_status trd ON trd.status = h.name
WHERE h.LEVEL < 3
)
SELECT * FROM HIERARCHY

---

## Task 2: User Session Streaks (Gaps-and-Islands)

**Scenario:**
The engagement team wants to identify "power users" — users with long streaks of consecutive days where they had at least one session (`count_sessions > 0`).

Find users whose longest active streak is at least 5 consecutive days. For each qualifying user, show their longest streak only.

**Expected Output Columns:**
- `user_id` (integer)
- `streak_start` (date)
- `streak_end` (date)
- `streak_length` (bigint) — number of consecutive active days
- `avg_daily_sessions` (numeric) — average `count_sessions` during the streak, rounded to 2 decimals

**Requirements:**
- Use `user_sessions_daily` table
- Active day = `count_sessions > 0`
- A streak is a sequence of consecutive calendar days with no gaps
- If a user has multiple streaks of equal max length, show the most recent one
- Order by `streak_length DESC`, `avg_daily_sessions DESC`

**Hint:** The gaps-and-islands pattern — subtract `ROW_NUMBER()` from the date to create a streak group key. Dates within the same streak will share the same group key.

**Difficulty Rating:** 5/5


WITH users_dates AS (
SELECT 
	user_id,
	date,
	LAG(date) OVER (PARTITION BY user_id ORDER BY date) AS prev_session_date,
	count_sessions
FROM user_sessions_daily usd
ORDER BY user_id
),
users_dates_rn AS (
SELECT 
	*,
	ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY date) AS rn
FROM users_dates
WHERE prev_session_date IS NOT NULL
),
users_streak_keys AS (
SELECT 
	*,
	date - rn * INTERVAL '1' DAY AS streak_key
FROM users_dates_rn
)
SELECT 
	user_id,
	streak_key,
	MIN(date) AS streak_start,
	MAX(date) AS streak_end,
	COUNT(*) AS streak_length,
	ROUND(AVG(count_sessions), 2) AS avg_daily_sessions
FROM users_streak_keys
GROUP BY user_id, streak_key
HAVING COUNT(*) >= 5
ORDER BY streak_length DESC, avg_daily_sessions DESC


Completed after a long struggle, but I definitely don't feel strong with this pattern AND I'M FEELING like it looks a bit weird e.g. streak_keys differ from the actual streak_starts and I wouldn't trust this with 100% trust score.

---

## Task 3: Category Revenue Ranking with Rolling Comparison

**Scenario:**
The product team wants a monthly revenue leaderboard by product category.

For each month and category, calculate:
- Total revenue from that category in that month (`quantity × price`)
- The category's rank that month (by revenue, highest first)
- Revenue from the same category in the previous month (LAG)
- Month-over-month revenue change (current − previous), NULL if no previous month

Show only months and categories where total revenue > 0.

**Expected Output Columns:**
- `month` (date) — truncated to month
- `category_name` (text)
- `monthly_revenue` (numeric) — total revenue, rounded to 2 decimals
- `revenue_rank` (bigint) — rank within that month (1 = highest revenue)
- `prev_month_revenue` (numeric) — previous month's revenue for same category, rounded to 2 decimals, NULL if none
- `mom_change` (numeric) — month-over-month change, rounded to 2 decimals, NULL if no previous month

**Requirements:**
- Use `orders`, `orders_products`, `products`, `product_categories`
- Revenue = `orders_products.quantity × products.price`
- Order by `month ASC`, `revenue_rank ASC`

**Difficulty Rating:** 4/5


WITH categories_monthly_revenues AS (
SELECT 
	DATE_TRUNC('Month', o.created_at) AS month_,
	pc."name" AS category_name,
	SUM(p.price * op.quantity) AS monthly_revenue
FROM orders_products op
JOIN orders o ON op.order_id = o.id
JOIN products p ON op.product_id = p.id
JOIN product_categories pc ON p.category_id = pc.id
GROUP BY DATE_TRUNC('Month', o.created_at), pc.name
)
SELECT 
	month_,
	category_name,
	monthly_revenue,
	RANK() OVER (PARTITION BY month_ ORDER BY monthly_revenue DESC) AS revenue_rank,
	COALESCE(LAG(monthly_revenue) OVER (PARTITION BY category_name ORDER BY month_), NULL) AS prev_month_revenue,
	monthly_revenue - COALESCE(LAG(monthly_revenue) OVER (PARTITION BY category_name ORDER BY month_), NULL) AS mom_change
FROM categories_monthly_revenues


Your requirements were met in this case and the order also matches as I've checked everything. I could add one more CTE for more clarity, but I decieded to make it the most efficient instead and did one-liner with mom_change.

It all works and definitely satisfies all the requirements

---

## Submission Instructions

1. Task 1 — 3-level delivery hierarchy (3/5)
2. Task 2 — User session streaks (5/5)
3. Task 3 — Category revenue ranking with rolling comparison (4/5)

### Task Archive: 2026-02-18 (Week 10, Day 3)
# Daily SQL Practice Tasks

**Generated:** 2026-02-17
**Week 10, Day 2 Focus:** Gaps-and-Islands, Session Engagement Analysis, Product Category Ranking

---

## Task 1: 3-Level Hierarchy — Delivery Statuses by Month

**Scenario:**
Build a 3-level hierarchy over delivery data:
- Level 1: `'All Deliveries'`
- Level 2: Distinct delivery statuses from the `deliveries` table (pulled dynamically, no hardcoded VALUES)
- Level 3: The 3 most recent deliveries per status (show delivery ID as text)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — status name at Level 2, delivery ID at Level 3
- `parent_name` (text)
- `path` (text) — full path from root

**Requirements:**
- Pre-aggregate distinct statuses and top-3-per-status before the recursive CTE
- Use `created_at DESC` to define "most recent"
- Do not forget the termination condition
- Do not hardcode status values

**Difficulty Rating:** 3/5

WITH RECURSIVE delivery_ranks AS (
SELECT 
	*,
	DENSE_RANK() OVER (PARTITION BY status ORDER BY created_at DESC)
FROM deliveries
),
three_recent_deliveries_by_status AS (
SELECT 
	*
FROM delivery_ranks
WHERE DENSE_RANK <= 3
),
distinct_statuses AS (
SELECT DISTINCT status FROM deliveries
),
HIERARCHY AS (
SELECT
	1 AS level,
	'All Deliveries' AS name,
	NULL::TEXT AS parent_name,
	'All Deliveries' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(ds.status, trd.id::TEXT),
	h.name,
	h.PATH || ' > ' || COALESCE(ds.status, trd.id::TEXT)
FROM HIERARCHY h
LEFT JOIN distinct_statuses ds ON h.LEVEL = 1
LEFT JOIN three_recent_deliveries_by_status trd ON trd.status = h.name
WHERE h.LEVEL < 3
)
SELECT * FROM HIERARCHY

---

## Task 2: User Session Streaks (Gaps-and-Islands)

**Scenario:**
The engagement team wants to identify "power users" — users with long streaks of consecutive days where they had at least one session (`count_sessions > 0`).

Find users whose longest active streak is at least 5 consecutive days. For each qualifying user, show their longest streak only.

**Expected Output Columns:**
- `user_id` (integer)
- `streak_start` (date)
- `streak_end` (date)
- `streak_length` (bigint) — number of consecutive active days
- `avg_daily_sessions` (numeric) — average `count_sessions` during the streak, rounded to 2 decimals

**Requirements:**
- Use `user_sessions_daily` table
- Active day = `count_sessions > 0`
- A streak is a sequence of consecutive calendar days with no gaps
- If a user has multiple streaks of equal max length, show the most recent one
- Order by `streak_length DESC`, `avg_daily_sessions DESC`

**Hint:** The gaps-and-islands pattern — subtract `ROW_NUMBER()` from the date to create a streak group key. Dates within the same streak will share the same group key.

**Difficulty Rating:** 5/5


WITH users_dates AS (
SELECT 
	user_id,
	date,
	LAG(date) OVER (PARTITION BY user_id ORDER BY date) AS prev_session_date,
	count_sessions
FROM user_sessions_daily usd
ORDER BY user_id
),
users_dates_rn AS (
SELECT 
	*,
	ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY date) AS rn
FROM users_dates
WHERE prev_session_date IS NOT NULL
),
users_streak_keys AS (
SELECT 
	*,
	date - rn * INTERVAL '1' DAY AS streak_key
FROM users_dates_rn
)
SELECT 
	user_id,
	streak_key,
	MIN(date) AS streak_start,
	MAX(date) AS streak_end,
	COUNT(*) AS streak_length,
	ROUND(AVG(count_sessions), 2) AS avg_daily_sessions
FROM users_streak_keys
GROUP BY user_id, streak_key
HAVING COUNT(*) >= 5
ORDER BY streak_length DESC, avg_daily_sessions DESC


Completed after a long struggle, but I definitely don't feel strong with this pattern AND I'M FEELING like it looks a bit weird e.g. streak_keys differ from the actual streak_starts and I wouldn't trust this with 100% trust score.

---

## Task 3: Category Revenue Ranking with Rolling Comparison

**Scenario:**
The product team wants a monthly revenue leaderboard by product category.

For each month and category, calculate:
- Total revenue from that category in that month (`quantity × price`)
- The category's rank that month (by revenue, highest first)
- Revenue from the same category in the previous month (LAG)
- Month-over-month revenue change (current − previous), NULL if no previous month

Show only months and categories where total revenue > 0.

**Expected Output Columns:**
- `month` (date) — truncated to month
- `category_name` (text)
- `monthly_revenue` (numeric) — total revenue, rounded to 2 decimals
- `revenue_rank` (bigint) — rank within that month (1 = highest revenue)
- `prev_month_revenue` (numeric) — previous month's revenue for same category, rounded to 2 decimals, NULL if none
- `mom_change` (numeric) — month-over-month change, rounded to 2 decimals, NULL if no previous month

**Requirements:**
- Use `orders`, `orders_products`, `products`, `product_categories`
- Revenue = `orders_products.quantity × products.price`
- Order by `month ASC`, `revenue_rank ASC`

**Difficulty Rating:** 4/5


WITH categories_monthly_revenues AS (
SELECT 
	DATE_TRUNC('Month', o.created_at) AS month_,
	pc."name" AS category_name,
	SUM(p.price * op.quantity) AS monthly_revenue
FROM orders_products op
JOIN orders o ON op.order_id = o.id
JOIN products p ON op.product_id = p.id
JOIN product_categories pc ON p.category_id = pc.id
GROUP BY DATE_TRUNC('Month', o.created_at), pc.name
)
SELECT 
	month_,
	category_name,
	monthly_revenue,
	RANK() OVER (PARTITION BY month_ ORDER BY monthly_revenue DESC) AS revenue_rank,
	COALESCE(LAG(monthly_revenue) OVER (PARTITION BY category_name ORDER BY month_), NULL) AS prev_month_revenue,
	monthly_revenue - COALESCE(LAG(monthly_revenue) OVER (PARTITION BY category_name ORDER BY month_), NULL) AS mom_change
FROM categories_monthly_revenues


Your requirements were met in this case and the order also matches as I've checked everything. I could add one more CTE for more clarity, but I decieded to make it the most efficient instead and did one-liner with mom_change.

It all works and definitely satisfies all the requirements

---

## Submission Instructions

1. Task 1 — 3-level delivery hierarchy (3/5)
2. Task 2 — User session streaks (5/5)
3. Task 3 — Category revenue ranking with rolling comparison (4/5)
n### Task Archive: 2026-02-18 (Week 10, Day 2)n
# Daily SQL Practice Tasks

**Generated:** 2026-02-17
**Week 10, Day 2 Focus:** Gaps-and-Islands, Session Engagement Analysis, Product Category Ranking

---

## Task 1: 3-Level Hierarchy — Delivery Statuses by Month

**Scenario:**
Build a 3-level hierarchy over delivery data:
- Level 1: `'All Deliveries'`
- Level 2: Distinct delivery statuses from the `deliveries` table (pulled dynamically, no hardcoded VALUES)
- Level 3: The 3 most recent deliveries per status (show delivery ID as text)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — status name at Level 2, delivery ID at Level 3
- `parent_name` (text)
- `path` (text) — full path from root

**Requirements:**
- Pre-aggregate distinct statuses and top-3-per-status before the recursive CTE
- Use `created_at DESC` to define "most recent"
- Do not forget the termination condition
- Do not hardcode status values

**Difficulty Rating:** 3/5

WITH RECURSIVE delivery_ranks AS (
SELECT 
	*,
	DENSE_RANK() OVER (PARTITION BY status ORDER BY created_at DESC)
FROM deliveries
),
three_recent_deliveries_by_status AS (
SELECT 
	*
FROM delivery_ranks
WHERE DENSE_RANK <= 3
),
distinct_statuses AS (
SELECT DISTINCT status FROM deliveries
),
HIERARCHY AS (
SELECT
	1 AS level,
	'All Deliveries' AS name,
	NULL::TEXT AS parent_name,
	'All Deliveries' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(ds.status, trd.id::TEXT),
	h.name,
	h.PATH || ' > ' || COALESCE(ds.status, trd.id::TEXT)
FROM HIERARCHY h
LEFT JOIN distinct_statuses ds ON h.LEVEL = 1
LEFT JOIN three_recent_deliveries_by_status trd ON trd.status = h.name
WHERE h.LEVEL < 3
)
SELECT * FROM HIERARCHY

---

## Task 2: User Session Streaks (Gaps-and-Islands)

**Scenario:**
The engagement team wants to identify "power users" — users with long streaks of consecutive days where they had at least one session (`count_sessions > 0`).

Find users whose longest active streak is at least 5 consecutive days. For each qualifying user, show their longest streak only.

**Expected Output Columns:**
- `user_id` (integer)
- `streak_start` (date)
- `streak_end` (date)
- `streak_length` (bigint) — number of consecutive active days
- `avg_daily_sessions` (numeric) — average `count_sessions` during the streak, rounded to 2 decimals

**Requirements:**
- Use `user_sessions_daily` table
- Active day = `count_sessions > 0`
- A streak is a sequence of consecutive calendar days with no gaps
- If a user has multiple streaks of equal max length, show the most recent one
- Order by `streak_length DESC`, `avg_daily_sessions DESC`

**Hint:** The gaps-and-islands pattern — subtract `ROW_NUMBER()` from the date to create a streak group key. Dates within the same streak will share the same group key.

**Difficulty Rating:** 5/5


WITH users_dates AS (
SELECT 
	user_id,
	date,
	LAG(date) OVER (PARTITION BY user_id ORDER BY date) AS prev_session_date,
	count_sessions
FROM user_sessions_daily usd
ORDER BY user_id
),
users_dates_rn AS (
SELECT 
	*,
	ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY date) AS rn
FROM users_dates
WHERE prev_session_date IS NOT NULL
),
users_streak_keys AS (
SELECT 
	*,
	date - rn * INTERVAL '1' DAY AS streak_key
FROM users_dates_rn
)
SELECT 
	user_id,
	streak_key,
	MIN(date) AS streak_start,
	MAX(date) AS streak_end,
	COUNT(*) AS streak_length,
	ROUND(AVG(count_sessions), 2) AS avg_daily_sessions
FROM users_streak_keys
GROUP BY user_id, streak_key
HAVING COUNT(*) >= 5
ORDER BY streak_length DESC, avg_daily_sessions DESC


Completed after a long struggle, but I definitely don't feel strong with this pattern AND I'M FEELING like it looks a bit weird e.g. streak_keys differ from the actual streak_starts and I wouldn't trust this with 100% trust score.

---

## Task 3: Category Revenue Ranking with Rolling Comparison

**Scenario:**
The product team wants a monthly revenue leaderboard by product category.

For each month and category, calculate:
- Total revenue from that category in that month (`quantity × price`)
- The category's rank that month (by revenue, highest first)
- Revenue from the same category in the previous month (LAG)
- Month-over-month revenue change (current − previous), NULL if no previous month

Show only months and categories where total revenue > 0.

**Expected Output Columns:**
- `month` (date) — truncated to month
- `category_name` (text)
- `monthly_revenue` (numeric) — total revenue, rounded to 2 decimals
- `revenue_rank` (bigint) — rank within that month (1 = highest revenue)
- `prev_month_revenue` (numeric) — previous month's revenue for same category, rounded to 2 decimals, NULL if none
- `mom_change` (numeric) — month-over-month change, rounded to 2 decimals, NULL if no previous month

**Requirements:**
- Use `orders`, `orders_products`, `products`, `product_categories`
- Revenue = `orders_products.quantity × products.price`
- Order by `month ASC`, `revenue_rank ASC`

**Difficulty Rating:** 4/5


WITH categories_monthly_revenues AS (
SELECT 
	DATE_TRUNC('Month', o.created_at) AS month_,
	pc."name" AS category_name,
	SUM(p.price * op.quantity) AS monthly_revenue
FROM orders_products op
JOIN orders o ON op.order_id = o.id
JOIN products p ON op.product_id = p.id
JOIN product_categories pc ON p.category_id = pc.id
GROUP BY DATE_TRUNC('Month', o.created_at), pc.name
)
SELECT 
	month_,
	category_name,
	monthly_revenue,
	RANK() OVER (PARTITION BY month_ ORDER BY monthly_revenue DESC) AS revenue_rank,
	COALESCE(LAG(monthly_revenue) OVER (PARTITION BY category_name ORDER BY month_), NULL) AS prev_month_revenue,
	monthly_revenue - COALESCE(LAG(monthly_revenue) OVER (PARTITION BY category_name ORDER BY month_), NULL) AS mom_change
FROM categories_monthly_revenues


Your requirements were met in this case and the order also matches as I've checked everything. I could add one more CTE for more clarity, but I decieded to make it the most efficient instead and did one-liner with mom_change.

It all works and definitely satisfies all the requirements

---

## Submission Instructions

1. Task 1 — 3-level delivery hierarchy (3/5)
2. Task 2 — User session streaks (5/5)
3. Task 3 — Category revenue ranking with rolling comparison (4/5)
n### Task Archive: 2026-02-19 (Week 10, Day 4)n
# Daily SQL Practice Tasks

**Generated:** 2026-02-18
**Week 10, Day 3 Focus:** Gaps-and-Islands Mastery (Scaffolded) + Hierarchy + Rolling Windows

---

## Task 1: 3-Level Hierarchy — Product Categories and Top Products

**Scenario:**
Build a 3-level hierarchy over product data:
- Level 1: `'All Products'`
- Level 2: Distinct category names from `product_categories` (pulled dynamically)
- Level 3: Top 3 most expensive products per category (show product name as text)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — category name at Level 2, product name at Level 3
- `parent_name` (text)
- `path` (text) — full path from root

**Requirements:**
- Pre-aggregate distinct categories and top-3-per-category before the recursive CTE
- Use `price DESC` to define "most expensive"
- Termination condition required
- No hardcoded category values

**Difficulty Rating:** 3/5


WITH RECURSIVE products_categories_rank AS (
SELECT 
	p.name AS product_name,
	pc.name AS category_name,
	pc.id AS category_id,
	p.price,
	RANK() OVER (PARTITION BY category_id ORDER BY price DESC) AS category_price_rank
FROM products p JOIN product_categories pc ON p.category_id  = pc.id
),
distinct_categories AS (
SELECT 
	DISTINCT id AS category_id,
	name AS category_name
FROM product_categories
),
top_three_products_per_category AS (
SELECT 
	*
FROM products_categories_rank
WHERE category_price_rank <= 3
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Products' AS name,
	NULL::TEXT AS parent_name,
	'All Products' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(dc.category_name::TEXT, ttp.product_name::TEXT),
	h.name,
	h.path || ' > ' || COALESCE(dc.category_name::TEXT, ttp.product_name::TEXT)
FROM HIERARCHY h
LEFT JOIN distinct_categories dc ON h.LEVEL = 1
LEFT JOIN top_three_products_per_category ttp ON h.name = ttp.category_name AND h.LEVEL = 2
WHERE H.LEVEL < 3
)
SELECT * FROM hierarchy


Everything works properly.

---

## Task 2: Gaps-and-Islands — Scaffolded Drill (User Sessions)

This task is broken into 3 sub-questions that build the pattern incrementally. Complete each step before moving to the next.

### Step A — Generate the streak key
Using `user_sessions_daily`, write a query that:
- Filters to active days only (the table only contains active days, so no filter needed)
- Assigns a `ROW_NUMBER()` per user ordered by date
- Computes `streak_key` as `date - (rn * INTERVAL '1 day')`

Output: `user_id`, `date`, `count_sessions`, `rn`, `streak_key`

No aggregation yet — just show the raw rows with the computed key. Pick user_id = 1 to inspect visually.

**Expected insight:** Dates within the same consecutive streak share the same `streak_key`. A gap in dates causes `streak_key` to shift.

---

### Step B — Aggregate streaks
Using Step A as a CTE, GROUP BY `(user_id, streak_key)` to produce one row per streak.

Output: `user_id`, `streak_key`, `streak_start` (MIN date), `streak_end` (MAX date), `streak_length` (COUNT), `avg_daily_sessions` (AVG, rounded to 2 decimals)

No filtering yet — show all streaks for all users.

---

### Step C — Pick the longest streak per user
Using Step B as a CTE, add a final step that:
- Ranks streaks per user by `streak_length DESC`, then `streak_end DESC` (most recent if tied)
- Keeps only rank = 1 (longest streak per user)
- Filters to users whose longest streak is >= 5 days
- Orders by `streak_length DESC`, `avg_daily_sessions DESC`

Final output: `user_id`, `streak_start`, `streak_end`, `streak_length`, `avg_daily_sessions`

**This is the complete solution to Day 2 Task 2 — assembled step by step.**

**Difficulty Rating:** 4/5 (scaffolded, but you must write all three steps)

WITH users_sessions_streak_keys AS (
SELECT 
	*,
	ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY date) AS rn,
	date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY date) * INTERVAL '1' DAY) AS streak_key
FROM user_sessions_daily
),
users_session_streaks_consecutive_days AS (
SELECT 
	user_id,
	streak_key,
	MIN(date) AS streak_start,
	MAX(date) AS streak_end,
	COUNT(*) AS streak_length,
	ROUND(AVG(count_sessions), 2) AS avg_daily_sessions
FROM users_sessions_streak_keys
GROUP BY user_id, streak_key
ORDER BY user_id
),
users_streak_ranks AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY user_id ORDER BY streak_length DESC, streak_end DESC) AS streak_rank
FROM users_session_streaks_consecutive_days
)
SELECT 
	user_id,
	streak_start,
	streak_end,
	streak_length,
	avg_daily_sessions
FROM users_streak_ranks 
WHERE streak_rank = 1


Nice, it makes sense now - we definitely MUST practice this pattern more.

---

## Task 3: 7-Day Rolling Order Revenue

**Scenario:**
The finance team wants a daily rolling revenue report. For each day in the `dates` table (within the range of actual order data), calculate the total order revenue for the past 7 days (including that day).

**Expected Output Columns:**
- `date` (date)
- `daily_revenue` (numeric) — total order revenue on that specific day, rounded to 2 decimals (0 if no orders)
- `rolling_7d_revenue` (numeric) — sum of daily_revenue over the past 7 days including today, rounded to 2 decimals

**Requirements:**
- Use `dates` table as the spine (left join orders to it)
- Only include dates within the min/max range of `orders.created_at`
- Use a window frame: `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`
- Days with no orders should show 0, not NULL
- Order by `date ASC`

**Difficulty Rating:** 4/5

WITH daily_orders_revenues AS (
SELECT 
	d.date,
	COALESCE(COUNT(o.id), 0) AS order_cnt,
	COALESCE(SUM(o.amount), 0) AS orders_revenue
FROM dates d
LEFT JOIN orders o ON d."date" = DATE(o.created_at)
GROUP BY d.date
ORDER BY d.date
)
SELECT 
	date,
	orders_revenue AS daily_revenue,
	ROUND(SUM(orders_revenue) OVER (ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)::NUMERIC, 2) AS rolling_7d_revenue
FROM daily_orders_revenues

Here, exactly as you wanted.

---

## Submission Instructions

1. Task 1 — Product category hierarchy (3/5)
2. Task 2 — Gaps-and-islands scaffolded drill, Steps A + B + C (4/5)
3. Task 3 — 7-day rolling revenue (4/5)
n### Task Archive: 2026-02-20 (Week 10, Day 5)n
# Daily SQL Practice Tasks

**Generated:** 2026-02-19
**Week 10, Day 4 Focus:** Advanced Window Functions + Gaps-and-Islands Variant + Hierarchy Consolidation

---

## Task 1: 3-Level Hierarchy — Users by Country and City

**Scenario:**
Build a 3-level hierarchy over user location data:
- Level 1: `'All Users'`
- Level 2: Distinct countries from the `users` table (exclude NULLs)
- Level 3: Distinct cities within each country (exclude NULLs), show city name

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — country at Level 2, city at Level 3
- `parent_name` (text)
- `path` (text) — full path from root

**Requirements:**
- Pre-aggregate distinct countries and distinct country+city pairs before the recursive CTE
- Exclude NULL countries and NULL cities
- Termination condition required
- No hardcoded values

**Difficulty Rating:** 3/5

WITH RECURSIVE distinct_countries AS (
SELECT 
	DISTINCT country
FROM users u 
WHERE country IS NOT NULL
),
distinct_cities AS (
SELECT DISTINCT 
COUNTRY, CITY FROM users
WHERE city IS NOT NULL
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Users' AS name,
	NULL::TEXT AS parent_name,
	'All Users' AS PATH
UNION ALL
SELECT 
	h.LEVEL + 1,
	COALESCE(dc.country::TEXT, dcit.city::TEXT),
	h.name,
	h.PATH || ' > ' || COALESCE(dc.country::TEXT, dcit.city::TEXT)
FROM HIERARCHY h
LEFT JOIN distinct_countries dc ON h.LEVEL = 1
LEFT JOIN distinct_cities dcit ON h.LEVEL = 2 AND dcit.country = h.name
WHERE H.LEVEL < 3
)
SELECT * FROM HIERARCHY

---

## Task 2: Gaps-and-Islands — Transaction Dry Spells

**Scenario:**
The finance team wants to identify periods of inactivity — gaps between transactions for each user. Specifically, find users who had a gap of **at least 30 days** between two consecutive transactions.

For each such gap, show:

**Expected Output Columns:**
- `user_id` (integer)
- `last_transaction_date` (date) — date of the transaction before the gap
- `next_transaction_date` (date) — date of the transaction after the gap
- `gap_days` (integer) — number of days between the two transactions
- `longest_gap` (boolean) — true if this is the longest gap for that user, false otherwise

**Requirements:**
- Use `transactions` table
- Use `DATE(created_at)` to work at day granularity
- Only include gaps >= 30 days
- Order by `gap_days DESC`, `user_id ASC`

**Note:** This is a gaps problem, not islands — you're finding the spaces *between* data points, not grouping consecutive ones. LAG is the right tool here, not the RN-subtraction pattern.

**Difficulty Rating:** 4/5

WITH users_transactions AS (
SELECT 
 id AS transaction_id,
 user_id,
 DATE_TRUNC('Day', created_at) AS current_transaction_date,
 LAG(DATE_TRUNC('Day', created_at)) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_transaction_date
FROM transactions
),
users_gaps AS (
SELECT 
	*,
	EXTRACT('Day' FROM current_transaction_date - prev_transaction_date) AS gap_days,
	MAX(EXTRACT('Day' FROM current_transaction_date - prev_transaction_date)) OVER (PARTITION BY user_id) AS longest_gap
FROM users_transactions ns
WHERE prev_transaction_date IS NOT NULL
)
SELECT 
	user_id,
	transaction_id,
	current_transaction_date,
	prev_transaction_date,
	gap_days
FROM users_gaps
WHERE longest_gap = gap_days
AND gap_days >= 1
ORDER BY gap_days DESC, user_id ASC


Look, after this step It's clear to me THAT THERE ARE NO USERS WITH GAPS ABOVE 1 day - 1 day gap IS LITERALLY THE MAXIMUM in this dataset, so the best thing I could do is filter out users with 0 day gaps (there were a lot of them as well).

IMO I handled it well and adapted to available data.


---

## Task 3: Percentile Bands + Cumulative Share

**Scenario:**
The analytics team wants a transaction amount distribution report. Bucket transactions into percentile bands and show what share of total volume each band represents.

Classify each transaction into one of 4 quartile bands using `NTILE(4)`:
- Band 1: Bottom 25%
- Band 2: 25–50%
- Band 3: 50–75%
- Band 4: Top 25%

Then aggregate by band and show:

**Expected Output Columns:**
- `quartile_band` (integer) — 1 to 4
- `transaction_count` (bigint)
- `band_revenue` (numeric) — total amount in this band, rounded to 2 decimals
- `pct_of_total_revenue` (numeric) — this band's revenue as % of all revenue, rounded to 1 decimal
- `cumulative_revenue_pct` (numeric) — running cumulative % from band 1 to 4, rounded to 1 decimal

**Requirements:**
- Use `transactions` table, exclude NULL amounts
- Compute NTILE in a CTE, then aggregate
- Cumulative % must use a window SUM over the aggregated results
- Order by `quartile_band ASC`

**Difficulty Rating:** 4/5


WITH transactions_amount_quartiles AS (
SELECT 
	*,
	ntile(4) OVER (ORDER BY amount DESC) AS quartile_band
FROM transactions
),
quartile_bands_revenues AS (
SELECT
	quartile_band,
	COUNT(*) AS transaction_count,
	SUM(amount) AS band_revenue,
	(SELECT sum(amount) FROM transactions) AS total_revenue
FROM transactions_amount_quartiles
GROUP BY quartile_band
ORDER BY band_revenue DESC
)
SELECT 
	*,
	ROUND((band_revenue / total_revenue) * 100, 1) || ' %' AS pct_of_total_revenue,
	ROUND(SUM((band_revenue / total_revenue) * 100) OVER (ORDER BY quartile_band), 1) || '%' AS cumulative_revenue_pct
FROM quartile_bands_revenues


Done, your requirements satisfied with 100% effect :)).

---

## Submission Instructions

1. Task 1 — Users by country/city hierarchy (3/5)
2. Task 2 — Transaction dry spells / gaps (4/5)
3. Task 3 — Percentile bands + cumulative share (4/5)
n### Task Archive: 2026-02-23 (Week 11, Day 1)n
# Daily SQL Practice Tasks

**Generated:** 2026-02-20
**Week 10, Day 5 Focus:** HackerRank Hard Final Puzzles — Multi-Pattern Combinations

---

## Task 1: 3-Level Hierarchy — Order Status by User Segment

**Scenario:**
Build a 3-level hierarchy combining orders and deliveries:
- Level 1: `'All Orders'`
- Level 2: Distinct delivery statuses from the `deliveries` table (dynamic, no hardcoded values)
- Level 3: For each status, the 3 users with the most orders in that delivery status (show user_id as text)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — status at Level 2, user_id at Level 3
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Pre-aggregate distinct statuses and top-3-users-per-status before the recursive CTE
- Join `deliveries` → `orders` to get user_id per delivery
- Use order count to rank users within each status
- Termination condition required

**Difficulty Rating:** 4/5


WITH RECURSIVE order_statuses_cnt AS (
SELECT 
	o.user_id,
	d.status,
	COUNT(o.id) AS order_cnt
FROM orders o
JOIN deliveries d ON o.id = d.order_id
GROUP BY o.user_id, d.status 
ORDER BY order_cnt DESC
),
ranked_statuses AS (
SELECT 
	*,
	ROW_NUMBER() OVER (PARTITION BY status ORDER BY order_cnt DESC) AS order_rank
FROM order_statuses_cnt
),
top_three_users_per_order_status AS (
SELECT * 
FROM ranked_statuses
WHERE order_rank <= 3
),
distinct_delivery_statuses AS (
SELECT DISTINCT status FROM deliveries
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Orders' AS name,
	NULL::text AS parent_name,
	'All Orders' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(dds.status::TEXT, ttu.user_id::TEXT),
	h.name,
	h.PATH || ' > ' || COALESCE(dds.status::TEXT, ttu.user_id::TEXT)
FROM HIERARCHY h
LEFT JOIN distinct_delivery_statuses dds ON h.LEVEL = 1
LEFT JOIN top_three_users_per_order_status ttu ON h.LEVEL = 2 AND h.name = ttu.status
WHERE h.LEVEL < 3
)
SELECT * 
FROM hierarchy



---

## Task 2: Order Gap Analysis per User (Gaps Pattern on Orders)

**Scenario:**
The retention team wants to understand ordering cadence. For each user who has placed at least 2 orders, calculate the average and maximum number of days between consecutive orders.

Then classify users into cadence segments:
- `frequent`: avg_days_between_orders < 30
- `regular`: avg_days_between_orders between 30 and 90 (inclusive)
- `infrequent`: avg_days_between_orders > 90

**Expected Output Columns:**
- `user_id` (integer)
- `order_count` (bigint)
- `avg_days_between_orders` (numeric) — rounded to 1 decimal
- `max_days_between_orders` (integer)
- `cadence_segment` (text)

**Requirements:**
- Use `orders` table
- Use LAG to compute days between consecutive orders per user
- Exclude users with only 1 order (no gap to compute)
- Order by `avg_days_between_orders ASC`

**Difficulty Rating:** 4/5


WITH users_orders_cnt AS (
SELECT 
	user_id,
	COUNT(id) AS orders_cnt
FROM orders o
GROUP BY user_id
HAVING count(id) > 1
),
users_order_dates AS (
SELECT
	uoc.user_id,
	uoc.orders_cnt,
	DATE_TRUNC('Day', o.created_at) AS order_date,
	LAG(DATE_TRUNC('Day', o.created_at)) OVER (PARTITION BY o.user_id ORDER BY o.created_at) AS previous_order_date
FROM users_orders_cnt uoc
JOIN orders o ON uoc.user_id = o.user_id
),
users_day_gaps AS (
SELECT 
	user_id,
	orders_cnt,
	order_date,
	previous_order_date,
	EXTRACT('Day' FROM order_date - previous_order_date) AS days_gap
FROM users_order_dates
WHERE previous_order_date IS NOT NULL
),
users_avg_gaps AS (
SELECT 
	*,
	AVG(days_gap) OVER (PARTITION BY user_id) AS average_user_gap
FROM users_day_gaps
),
users_orders_metrics AS (
SELECT 
	user_id,
	orders_cnt,
	ROUND(average_user_gap, 1) AS avg_days_between_orders,
	MAX(days_gap) AS max_days_between_orders
FROM users_avg_gaps
GROUP BY user_id, orders_cnt, average_user_gap
)
SELECT 
	*,
	CASE
		WHEN avg_days_between_orders < 3 THEN 'frequent' 
		WHEN avg_days_between_orders >= 3 AND avg_days_between_orders <= 6 THEN 'regular' 
		WHEN avg_days_between_orders > 6 THEN 'infrequent' 
	END AS cadence_segment
FROM users_orders_metrics
ORDER BY avg_days_between_orders ASC
	

Here, again I've adjusted the cadence segments to match the reality of our data, as there was literally not a single user with an average gap above 20, so it wouldn't make any sense. As for the rest, I've adjusted everything to your needs - eliminated unnecessary users early in the query, then calculated gaps and assigned them to users based on window funcs and finally aggregated everything nicely with a final CASE WHEN statement to create the segments, with numbers matching the reality of our data.

---

## Task 3: The Friday Challenge — Power User Leaderboard

**Scenario:**
The growth team wants a comprehensive power user leaderboard combining session activity, order behavior, and transaction volume.

For each user, calculate:
- Total sessions across all days (`user_sessions_daily`)
- Total orders placed (`orders`)
- Total transaction amount (`transactions`)
- A composite score: `(total_sessions * 0.3) + (total_orders * 10) + (total_transaction_amount * 0.01)`, rounded to 2 decimals
- Their overall rank by composite score (highest first)
- Their percentile (using `PERCENT_RANK()`), rounded to 1 decimal, shown as a value between 0 and 100

Only include users who appear in **all three** tables (sessions, orders, transactions).

**Expected Output Columns:**
- `user_id` (integer)
- `total_sessions` (bigint)
- `total_orders` (bigint)
- `total_transaction_amount` (numeric) — rounded to 2 decimals
- `composite_score` (numeric)
- `rank` (bigint)
- `percentile` (numeric)

**Requirements:**
- Use `user_sessions_daily`, `orders`, `transactions`
- Aggregate each source separately in CTEs before joining
- Use INNER JOINs to enforce presence in all three tables
- Order by `rank ASC`

**Difficulty Rating:** 5/5

WITH users_session_cnt AS (
SELECT 
	user_id,
	SUM(count_sessions) AS total_sessions
FROM user_sessions_daily
GROUP BY user_id
),
users_orders_cnt AS (
SELECT 
	user_id,
	COUNT(*) AS total_orders
FROM orders
GROUP BY user_id
),
users_total_transaction_amts AS (
SELECT 
	user_id,
	SUM(amount) AS total_transaction_amount
FROM transactions t
GROUP BY user_id
),
users_combined_statistics AS (
SELECT 
	u.id AS user_id,
	COALESCE(usc.total_sessions, 0) AS total_sessions,
	COALESCE(uoc.total_orders, 0) AS total_orders,
	COALESCE(utt.total_transaction_amount, 0) AS total_transaction_amount,
	ROUND((usc.total_sessions * 0.3) + (uoc.total_orders * 10) + (utt.total_transaction_amount * 0.01), 2) AS composite_score
FROM users u
JOIN users_session_cnt usc ON u.id = usc.user_id
JOIN users_orders_cnt uoc ON u.id = uoc.user_id
JOIN users_total_transaction_amts utt ON u.id = utt.user_id
ORDER BY user_id
)
SELECT 
	*,
	RANK() OVER (ORDER BY composite_score DESC) AS rank,
	ROUND((PERCENT_RANK() OVER (ORDER BY composite_score)::NUMERIC * 100), 1)  AS percentile
FROM users_combined_statistics
ORDER BY RANK


What a cool task.
I instinctively started from LEFT JOINs and COALESCE to join all tables in the pre-final CTE and replaced NULLs with zeros, as I thought that would be the approach. I'm glad I've finally checked your requirements, especialyl that there were some other caveats - like the format of our percentile, which is kinda unusual, but also a great way to practice.



---

## Submission Instructions

1. Task 1 — Order status hierarchy (4/5)
2. Task 2 — Order gap analysis (4/5)
3. Task 3 — Power user leaderboard (5/5)
n### Task Archive: 2026-02-24 (Week 11, Day 2)n
# Daily SQL Practice Tasks

**Generated:** 2026-02-23
**Week 11, Day 1 Focus:** Warm-Up — Consolidation + Light Review

---

## Task 1: 3-Level Hierarchy — Transaction Types by Top Users

**Scenario:**
Build a 3-level hierarchy over transactions:
- Level 1: `'All Transactions'`
- Level 2: Distinct transaction types (dynamic, from `transactions`)
- Level 3: For each type, the 3 users with the highest total transaction amount (show user_id as text)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — type at Level 2, user_id at Level 3
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Pre-aggregate totals per user+type, then rank before the recursive CTE
- Termination condition required
- No hardcoded values

**Difficulty Rating:** 3/5


WITH RECURSIVE distinct_transaction_types AS (
SELECT DISTINCT TYPE FROM transactions
),
transaction_types_totals AS (
SELECT
	user_id,
	type,
	SUM(amount) AS total_transactions_amt
FROM transactions
GROUP BY user_id, TYPE
),
transaction_types_ranks AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY TYPE ORDER BY total_transactions_amt DESC) AS transaction_type_rank
FROM transaction_types_totals
),
transaction_types_top_three AS (
SELECT * FROM transaction_types_ranks
WHERE transaction_type_rank <= 3
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Transactions' AS name,
	NULL::TEXT AS parent_name,
	'All Transactions' AS PATH
UNION ALL
SELECT 
	H.LEVEL + 1,
	COALESCE(dtt.TYPE, ttt.user_id::TEXT),
	h.name,
	h.PATH || ' > ' || COALESCE(dtt.TYPE, ttt.user_id::TEXT)
FROM HIERARCHY h
LEFT JOIN distinct_transaction_types dtt ON h.LEVEL = 1
LEFT JOIN transaction_types_top_three ttt ON h.LEVEL = 2 AND ttt."type" = h."name"
WHERE h.LEVEL < 3
)
SELECT * FROM hierarchy


---

## Task 2: User Order Frequency Cohorts

**Scenario:**
The product team wants to understand how many orders users typically place. Group users into frequency buckets based on their total order count:

- `one_time`: exactly 1 order
- `occasional`: 2 to 4 orders
- `regular`: 5 to 9 orders
- `loyal`: 10 or more orders

For each bucket, show how many users fall in it and their average order value (avg of all order amounts for users in that bucket), rounded to 2 decimals.

**Expected Output Columns:**
- `frequency_bucket` (text)
- `user_count` (bigint)
- `avg_order_value` (numeric)

**Requirements:**
- Use `users` and `orders` tables
- Include users with 0 orders in `one_time`? No — only users who appear in `orders`
- Order by `user_count DESC`

**Difficulty Rating:** 3/5

WITH users_orders AS (
SELECT 
	user_id,
	COUNT(*) AS order_cnt
FROM orders
GROUP BY user_id
),
users_frequencies AS (
SELECT 
	*,
	CASE
		WHEN order_cnt = 1 THEN 'one_time'
		WHEN order_cnt > 1 AND order_cnt < 5 THEN 'occasional'
		WHEN order_cnt > 4 AND order_cnt < 10 THEN 'regular'
		WHEN order_cnt >= 10 THEN 'loyal'
	END AS frequency_bucket
FROM USERS_ORDERS
)
SELECT 
	uf.frequency_bucket,
	COUNT(uf.user_id) AS user_count,
	ROUND(AVG(o.amount)::numeric, 2) AS avg_order_value
FROM users_frequencies uf
JOIN orders o ON uf.user_id = o.user_id
GROUP BY uf.frequency_bucket
ORDER BY user_count DESC

There was literally no need to use users table, so I didn't do it and followed best practices.

---

## Task 3: Daily Session Trends with 3-Day Rolling Average

**Scenario:**
The analytics team wants a daily overview of platform engagement. For each day in the `user_sessions_daily` data, calculate:
- Total sessions across all users that day
- A 3-day rolling average of total sessions (current day + 2 preceding days), rounded to 1 decimal

**Expected Output Columns:**
- `date` (date)
- `total_daily_sessions` (bigint)
- `rolling_3d_avg` (numeric)

**Requirements:**
- Use `user_sessions_daily`
- Rolling average window: `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW`
- Order by `date ASC`

**Difficulty Rating:** 2/5

WITH dates_sessions AS (
SELECT 
	date,
	SUM(count_sessions) AS total_daily_sessions
FROM user_sessions_daily
GROUP BY date
)
SELECT 
	*,
	ROUND(AVG(total_daily_sessions) OVER (ORDER BY date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS rolling_avg_3d
FROM dates_sessions

Easy!

---

## Submission Instructions

1. Task 1 — Transaction type hierarchy (3/5)
2. Task 2 — Order frequency cohorts (3/5)
3. Task 3 — Daily session trends with rolling average (2/5)
n### Task Archive: 2026-02-25 (Week 11, Day 3)n
# Daily SQL Practice Tasks

**Generated:** 2026-02-24
**Week 11, Day 2 Focus:** HackerRank Hard — Correlated Subqueries + Advanced Aggregations + Hierarchy

---

## Task 1: 3-Level Hierarchy — Chat Ticket Priorities and Top Tickets

**Scenario:**
Build a 3-level hierarchy over support tickets:
- Level 1: `'All Tickets'`
- Level 2: Distinct priority levels from `chat_tickets` (dynamic)
- Level 3: For each priority, the 3 most recently updated tickets (show ticket ID as text)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — priority at Level 2, ticket ID at Level 3
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Pre-aggregate distinct priorities and top-3-per-priority before the recursive CTE
- Use `updated_at DESC` to define "most recently updated"
- Termination condition required

**Difficulty Rating:** 3/5

WITH RECURSIVE priorities AS (
SELECT 
DISTINCT priority 
FROM chat_tickets
),
recently_updated_tickets AS (
SELECT
	priority,
	created_at,
	id,
	RANK() OVER (PARTITION BY priority ORDER BY created_at DESC) AS recent_update_rank
FROM chat_tickets
),
top3_recently_updated_tickets AS (
SELECT 
* 
FROM recently_updated_tickets
WHERE recent_update_rank <= 3
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Tickets' AS name,
	NULL::TEXT AS parent_name,
	'All Tickets' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(pr.priority, t3r.id::TEXT),
	h.name,
	h.PATH || ' > ' || COALESCE(pr.priority, t3r.id::TEXT)
FROM HIERARCHY h
LEFT JOIN priorities pr ON h.LEVEL = 1
LEFT JOIN top3_recently_updated_tickets t3r ON h.name = t3r.priority AND h.LEVEL = 2
WHERE h.LEVEL < 3
)
SELECT * FROM hierarchy


---

## Task 2: Order Value Outliers (Correlated Subquery)

**Scenario:**
The analytics team wants to flag orders that are significantly above average for that user — specifically, orders where the amount is more than **1.5x the user's own average order value**.

For each such outlier order, show:

**Expected Output Columns:**
- `order_id` (integer)
- `user_id` (integer)
- `order_amount` (numeric) — rounded to 2 decimals
- `user_avg_order_value` (numeric) — that user's average order amount, rounded to 2 decimals
- `ratio` (numeric) — order_amount / user_avg_order_value, rounded to 2 decimals

**Requirements:**
- Use `orders` table only
- Exclude NULL amounts
- A correlated subquery OR a CTE-based approach is both acceptable — choose what feels right
- Order by `ratio DESC`

**Difficulty Rating:** 4/5

WITH avg_users_orders AS (
SELECT
	user_id,
	ROUND(AVG(amount)::NUMERIC, 2) AS user_avg_order_value
FROM ORDERS
GROUP BY user_id
),
users_avg_orders_comparison AS (
SELECT 
	auo.user_id,
	auo.user_avg_order_value,
	o.id AS order_id,
	o.amount AS order_amount,
	ROUND(o.amount::NUMERIC / auo.user_avg_order_value, 2) AS ratio
FROM avg_users_orders auo
JOIN orders o ON auo.user_id = o.user_id
)
SELECT * FROM users_avg_orders_comparison
WHERE ratio > 1.5
ORDER BY ratio DESC

Please mind THAT excluding null amounts is pointless, as every order has an amount.

---

## Task 3: Message Response Time Analysis

**Scenario:**
The support team wants to understand how quickly agents respond to users within each ticket. For each ticket, find the first user message and the first agent response after it, then calculate the response time in minutes.

A user message has `author_id IS NULL` (sent by the client).
An agent message has `user_id IS NULL` (sent by the agent).

**Expected Output Columns:**
- `ticket_id` (bigint)
- `first_user_message_at` (timestamp)
- `first_agent_response_at` (timestamp) — first agent message AFTER the first user message, NULL if none
- `response_time_minutes` (numeric) — minutes between the two, rounded to 1 decimal, NULL if no response

**Requirements:**
- Use `chat_messages` table
- First user message = earliest message where `author_id IS NULL`
- First agent response = earliest message where `user_id IS NULL` AND `created_at > first_user_message_at`
- Order by `response_time_minutes ASC NULLS LAST`

**Difficulty Rating:** 5/5
WITH users_first_msg AS (
SELECT 
	ticket_id,
	MIN(created_at) AS first_user_message_at
FROM chat_messages
WHERE message_type = 'text' AND author_id IS NULL
GROUP BY ticket_id
),
agents_response_times AS (
SELECT 
	ticket_id,
	MIN(created_at) AS first_agent_response_at
FROM chat_messages
WHERE message_type = 'text' AND USER_ID IS NULL
GROUP BY ticket_id
)
SELECT 
	ufm.ticket_id,
	ufm.first_user_message_at,
	art.first_agent_response_at,
	EXTRACT('Minute' FROM art.first_agent_response_at - ufm.first_user_message_at) AS response_time_minutes
FROM users_first_msg ufm
JOIN agents_response_times art ON ufm.ticket_id = art.ticket_id
ORDER BY response_time_minutes ASC


Again, there were no nulls SO I OMITTED the NULLS LAST in order by, as it was pointless.

---

## Submission Instructions

1. Task 1 — Chat ticket priority hierarchy (3/5)
2. Task 2 — Order value outliers (4/5)
3. Task 3 — Message response time analysis (5/5)
n### Task Archive: 2026-02-26 (Week 11, Day 4)n
# Daily SQL Practice Tasks

**Generated:** 2026-02-25
**Week 11, Day 3 Focus:** HackerRank Hard — Multi-CTE Combinations + EPOCH Practice + Hierarchy

---

## Task 1: 3-Level Hierarchy — Users by Registration Month and Country

**Scenario:**
Build a 3-level hierarchy over user registration data:
- Level 1: `'All Users'`
- Level 2: Distinct registration months (format: `YYYY-MM`, pulled dynamically from `users.created_at`)
- Level 3: For each month, the 3 countries with the most registrations (show country name, exclude NULLs)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — month string at Level 2, country at Level 3
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Pre-aggregate distinct months and top-3-countries-per-month before the recursive CTE
- Format months as text using `TO_CHAR(created_at, 'YYYY-MM')`
- Exclude NULL countries at Level 3
- Termination condition required

**Difficulty Rating:** 4/5

WITH RECURSIVE distinct_registration_months AS (
SELECT 
	DISTINCT date_trunc('Month', created_at) AS registration_month
FROM users
ORDER BY registration_month
),
countries_registrations AS (
SELECT 
	date_trunc('Month', created_at) AS registration_month,
	country,
	COUNT(*) AS registered_users_cnt
FROM users
WHERE country IS NOT NULL
GROUP BY date_trunc('Month', created_at), country
ORDER BY registration_month
),
monthly_registration_rank AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY registration_month ORDER BY registered_users_cnt DESC) AS registration_rank
FROM countries_registrations
),
top_three_countries_registration_rank AS (
SELECT * FROM monthly_registration_rank
WHERE registration_rank <= 3
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Users'::TEXT AS name,
	NULL::TEXT AS parent_name,
	'All Users' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(TO_CHAR(drm.registration_month, 'YYYY-MM'), ttcr.country::TEXT),
	h.name,
	h.PATH || ' > ' || COALESCE(TO_CHAR(drm.registration_month, 'YYYY-MM'), ttcr.country::TEXT)
FROM HIERARCHY h
LEFT JOIN distinct_registration_months drm ON h.LEVEL = 1
LEFT JOIN top_three_countries_registration_rank ttcr ON h.LEVEL = 2 AND h.name = TO_CHAR(ttcr.registration_month, 'YYYY-MM')
WHERE h.LEVEL < 3
)
SELECT * FROM hierarchy


---

## Task 2: Ticket Resolution Time Analysis (EPOCH Practice)

**Scenario:**
The support team wants to measure how long it takes to resolve tickets. For each resolved ticket (status = `'resolved'`), calculate the time between creation and last update (as a proxy for resolution time).

Then segment tickets by resolution speed:
- `fast`: resolved in under 1 hour
- `medium`: 1 to 24 hours
- `slow`: more than 24 hours

**Expected Output Columns:**
- `ticket_id` (bigint)
- `priority` (text)
- `resolution_hours` (numeric) — hours between created_at and updated_at, rounded to 2 decimals
- `resolution_segment` (text)

**Requirements:**
- Use `chat_tickets` table
- Only include tickets where `status = 'resolved'`
- Use `EXTRACT(EPOCH FROM ...)` to calculate the interval in seconds, then convert to hours
- Order by `resolution_hours ASC`

**Difficulty Rating:** 3/5

WITH resolved_tickets_resolUtion_times AS (
SELECT 
  cm.ticket_id,
  ct.created_at AS ticket_creation_time,
  cm.created_at AS ticket_resolve_time,
  EXTRACT('Epoch' FROM cm.created_at - ct.created_at) / 60 AS resolution_minutes
FROM chat_messages cm
JOIN chat_tickets ct ON cm.ticket_id = ct.id
WHERE cm.status = 'resolved'
)
SELECT 
	*,
	CASE 
		WHEN resolution_minutes <= 5 THEN 'fast'
		WHEN resolution_minutes <= 10 THEN 'medium'
		WHEN resolution_minutes > 10 THEN 'slow'
	END AS resolution_segment
FROM resolved_tickets_resolUtion_times
ORDER BY resolution_minutes

Adapted this to the acutal data, where 26 MINUTES was the maximum amount, so I adapted the resolution to minutes and used relevant values as filters for fast/medium/slow.


---

## Task 3: Product Affinity — Frequently Co-Purchased Products

**Scenario:**
The recommendations team wants to find product pairs that are frequently bought together in the same order. Find all pairs of products that appear together in at least 2 orders.

**Expected Output Columns:**
- `product_a_id` (integer)
- `product_b_id` (integer)
- `product_a_name` (text)
- `product_b_name` (text)
- `co_purchase_count` (bigint) — number of orders containing both products
- `co_purchase_rank` (bigint) — rank by co_purchase_count DESC

**Requirements:**
- Use `orders_products` and `products` tables
- A pair is defined as `product_a_id < product_b_id` to avoid duplicates
- Only include pairs appearing in >= 2 orders
- Order by `co_purchase_count DESC`, `product_a_id ASC`

**Difficulty Rating:** 5/5

There's a lot of co purchase count 3 and 2, so I used row number to actually rank them based on co_purchase_count and product_a_id ASC, as otherwise it wouldn't make sense :))

WITH orders_product_pairs AS (
SELECT 
	op1.product_id AS product_a_id,
	op2.product_id AS product_b_id,
	p1."name" AS product_a_name,
	p2.name AS product_b_name,
	COUNT(op1.order_id) AS co_purchase_count
FROM orders_products op1
JOIN orders_products op2 ON op1.order_id = op2.order_id
JOIN products p1 ON op1.product_id = p1.id
JOIN products p2 ON op2.product_id = p2.id
WHERE op1.product_id > op2.product_id
GROUP BY op1.product_id, op2.product_id, p1."name", p2."name"
ORDER BY co_purchase_count DESC, product_a_id
)
SELECT 
	*,
	ROW_NUMBER() OVER (ORDER BY co_purchase_count DESC)
FROM orders_product_pairs
WHERE co_purchase_count >= 2

---

## Submission Instructions

1. Task 1 — Registration month/country hierarchy (4/5)
2. Task 2 — Ticket resolution time with EPOCH (3/5)
3. Task 3 — Product affinity pairs (5/5)
n### Task Archive: 2026-02-27 (Week 11, Day 5)n
# Daily SQL Practice Tasks

**Generated:** 2026-02-26
**Week 11, Day 4 Focus:** HackerRank Hard — Session Outliers + Multi-Table Aggregation + Hierarchy

---

## Task 1: 3-Level Hierarchy — Product Categories, Products, and Order Count

**Scenario:**
Build a 3-level hierarchy over product sales:
- Level 1: `'All Categories'`
- Level 2: Distinct category names (dynamic, from `product_categories`)
- Level 3: For each category, the 3 products with the highest total quantity sold (show product name)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — category name at Level 2, product name at Level 3
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Pre-aggregate total quantity sold per product before the recursive CTE
- Use `orders_products` and `products` and `product_categories`
- Termination condition required

**Difficulty Rating:** 4/5


WITH RECURSIVE product_sold_amt AS (
SELECT 
	op.product_id,
	p."name" AS product_name,
	p.category_id AS category_id,
	pc.name AS category_name,
	SUM(op.quantity) AS total_quantity_sold
FROM orders_products op
JOIN products p ON op.product_id = p.id
JOIN product_categories pc ON pc.id = p.category_id
GROUP BY op.product_id, p."name", p.category_id, pc.name
),
product_sales_ranking AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY category_id ORDER BY total_quantity_sold DESC) AS product_sales_rank
FROM product_sold_amt
),
top_three_products AS (
SELECT * FROM product_sales_ranking
WHERE product_sales_rank <= 3
),
distinct_categories AS (
SELECT DISTINCT id, name FROM product_categories PC
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Categories' AS name,
	NULL::TEXT AS parent_name,
	'All Categories' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(dc.name::TEXT, ttp.product_name),
	h.name,
	h.PATH || ' > ' || COALESCE(dc.name::TEXT, ttp.product_name)
FROM HIERARCHY h
LEFT JOIN distinct_categories dc ON h.LEVEL = 1
LEFT JOIN top_three_products ttp ON h.LEVEL = 2 AND h.name = ttp.category_name
WHERE h.LEVEL < 3
)
SELECT * FROM hierarchy


---

## Task 2: User Session Outlier Days

**Scenario:**
The engagement team wants to identify days where a user's session count was unusually high — specifically, days where their session count was more than **2 standard deviations above their own mean**.

For each such outlier day, show:

**Expected Output Columns:**
- `user_id` (integer)
- `date` (date)
- `count_sessions` (integer)
- `user_avg_sessions` (numeric) — that user's average daily sessions, rounded to 2 decimals
- `user_stddev_sessions` (numeric) — that user's standard deviation of daily sessions, rounded to 2 decimals
- `z_score` (numeric) — (count_sessions - user_avg) / user_stddev, rounded to 2 decimals

**Requirements:**
- Use `user_sessions_daily`
- Use `AVG()` and `STDDEV()` as window functions (no GROUP BY needed)
- Exclude rows where stddev = 0 (user has identical session count every day — no outliers possible)
- Order by `z_score DESC`

**Difficulty Rating:** 4/5

WITH users_sessions_metrics AS (
SELECT
	user_id,
	ROUND(STDDEV(count_sessions), 2) AS user_stddev_sessions,
	ROUND(AVG(count_sessions), 2) AS avg_user_daily_sessions
FROM user_sessions_daily usd 
GROUP BY user_id
)
SELECT 
	usd.user_id,
	usd.date,
	usd.count_sessions,
	usm.avg_user_daily_sessions AS user_avg_sessions,
	usm.user_stddev_sessions,
	ROUND((usd.count_sessions - usm.avg_user_daily_sessions) / usm.user_stddev_sessions, 2) AS z_score
FROM users_sessions_metrics usm
JOIN user_sessions_daily usd ON usd.user_id = usm.user_id
WHERE usm.user_stddev_sessions != 0
ORDER BY z_score DESC

---

## Task 3: Monthly Revenue vs Previous Year Same Month

**Scenario:**
The finance team wants a year-over-year comparison of monthly order revenue. For each month, show the revenue and compare it to the same month in the previous year.

**Expected Output Columns:**
- `month` (date) — truncated to month
- `monthly_revenue` (numeric) — total order amount that month, rounded to 2 decimals
- `same_month_prev_year_revenue` (numeric) — revenue for the same month 1 year ago, rounded to 2 decimals, NULL if no data
- `yoy_change` (numeric) — monthly_revenue minus same_month_prev_year_revenue, rounded to 2 decimals, NULL if no previous year data
- `yoy_pct_change` (numeric) — percentage change vs previous year, rounded to 1 decimal, NULL if no previous year data

**Requirements:**
- Use `orders` table
- Use `LAG` with `OFFSET 12` to get the same month from the previous year
- Exclude NULL order amounts
- Order by `month ASC`

**Difficulty Rating:** 4/5

WITH orders_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
	--DATE_TRUNC('Month', (DATE_TRUNC('Month', created_at) - INTERVAL '365' DAY)) AS prev_year
FROM orders
),
monthly_revenueS AS (
SELECT 
	month_,
	--prev_year,
	SUM(amount) AS total_revenue
FROM orders_months
GROUP BY month_
ORDER BY month_
),
revenues_prev_year AS (
SELECT 
	*,
	LAG(total_revenue, 12) OVER (ORDER BY month_) AS same_month_prev_year_revenue
FROM monthly_revenues
)
SELECT 
	*,
	total_revenue - same_month_prev_year_revenue AS yoy_change,
	ROUND(total_revenue::NUMERIC / same_month_prev_year_revenue::NUMERIC * 100, 1) || '%' AS yoy_pct_change
FROM revenues_prev_year
WHERE same_month_prev_year_revenue IS NOT NULL

Learning that we can specify the offset number in LAG is very useful - I didn't know that to be honest.


---

## Submission Instructions

1. Task 1 — Category/product hierarchy by quantity sold (4/5)
2. Task 2 — User session outlier days with z-score (4/5)
3. Task 3 — Monthly revenue vs previous year (4/5)
n### Task Archive: 2026-03-02 (Week 12, Day 1)n
# Daily SQL Practice Tasks

**Generated:** 2026-02-27
**Week 11, Day 5 Focus:** Friday Challenge — Full HackerRank Hard Simulation

---

## Task 1: 3-Level Hierarchy — Chat Tickets by Status and Priority

**Scenario:**
Build a 3-level hierarchy over support ticket data:
- Level 1: `'All Tickets'`
- Level 2: Distinct ticket statuses (dynamic, from `chat_tickets`)
- Level 3: For each status, the 3 priorities with the most tickets (show priority name and count as text, e.g. `'high (42)'`)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — status at Level 2, `'priority (count)'` string at Level 3
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Pre-aggregate ticket counts per status+priority before the recursive CTE
- Level 3 name should be formatted as `priority || ' (' || count::text || ')'`
- Termination condition required

**Difficulty Rating:** 4/5

WITH RECURSIVE ticket_counts AS (
SELECT 
	priority,
	status,
	COUNT(*) AS ticket_cnt
FROM chat_tickets
GROUP BY priority, status
),
counts_rank AS
(
SELECT 
	*,
	ROW_NUMBER() OVER (PARTITION BY status ORDER BY ticket_cnt DESC) AS rank_
FROM ticket_counts
),
distinct_statuses AS (
SELECT DISTINCT status FROM  chat_tickets
),
top_three_counts AS (
SELECT * FROM counts_rank
WHERE rank_ <= 3
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Tickets' AS name,
	NULL::TEXT AS parent_name,
	'All Tickets' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(ds.status, ttc.priority || ' (' || ttc.ticket_cnt || ')'),
	h.name,
	h.PATH || ' > ' || COALESCE(ds.status, ttc.priority || ' (' || ttc.ticket_cnt || ')')
FROM HIERARCHY h
LEFT JOIN distinct_statuses ds ON h.LEVEL = 1
LEFT JOIN top_three_counts ttc ON h.LEVEL = 2 AND ttc.status = h.name
WHERE h.LEVEL < 3
)
SELECT * FROM hierarchy

It wasn't easy, but it's a nice practice exercise for formatting


---

## Task 2: Gaps-and-Islands — User Order Streaks by Month

**Scenario:**
The retention team wants to find users with long consecutive monthly ordering streaks. A streak is a sequence of calendar months where the user placed at least one order each month, with no gaps.

Find users with a streak of at least 3 consecutive months. For each qualifying user show their longest streak only.

**Expected Output Columns:**
- `user_id` (integer)
- `streak_start` (date) — first month of the streak (truncated to month)
- `streak_end` (date) — last month of the streak
- `streak_length` (bigint) — number of consecutive months
- `streak_revenue` (numeric) — total order revenue during the streak, rounded to 2 decimals

**Requirements:**
- Use `orders` table
- Truncate orders to month, deduplicate (one row per user per month)
- Apply gaps-and-islands: `ROW_NUMBER()` subtracted from the month produces the streak group key
- For streak_revenue: join back to raw orders to sum amounts within the streak period
- If a user has multiple streaks of equal length, show the most recent one
- Order by `streak_length DESC`, `streak_revenue DESC`

**Difficulty Rating:** 5/5

That's one of the hardest tasks we've had recently, but I've handled it well and check whether it works properly - it does and correctly sums up the revenues across different months of the streak. However, THERE WERE NO users with streak above 2 months, so obviously I didn't filter for that. FYI: Streak revenue is rounded to 2 decimals by default.

WITH users_order_months AS (
SELECT 
	DISTINCT user_id,
	DATE_TRUNC('Month', created_at) AS order_month
FROM orders
),
users_months_rn AS (
SELECT 
	*,
	ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_month)
FROM users_order_months
),
users_streak_keys AS (
SELECT 
	*,
	order_month - row_number * INTERVAL '1' MONTH AS streak_key
FROM users_months_rn
),
users_monthly_streaks_no_revenue AS (
SELECT
 	user_id,
 	streak_key,
 	MIN(order_month) AS streak_start,
 	MAX(order_month) AS streak_end,
 	max(row_number) AS streak_length
 FROM users_streak_keys
 GROUP BY user_id, streak_key
ORDER BY user_id
),
users_orders AS (
SELECT 
	user_id,
	amount,
	DATE_TRUNC('Month', created_at) AS order_month
FROM orders
),
users_monthly_revenues AS (
SELECT
	user_id,
	order_month,
	SUM(amount) AS total_revenue
FROM users_orders
GROUP BY user_id, order_month
)
SELECT 
	umsnr.user_id,
	umsnr.streak_start,
	umsnr.streak_end,
	umsnr.streak_length,
	SUM(umr.total_revenue) AS streak_revenue
FROM users_monthly_streaks_no_revenue umsnr
JOIN users_monthly_revenues umr ON umsnr.user_id = umr.user_id
WHERE umr.order_month >= umsnr.streak_start AND umr.order_month <= umsnr.streak_end
GROUP BY umsnr.user_id, umsnr.streak_start, umsnr.streak_end, umsnr.streak_length
ORDER BY streak_length DESC, streak_revenue DESC


---

## Task 3: Support Ticket Complexity Score

**Scenario:**
The support team wants to score each ticket by complexity — based on how many messages it received, how many unique participants were involved, and whether it was escalated (ever had `priority = 'urgent'`).

Complexity score formula:
`(message_count * 1.0) + (unique_participants * 2.0) + (is_urgent * 5.0)`

**Expected Output Columns:**
- `ticket_id` (bigint)
- `message_count` (bigint)
- `unique_participants` (bigint) — distinct non-NULL user_ids from `chat_messages`
- `is_urgent` (integer) — 1 if ticket priority is `'urgent'`, 0 otherwise
- `complexity_score` (numeric) — rounded to 1 decimal
- `complexity_rank` (bigint) — ranked by complexity_score DESC

**Requirements:**
- Use `chat_tickets` and `chat_messages`
- Only include tickets with at least 2 messages
- Order by `complexity_rank ASC`

**Difficulty Rating:** 4/5


WITH tickets_msgs_participants AS (
SELECT 
	ticket_id,
	COUNT(*) AS msg_cnt,
	COUNT(DISTINCT(author_id)) AS unique_authors,
	COUNT(DISTINCT(user_id)) AS unique_users
FROM chat_messages
WHERE message_type = 'text'
GROUP BY ticket_id
),
tickets_shallow_rank AS (
SELECT 
	ct.id AS ticket_id,
	tmp.msg_cnt,
	tmp.unique_authors + tmp.unique_users AS unique_participants,
	CASE 
		WHEN ct.priority = 'urgent' THEN 1 ELSE 0
	END AS is_urgent
FROM chat_tickets ct
JOIN tickets_msgs_participants tmp ON ct.id = tmp.ticket_id
),
tickets_complexity_scores AS (
SELECT 
	*,
	(msg_cnt * 1.0) + (unique_participants * 2.0) + (is_urgent * 5.0) AS complexity_score
FROM tickets_shallow_rank
WHERE msg_cnt >= 2
)
SELECT 
	*,
	DENSE_RANK() OVER (ORDER BY complexity_score DESC) AS complexity_rank
FROM tickets_complexity_scores
ORDER BY complexity_rank

FYI: Null users were not counted here, the score is already rounded, as all the results are int numbers (there's no other possibility), and everything is as you wanted.

---

## Submission Instructions

1. Task 1 — Chat ticket status/priority hierarchy (4/5)
2. Task 2 — Monthly order streaks (5/5)
3. Task 3 — Ticket complexity score (4/5)
n### Task Archive: 2026-03-03 (Week 12, Day 2)n
# Daily SQL Practice Tasks

**Generated:** 2026-03-02
**Week 12, Day 1 Focus:** Light Recovery Session — Core Fundamentals

---

## Task 1: Top Spenders per Country

**Scenario:**
Find the top 3 users by total order amount in each country.

**Expected Output Columns:**
- `country` (text)
- `user_id` (integer)
- `total_spent` (numeric) — rounded to 2 decimals
- `country_rank` (bigint)

**Requirements:**
- Use `users` and `orders` tables
- Exclude NULL countries
- Order by `country ASC`, `country_rank ASC`

**Difficulty Rating:** 2/5

WITH users_countries_spendings AS (
SELECT 
	u.country,
	o.user_id,
	SUM(o.amount) AS total_spent
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.country IS NOT NULL
GROUP BY u.country, o.user_id
),
countries_spendings_ranks AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY COUNTRY ORDER BY total_spent DESC) AS country_spending_rank
FROM users_countries_spendings
)
SELECT 
	*
FROM countries_spendings_ranks
WHERE country_spending_rank <= 3
ORDER BY country, country_spending_rank


---

## Task 2: Daily Order Count with 7-Day Rolling Average

**Scenario:**
For each day that has at least one order, show the number of orders placed and a 7-day rolling average of order count.

**Expected Output Columns:**
- `date` (date)
- `daily_order_count` (bigint)
- `rolling_7d_avg` (numeric) — rounded to 1 decimal

**Requirements:**
- Use `orders` table only
- Truncate `created_at` to date
- Rolling window: `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`
- Order by `date ASC`

**Difficulty Rating:** 2/5

WITH orders_dates AS (
SELECT 
	*,
	DATE_TRUNC('Day', created_at) AS date
FROM orders
),
dates_order_cnt AS (
SELECT 
	date,
	COUNT(id) AS daily_order_count
FROM orders_dates
GROUP BY date
ORDER BY date
)
SELECT 
	*,
	ROUND(AVG(daily_order_count) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW), 1) AS rolling_7d_avg
FROM dates_order_cnt

---

## Task 3: Most Active Support Ticket per User

**Scenario:**
For each user who has opened at least one ticket, find their most active ticket — the one with the most messages.

**Expected Output Columns:**
- `user_id` (bigint)
- `ticket_id` (bigint)
- `ticket_status` (text)
- `message_count` (bigint)

**Requirements:**
- Use `chat_tickets` and `chat_messages`
- If a user has multiple tickets with the same max message count, show the most recently created one
- Order by `message_count DESC`

**Difficulty Rating:** 2/5

WITH tickets_msg_count AS (
SELECT 
	ticket_id,
	COUNT(id) AS msg_count
FROM chat_messages
WHERE message_type = 'text'
GROUP BY ticket_id
),
users_ticket_ranks AS (
SELECT
	*,
	RANK() OVER (PARTITION BY user_id ORDER BY msg_count DESC) AS user_ticket_rank
FROM tickets_msg_count tmc
JOIN chat_tickets ct ON TMC.ticket_id = ct.id
)
SELECT 
	user_id,
	ticket_id,
	status,
	msg_count
FROM users_ticket_ranks
WHERE user_ticket_rank = 1
ORDER BY msg_count DESC

Done and functioning as expected. There were no duplicate msg counts, so it worked perfectly.


---

## Submission Instructions

1. Task 1 — Top spenders per country (2/5)
2. Task 2 — Daily orders with rolling average (2/5)
3. Task 3 — Most active ticket per user (2/5)
n### Task Archive: 2026-03-04 (Week 12, Day 3)n
# Daily SQL Practice Tasks

**Generated:** 2026-03-03
**Week 12, Day 2 Focus:** HackerRank Hard — Full Difficulty

---

## Task 1: 3-Level Hierarchy — Orders by Delivery Status and Top Users

**Scenario:**
Build a 3-level hierarchy over order/delivery data:
- Level 1: `'All Orders'`
- Level 2: Distinct delivery statuses (dynamic, from `deliveries`)
- Level 3: For each status, the 3 users with the highest total order amount under that delivery status (show user_id as text)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — status at Level 2, user_id at Level 3
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Join `deliveries → orders` to get user_id and amount per delivery status
- Pre-aggregate totals per user+status, then rank before the recursive CTE
- Termination condition required

**Difficulty Rating:** 4/5

WITH RECURSIVE distinct_statuses AS (
SELECT DISTINCT status FROM deliveries
),
users_delivery_statuses AS (
SELECT 
	o.user_id,
	d.status,
	SUM(o.amount) AS total_amount
FROM deliveries d
JOIN orders o ON d.order_id = o.id
GROUP BY o.user_id, d.status
ORDER BY o.user_id
),
users_statuses_rank AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY status ORDER BY total_amount DESC) AS status_rank
FROM users_delivery_statuses
),
top_three_orders_per_status AS (
SELECT 
	* 
FROM users_statuses_rank
WHERE status_rank <= 3
),
HIERARCHY AS (
SELECT
	1 AS level,
	'All Orders' AS name,
	NULL::TEXT AS parent_name,
	'All Orders' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(ds.status, tto.user_id::TEXT),
	h.name,
	h.PATH || ' < ' || COALESCE(ds.status, tto.user_id::TEXT)
FROM HIERARCHY h
LEFT JOIN distinct_statuses ds ON h.LEVEL = 1
LEFT JOIN top_three_orders_per_status tto ON h.LEVEL = 2 AND tto.status = h.name
WHERE h.LEVEL < 3
)
SELECT * FROM hierarchy


---

## Task 2: First and Repeat Purchaser Revenue Split

**Scenario:**
The growth team wants to understand how much revenue comes from first-time buyers vs repeat buyers each month.

For each month, classify each order as either a user's first-ever order (`first_purchase`) or a subsequent one (`repeat_purchase`), then aggregate revenue by month and purchase type.

**Expected Output Columns:**
- `month` (date) — truncated to month
- `purchase_type` (text) — `'first_purchase'` or `'repeat_purchase'`
- `order_count` (bigint)
- `total_revenue` (numeric) — rounded to 2 decimals
- `pct_of_monthly_revenue` (numeric) — this type's revenue as % of total revenue that month, rounded to 1 decimal

**Requirements:**
- Use `orders` table only
- Identify each user's first order using `MIN(created_at)` or `ROW_NUMBER()`
- `pct_of_monthly_revenue` requires a window SUM over the month partition
- Order by `month ASC`, `purchase_type ASC`

**Difficulty Rating:** 5/5

WITH orders_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
FROM orders
),
users_order_types AS (
SELECT 
	*,
	FIRST_VALUE(created_at) OVER (PARTITION BY month_, user_id ORDER BY created_at) AS first_order_time,
	CASE WHEN FIRST_VALUE(created_at) OVER (PARTITION BY month_, user_id ORDER BY created_at) = created_at THEN 'first_purchase' ELSE 'repeat_purchase' END AS purchase_type
FROM orders_months
),
users_order_types_total_revenue AS (
SELECT 
	*,
	SUM(amount) OVER (PARTITION BY month_, user_id, purchase_type ORDER BY created_at) AS total_revenue
FROM users_order_types
),
monthly_users_repeat_purchases_revenues AS (
SELECT 
	user_id,
	month_,
	COUNT(*) AS order_count,
	MAX(total_revenue) AS repeat_purchase_revenue
FROM users_order_types_total_revenue
GROUP BY user_id, month_
)
SELECT 
	uot.month_,
	uot.user_id,
	uot.total_revenue AS first_purchase_revenue,
	mur.repeat_purchase_revenue AS total_revenue,
	mur.order_count,
	CASE WHEN uot.total_revenue = mur.repeat_purchase_revenue THEN 100 ELSE ROUND((uot.total_revenue / (mur.repeat_purchase_revenue - uot.total_revenue))::NUMERIC * 100, 1) END AS first_purchases_pct_of_monthly_revenue
FROM monthly_users_repeat_purchases_revenues mur
JOIN users_order_types_total_revenue uot ON mur.month_ = uot.month_ AND mur.user_id = uot.user_id AND uot.purchase_type = 'first_purchase'
ORDER BY uot.user_id


So it works perfectly fine and the logic is sound, but there are some differences, and data is shown in a bit different way, but it's still clear.

First of all, I don't have this division on first_purchase/repeat_purchase, but rather I calculated it on my own and divided the first_purchase_revenue from total_revenue. I run proper checks to subtract first_purchase_revenue for percent calculations in case it's different (because some months and users only had one purchase for a given month etc.).

This logic works, and I've named the rows properly to make it all sound and clear.
The differences have risen because I DIDN'T FOLLOW your instructions step by step, but rather ran my own thinking and reasoning process based on the base task instruction. I don't think it's bad as I wanted to stimulate thinking on my own. As a DA I will most likely not have an AI supervisor that will tell me how to do everything step by step, so that's my reasoning.

---

## Task 3: Session Engagement Deciles

**Scenario:**
The analytics team wants to segment users into 10 equal engagement buckets (deciles) based on their total session count across all time.

For each decile, show the number of users, the min/max/avg total sessions in that decile, and what percentage of all sessions that decile accounts for.

**Expected Output Columns:**
- `decile` (integer) — 1 (lowest) to 10 (highest)
- `user_count` (bigint)
- `min_sessions` (bigint)
- `max_sessions` (bigint)
- `avg_sessions` (numeric) — rounded to 1 decimal
- `pct_of_total_sessions` (numeric) — rounded to 1 decimal

**Requirements:**
- Use `user_sessions_daily`
- Aggregate total sessions per user first, then apply `NTILE(10)`
- `pct_of_total_sessions` = decile's total sessions / grand total sessions * 100
- Order by `decile ASC`

**Difficulty Rating:** 4/5

WITH user_session_cnt AS (
SELECT 
	user_id,
	SUM(count_sessions) AS total_sessions
FROM user_sessions_daily
GROUP BY user_id
),
user_session_deciles AS (
SELECT 
	*,
	NTILE(10) OVER (ORDER BY total_sessions) AS decile
FROM user_session_cnt
),
deciles_metrics AS (
SELECT 
	decile,
	COUNT(*) AS user_count,
	sum(total_sessions) AS total_sessions,
	MIN(total_sessions) AS min_sessions,
	MAX(total_sessions) AS max_sessions,
	AVG(total_sessions) AS avg_sessions,
	(SELECT SUM(count_sessions) FROM user_sessions_daily) AS sessions_grand_total
FROM user_session_deciles
GROUP BY decile
)
SELECT 
	*,
	decile,
	user_count,
	min_sessions,
	max_sessions,
	avg_sessions,
	ROUND(total_sessions / sessions_grand_total * 100, 1) AS pct_of_total_sessions
FROM deciles_metrics
ORDER BY decile

This task wasn't a problem for me, but I also enjoyed it as it required multi steps approach and some potential traps that I managed to avoid.

---

## Submission Instructions

1. Task 1 — Delivery status hierarchy with top users (4/5)
2. Task 2 — First vs repeat purchaser revenue split (5/5)
3. Task 3 — Session engagement deciles (4/5)
n### Task Archive: 2026-03-05 (Week 12, Day 4)n
# Daily SQL Practice Tasks

**Generated:** 2026-03-04
**Week 12, Day 3 Focus:** HackerRank Hard — Exam Simulation Style

---

## Task 1: 3-Level Hierarchy — Transaction Types, Top Users, and Their Cities

**Scenario:**
Build a 3-level hierarchy over transaction data:
- Level 1: `'All Transactions'`
- Level 2: Distinct transaction types (dynamic, from `transactions`)
- Level 3: For each type, the 3 users with the highest total transaction amount — show their city (or `'Unknown'` if NULL)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — type at Level 2, city name (or `'Unknown'`) at Level 3
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Join `transactions → users` to get city per user
- Pre-aggregate total amount per user+type, rank, keep top 3
- Use `COALESCE(city, 'Unknown')` for NULL cities
- Termination condition required

WITH RECURSIVE transaction_types_amounts AS (
SELECT 
	user_id,
	TYPE,
	SUM(amount) AS transaction_amount
FROM transactions
GROUP BY user_id, TYPE
),
user_type_ranks AS (
SELECT 
	tta.user_id,
	u.city,
	tta."type",
	tta.transaction_amount,
	RANK() OVER (PARTITION BY TYPE ORDER BY transaction_amount DESC) AS type_rank
FROM transaction_types_amounts tta
JOIN users u ON u.id = tta.user_id
),
top_three_user_per_transaction_type AS (
SELECT * FROM user_type_ranks
WHERE type_rank <= 3
),
distinct_types AS (
SELECT 
DISTINCT TYPE
FROM transactions
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Transactions' AS name,
	NULL::TEXT AS parent_name,
	'All Transactions' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(ds.TYPE, ttu.city),
	h.name,
	h.PATH || ' < ' || COALESCE(ds.TYPE, ttu.city)
FROM HIERARCHY h
LEFT JOIN distinct_types ds ON h.LEVEL = 1
LEFT JOIN top_three_user_per_transaction_type ttu ON h.LEVEL = 2 AND ttu.TYPE = h.name
WHERE h.LEVEL < 3
)
SELECT * FROM hierarchy



**Difficulty Rating:** 4/5

---

## Task 2: Global First Purchase vs Repeat — Monthly Revenue Split

**Scenario:**
The growth team wants to understand monthly revenue from brand-new buyers (placing their very first order ever) vs returning buyers (any order after their first).

For each month, show:

**Expected Output Columns:**
- `month` (date) — truncated to month
- `purchase_type` (text) — `'first_purchase'` or `'repeat_purchase'`
- `order_count` (bigint)
- `total_revenue` (numeric) — rounded to 2 decimals
- `pct_of_monthly_revenue` (numeric) — this type's % of total revenue that month, rounded to 1 decimal

**Requirements:**
- Use `orders` table only
- A user's **global first order** = the single earliest order across their entire history (not per month)
- All other orders by that user = `repeat_purchase`
- Aggregate to one row per month+type combination
- `pct_of_monthly_revenue`: window SUM partitioned by month, then divide
- Order by `month ASC`, `purchase_type ASC`

**Difficulty Rating:** 5/5


WITH orders_first_transactions AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_,
	FIRST_VALUE(created_at) OVER (PARTITION BY user_id ORDER BY created_at) AS first_transaction_time
FROM  orders
),
orders_purchase_types AS (
SELECT 
	*,
	CASE WHEN created_at = first_transaction_time THEN 'first_purchase' ELSE 'repeat_purchase' END AS purchase_type
FROM orders_first_transactions
),
monthly_purchase_type_revenues AS (
SELECT 
	month_,
	purchase_type,
	COUNT(id) AS order_count,
	SUM(amount) AS total_revenue
FROM orders_purchase_types
GROUP BY month_, purchase_type
ORDER BY month_
),
total_monthly_revenues AS (
SELECT 
	month_,
	SUM(amount) AS total_monthly_revenue
FROM orders_first_transactions
GROUP BY month_
)
SELECT 
	mpt.month_,
	mpt.purchase_type,
	mpt.order_count,
	mpt.total_revenue,
	round(mpt.total_revenue::numeric / tmr.total_monthly_revenue::numeric * 100, 1) || '%' AS pct_of_monthly_revenue
FROM monthly_purchase_type_revenues mpt
JOIN total_monthly_revenues tmr ON mpt.month_ = tmr.month_
ORDER BY mpt.month_, mpt.purchase_type

Here, it works perfectly.

---

## Task 3: Order Response Time — Time from Order to First Delivery Update

**Scenario:**
The operations team wants to measure fulfillment speed — specifically, how many hours pass between an order being placed and its delivery record being created.

**Expected Output Columns:**
- `order_id` (integer)
- `order_created_at` (timestamp)
- `delivery_created_at` (timestamp)
- `hours_to_fulfillment` (numeric) — rounded to 2 decimals using EPOCH
- `fulfillment_segment` (text):
  - `'same_day'`: fulfilled within 24 hours
  - `'next_day'`: 24 to 48 hours
  - `'delayed'`: more than 48 hours

**Requirements:**
- Use `orders` and `deliveries` tables
- Use `EXTRACT(EPOCH FROM ...) / 3600` for hours
- Only include orders that have a delivery record
- Order by `hours_to_fulfillment ASC`

**Difficulty Rating:** 3/5

WITH orders_deliveries AS (
SELECT 
	d.order_id,
	o.created_at AS order_created_at,
	d.created_at AS delivery_created_at
FROM orders o
JOIN deliveries d ON o.id = d.order_id
WHERE d.status = 'delivered'
),
deliveries_fulfillment_hours AS (
SELECT 
	*,
	EXTRACT('Epoch' FROM delivery_created_at - order_created_at)/3600 AS hours_to_fulfillment
FROM orders_deliveries
)
SELECT 
	*,
	CASE 
		WHEN hours_to_fulfillment < 24 THEN 'same_day'
		WHEN hours_to_fulfillment <= 48 THEN 'next_day'
		WHEN hours_to_fulfillment > 48 THEN 'delayed'
	END AS fulfillment_segment
FROM deliveries_fulfillment_hours
ORDER BY hours_to_fulfillment


---

## Submission Instructions

1. Task 1 — Transaction type/city hierarchy (4/5)
2. Task 2 — Global first purchase vs repeat monthly split (5/5)
3. Task 3 — Order to delivery fulfillment time (3/5)
n### Task Archive: 2026-03-06 (Week 12, Day 5)n
# Daily SQL Practice Tasks

**Generated:** 2026-03-05
**Week 12, Day 4 Focus:** HackerRank Hard — Final Exam Prep

---

## Task 1: 3-Level Hierarchy — User Segments by Country and Registration Year

**Scenario:**
Build a 3-level hierarchy over user registration data:
- Level 1: `'All Users'`
- Level 2: Distinct countries (exclude NULLs, dynamic from `users`)
- Level 3: For each country, the 3 registration years with the most users (show as `'YYYY (N users)'`)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — country at Level 2, formatted string at Level 3
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Pre-aggregate user counts per country+year before the recursive CTE
- Format Level 3 name as: `EXTRACT(YEAR FROM created_at)::text || ' (' || count::text || ' users)'`
- Exclude NULL countries
- Termination condition required

**Difficulty Rating:** 4/5

#Swapped to top three months instead of years, as there were only 2 years of data, so it wouldn't make sense!

WITH RECURSIVE users_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
FROM users
),
countries_months_registrations AS (
SELECT 
	month_,
	country,
	COUNT(id) AS registered_users_count
FROM users_months
WHERE country IS NOT NULL
GROUP BY month_, country
),
countries_monthly_reg_ranks AS (
SELECT 
	*,
	ROW_NUMBER() OVER (PARTITION BY country ORDER BY registered_users_count DESC, month_) AS country_registration_rank
FROM countries_months_registrations
),
countries_top_three_reg_rank_months AS (
SELECT 
* 
FROM countries_monthly_reg_ranks
WHERE country_registration_rank <= 3
),
distinct_countries AS (
SELECT DISTINCT country FROM users
WHERE country IS NOT NULL
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Users' AS name,
	NULL::TEXT AS parent_name,
	'All Users' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(dc.country::text, ctt.month_::TEXT || ' (' || ctt.registered_users_count::TEXT || ' user(s))'),
	h.name,
	h.PATH || ' > ' || COALESCE(dc.country::text, ctt.month_::TEXT || ' (' || ctt.registered_users_count::TEXT || ' user(s))')
FROM HIERARCHY h
LEFT JOIN distinct_countries dc ON h.LEVEL = 1
LEFT JOIN countries_top_three_reg_rank_months ctt ON h.LEVEL = 2 AND ctt.country = h.name
WHERE h.LEVEL < 3
)
SELECT * FROM HIERARCHY


Definitely a very long query, but I managed to handle it, as well as the formatting, which was quite tricky here. Feels good!




---

## Task 2: Running Total of Orders with Threshold Flags

**Scenario:**
The finance team wants to track each user's cumulative order spending over time, and flag the exact order where they crossed key spending milestones.

For each order, show the user's cumulative total spend up to and including that order, and flag whether it's the order that first pushed them over 500, 1000, or 2000 in total spend.

**Expected Output Columns:**
- `order_id` (integer)
- `user_id` (integer)
- `order_amount` (numeric) — rounded to 2 decimals
- `cumulative_spend` (numeric) — running total per user ordered by created_at, rounded to 2 decimals
- `milestone` (text) — `'500'`, `'1000'`, `'2000'` for the first order crossing each threshold, NULL otherwise

**Requirements:**
- Use `orders` table, exclude NULL amounts
- Use a window SUM for cumulative spend
- For milestone: an order crosses a threshold if cumulative_spend >= threshold AND the previous cumulative_spend < threshold
- Only one milestone per order (use the highest threshold crossed if multiple)
- Order by `user_id ASC`, `created_at ASC`

**Difficulty Rating:** 5/5

WITH users_cumulative_spend AS (
SELECT 
	id AS order_id,
	created_at,
	user_id,
	amount,
	SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS user_cumulative_spend
FROM orders
),
users_prev_spend AS (
SELECT 
	*,
	LAG(user_cumulative_spend) OVER (PARTITION BY user_id ORDER BY user_cumulative_spend) AS previous_cumulative_spend
FROM users_cumulative_spend
)
SELECT 
	order_id,
	user_id,
	amount AS order_amount,
	user_cumulative_spend AS cumulative_spend,
	CASE 
		WHEN user_cumulative_spend > 2000 AND previous_cumulative_spend < 2000 THEN '2000'
		WHEN user_cumulative_spend > 1000 AND previous_cumulative_spend < 1000 THEN '1000'
		WHEN user_cumulative_spend > 500 AND previous_cumulative_spend < 500 THEN '500' ELSE NULL
	END AS milestone
FROM users_prev_spend


---

## Task 3: Product Category Affinity — Categories Bought Together

**Scenario:**
The recommendations team wants to know which product categories are most frequently purchased together in the same order.

Find all pairs of distinct categories that appear together in at least 3 orders, ranked by co-occurrence frequency.

**Expected Output Columns:**
- `category_a` (text)
- `category_b` (text)
- `co_occurrence_count` (bigint)
- `co_occurrence_rank` (bigint)

**Requirements:**
- Use `orders_products`, `products`, `product_categories`
- A pair is `category_a < category_b` (alphabetically) to avoid duplicates
- Count distinct orders containing both categories
- Only pairs appearing in >= 3 orders
- Order by `co_occurrence_count DESC`, `category_a ASC`

**Difficulty Rating:** 5/5

WITH orders_categories_deduplicated AS (
SELECT 
	op1.order_id AS order_id1,
	pc1."name" AS category_a,
	pc2."name" AS category_b
FROM orders_products op1
JOIN products p1 ON op1.product_id = p1.id
JOIN product_categories pc1 ON p1.category_id = pc1.id
JOIN orders_products op2 ON op1.order_id = op2.order_id
JOIN products p2 ON op2.product_id = p2.id
JOIN product_categories pc2 ON p2.category_id = pc2.id
WHERE pc1.id < pc2.id
),
categories_cooccurences AS (
SELECT
	category_a,
	category_b,
	COUNT(DISTINCT(order_id1)) AS co_occurence_count
FROM orders_categories_deduplicated
GROUP BY category_a, category_b
)
SELECT 
	*,
	RANK() OVER (ORDER BY co_occurence_count DESC) AS co_occurence_rank
FROM categories_cooccurences
WHERE co_occurence_count >= 3

Handled the task with grace.
I had to add DISTINCT to filter out situations where two different products, even with the same name but different id from the same category were bought together.

---

## Submission Instructions

1. Task 1 — Country/year hierarchy with formatted Level 3 (4/5)
2. Task 2 — Running total with milestone flags (5/5)
3. Task 3 — Category co-occurrence pairs (5/5)
n### Task Archive: 2026-03-12 (Week 13, Day 4)n
# Daily SQL Practice Tasks

**Generated:** 2026-03-06
**Week 12, Day 5 — FINAL SESSION: Full HackerRank Hard Exam Simulation**

---

## Task 1: 3-Level Hierarchy — Support Ticket Statuses, Priorities, and Top Users

**Scenario:**
Build a 3-level hierarchy over support data:
- Level 1: `'All Tickets'`
- Level 2: Distinct ticket statuses (dynamic, from `chat_tickets`)
- Level 3: For each status, the 3 priorities with the highest average resolution time in minutes (show as `'priority (avg_minutes avg min)'`, rounded to 1 decimal)

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — status at Level 2, formatted string at Level 3
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Join `chat_tickets` to calculate avg resolution time per status+priority using `EXTRACT(EPOCH FROM updated_at - created_at) / 60`
- Pre-aggregate avg resolution time per status+priority, then rank before the recursive CTE
- Format Level 3 name as: `priority || ' (' || avg_minutes::text || ' avg min)'`
- Termination condition required

**Difficulty Rating:** 5/5

WITH RECURSIVE tickets_res_times AS (
SELECT 
	ct.id AS ticket_id,
	ct.status,
	ct.priority,
	ct.created_at AS ticket_creation_time,
	EXTRACT(EPOCH FROM ct.updated_at - ct.created_at)/60 AS ticket_resolution_minutes
FROM chat_tickets ct
),
distinct_statuses AS (
SELECT DISTINCT status FROM chat_tickets
),
tickets_statuses_priorities_avg_resolution AS (
SELECT 
	status,
	priority,
	ROUND(AVG(ticket_resolution_minutes), 1) AS avg_resolution_minutes
FROM tickets_res_times
GROUP BY status, priority
),
statuses_ranks AS (
SELECT 
	*,
	rank() OVER (PARTITION BY status ORDER BY avg_resolution_minutes) AS status_resolution_rank
FROM tickets_statuses_priorities_avg_resolution
),
top_three_status_resolution_ranks AS (
SELECT * FROM statuses_ranks
WHERE status_resolution_rank <= 3
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Tickets' AS name,
	NULL::TEXT AS parent_name,
	'All Tickets' AS PATH
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(ds.status, tts.priority || ' (' || tts.avg_resolution_minutes::TEXT || ' avg min)'),
	h.name,
	h.PATH || ' > ' || COALESCE(ds.status, tts.priority || ' (' || tts.avg_resolution_minutes::TEXT || ' avg min)')
FROM HIERARCHY h
LEFT JOIN distinct_statuses ds ON h.LEVEL = 1
LEFT JOIN top_three_status_resolution_ranks tts ON h.LEVEL = 2 AND tts.status = h.name
WHERE h.LEVEL < 3
)
SELECT * FROM hierarchy


---

## Task 2: User Lifetime Value Segments with Gaps-and-Islands

**Scenario:**
The growth team wants to identify high-value users who have maintained consistent monthly ordering habits. For each user, calculate their total lifetime order revenue and their longest consecutive monthly ordering streak. Then segment them:

- `champion`: lifetime_revenue >= 1000 AND longest_streak >= 3 months
- `loyal`: lifetime_revenue >= 500 AND longest_streak >= 2 months
- `at_risk`: lifetime_revenue >= 200 AND longest_streak = 1
- `other`: everything else

**Expected Output Columns:**
- `user_id` (integer)
- `lifetime_revenue` (numeric) — rounded to 2 decimals
- `longest_streak` (bigint) — in months
- `segment` (text)
- `segment_rank` (bigint) — ranked by lifetime_revenue DESC within each segment

**Requirements:**
- Use `orders` table
- Lifetime revenue = SUM of all order amounts per user
- Longest streak = gaps-and-islands on monthly order activity (DATE_TRUNC to month, ROW_NUMBER subtraction pattern)
- Order by `segment ASC`, `segment_rank ASC`

**Difficulty Rating:** 5/5

WITH users_revenues AS (
SELECT 
	user_id,
	SUM(amount) AS lifetime_revenue
FROM orders
GROUP BY user_id
),
orders_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
FROM orders
),
users_order_months AS (
SELECT 
	user_id,
	month_
FROM orders_months
GROUP BY user_id, month_
ORDER BY user_id
),
users_months_rn AS (
SELECT 
	*,
	ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY month_) AS rn
FROM users_order_months
),
users_streak_keys AS (
SELECT 
	*,
	month_ - rn * INTERVAL '1' MONTH AS streak_key,
	LAST_VALUE(month_) OVER (PARTITION BY user_id ORDER BY month_) AS prev_month
FROM users_months_rn 
),
users_streaks AS (
SELECT 
	user_id,
	streak_key,
	COUNT(*) AS streak_length
FROM users_streak_keys
GROUP BY user_id, streak_key
),
users_longest_streaks AS (
SELECT 
	user_id,
	MAX(streak_length) AS longest_streak
FROM users_streaks
GROUP BY user_id
),
users_segments AS (
SELECT 
	ur.user_id,
	ur.lifetime_revenue,
	uls.longest_streak,
	CASE 
		WHEN ur.lifetime_revenue >= 1000 AND uls.longest_streak >= 3 THEN 'champion'
		WHEN ur.lifetime_revenue >= 500 AND uls.longest_streak >= 2 THEN 'loyal'
		WHEN ur.lifetime_revenue >= 20 AND uls.longest_streak >= 1 THEN 'at_risk'
	END AS segment
	FROM users_revenues ur
JOIN users_longest_streaks uls ON ur.user_id = uls.user_id
)
SELECT 
	*,
	RANK() OVER (PARTITION BY segment ORDER BY lifetime_revenue DESC, longest_streak DESC) AS segment_rank
FROM users_segments

Definitely one of the longest queries we've ever done in this learning process, but I've managed to do it with full understanding from end to end. I'm also 100% sure that I've done the task perfectly, despite its complexity.

---

## Task 3: Complete Order Funnel Analysis

**Scenario:**
The analytics team wants a full funnel report showing how orders progress through the system. For each month, show:
- Total orders placed
- Orders that have a delivery record (fulfillment rate)
- Of those with deliveries, orders with status `'delivered'` (delivery success rate)
- Average hours from order creation to delivery creation (for orders that have a delivery), using EPOCH

**Expected Output Columns:**
- `month` (date) — truncated to month
- `total_orders` (bigint)
- `orders_with_delivery` (bigint)
- `delivered_orders` (bigint)
- `fulfillment_rate_pct` (numeric) — orders_with_delivery / total_orders * 100, rounded to 1 decimal
- `delivery_success_rate_pct` (numeric) — delivered_orders / orders_with_delivery * 100, rounded to 1 decimal
- `avg_hours_to_delivery` (numeric) — rounded to 2 decimals, NULL if no deliveries that month

**Requirements:**
- Use `orders` and `deliveries` tables
- Use LEFT JOIN to preserve orders without deliveries
- Use EPOCH for hours calculation
- Order by `month ASC`

**Difficulty Rating:** 4/5

WITH orders_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
FROM orders
),
monthly_total_orders AS (
SELECT 
	month_,
	COUNT(*) AS total_orders
FROM orders_months
GROUP BY month_
),
orders_with_delivery_status AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
FROM deliveries
),
orders_statuses_cnts AS (
SELECT 
	month_,
	status,
	COUNT(*) AS order_cnt
FROM orders_with_delivery_status
GROUP BY month_, status
),
orders_with_delivery_cnts AS (
SELECT 
	month_,
	SUM(order_cnt) AS orders_with_delivery
FROM orders_statuses_cnts
WHERE status = 'pending'
GROUP BY month_
),
orders_delivery_times AS (
SELECT 
	om.month_,
	om.created_at AS order_placed_at,
	d.created_at AS delivered_at,
	EXTRACT(EPOCH FROM d.created_at - om.created_at)/3600 AS order_delivery_time_in_hours
FROM orders_months om
JOIN deliveries d ON om.id = d.order_id
WHERE d.status = 'delivered'
),
orders_avg_delivery_times AS (
SELECT 
	month_,
	ROUND(AVG(order_delivery_time_in_hours), 2) AS avg_hours_to_delivery
FROM orders_delivery_times
GROUP BY month_
),
months_delivery_metrics AS (
SELECT 
	mto.month_,
	mto.total_orders,
	owd.orders_with_delivery,
	osc.order_cnt AS delivered_orders,
	ROUND(owd.orders_with_delivery / mto.total_orders * 100, 1) AS fulfillment_rate_pct,
	ROUND(osc.order_cnt / owd.orders_with_delivery * 100, 1) AS delivery_success_rate_pct,
	oad.avg_hours_to_delivery
FROM monthly_total_orders mto
LEFT JOIN orders_with_delivery_cnts owd ON mto.month_ = owd.month_
LEFT JOIN orders_statuses_cnts osc ON mto.month_ = osc.month_ AND osc.status = 'delivered'
LEFT JOIN orders_avg_delivery_times oad ON mto.month_ = oad.month_
)
SELECT * FROM months_delivery_metrics
ORDER BY month_

Another very long and complex query that needed multiple steps and thinking, but I handled it well.
---

## Submission Instructions

1. Task 1 — Ticket hierarchy with avg resolution time at Level 3 (5/5)
2. Task 2 — User lifetime value + monthly streak segmentation (5/5)
3. Task 3 — Complete order funnel analysis (4/5)

---

*This is the final session of the 12-week SQL Advanced program. Give it everything.*
n### Task Archive: 2026-03-16 (Week 14, Day 1)n
# Daily SQL Practice Tasks

**Generated:** 2026-03-12
**Week 13, Day 4 Focus:** Self-Referencing Recursive CTE + PIVOT Introduction + Anti-Join Patterns

---

## Task 1: Self-Referencing Recursive CTE — Employee Org Chart

**Scenario:**
You have an `employee` table with the following structure:

```
id | first_name | last_name | manager_id
1  | Madeline   | Ray       | NULL
2  | Violet     | Green     | 1
3  | Alton      | Vasquez   | 1
4  | Geoffrey   | Delgado   | 1
5  | Allen      | Garcia    | 2
6  | Marian     | Daniels   | 2
7  | Tricia     | Wong      | 3
8  | Bruce      | Grant     | 3
9  | Darin      | Burke     | 4
10 | Bob        | Freeman   | 5
```

Build a recursive CTE that traverses this hierarchy to unlimited depth. For each employee, show their full reporting path from the root.

**Expected Output Columns:**
- `id` (integer)
- `first_name` (text)
- `last_name` (text)
- `manager_id` (integer)
- `path` (text) — e.g. `'Ray'`, `'Ray->Green'`, `'Ray->Green->Garcia->Freeman'`

**Requirements:**
- Anchor: start with the root employee (manager_id IS NULL)
- Recursive part: JOIN employee back to itself on employee.manager_id = cte.id
- Path: built by appending last_name at each level
- No LEVEL + 1 pattern needed — termination happens naturally when no more children exist
- Order by path ASC

**Note:** This uses a real table called `employee` — it is NOT in schema.md as it's a standalone exercise table provided above. Write the query as if this table exists.

**Difficulty Rating:** 3/5

Dude. The problem is that WE DON'T HAVE SUCH A structure in our database. What are you expecting of me? Rejecting this task, don't count it for today, and next time please prepare.

---

## Task 2: PIVOT — Monthly Transaction Counts by Type

**Scenario:**
The finance team wants a pivoted monthly report showing how many transactions occurred per type, with each type as its own column.

Transform this row-based result:
```
month       | type        | count
2024-08-01  | deposit     | 45
2024-08-01  | withdrawal  | 32
2024-08-01  | payment     | 28
...
```

Into this pivoted shape:
```
month       | deposit | withdrawal | payment | transfer | purchase
2024-08-01  | 45      | 32         | 28      | ...      | ...
```

**Expected Output Columns:**
- `month` (date) — truncated to month
- `deposit` (bigint)
- `withdrawal` (bigint)
- `payment` (bigint)
- `transfer` (bigint)
- `purchase` (bigint)

**Requirements:**
- Use `transactions` table
- Use conditional aggregation: `COUNT(*) FILTER (WHERE type = 'deposit')` or `SUM(CASE WHEN type = 'deposit' THEN 1 ELSE 0 END)`
- One CTE to aggregate, final SELECT to present the pivot
- Order by `month ASC`

**Difficulty Rating:** 4/5

You see, I'm NOT AWARE how to actually create such a pivot yet - YOU have to create a scaffolded approach and teach me how to do it, as it's a new concept for me. Make sure you do it properly next time! Do not take away points from me, treat this as valuable feedback.

---

## Task 3: Anti-Join — Users Who Never Placed an Order

**Scenario:**
The marketing team wants to target users who have registered but never placed a single order. Find all such users.

Solve this three ways in the same file — one query per approach:

**Approach A:** Using `NOT IN`
**Approach B:** Using `NOT EXISTS`
**Approach C:** Using `LEFT JOIN ... WHERE IS NULL`

**Expected Output Columns** (same for all three):
- `user_id` (integer)
- `created_at` (timestamp) — user registration date

**Requirements:**
- Use `users` and `orders` tables
- Order by `created_at ASC`
- After writing all three, add a short comment on which approach you'd prefer and why

**Difficulty Rating:** 3/5
1.
SELECT 
	u.id AS user_id,
	u.created_at
FROM crappy_data_db.users U
WHERE u.id NOT IN (SELECT o.user_id FROM crappy_data_db.orders o)

2.
SELECT 
	u.id AS user_id,
	u.created_at
FROM crappy_data_db.users U
WHERE NOT EXISTS (
SELECT *
FROM crappy_data_db.orders o
WHERE o.user_id = u.id
)

This is definitely weird, it feels unnatural as it requires way more code and it's not that logical. Not sure, why would I pick this option though.

3. 

SELECT 
	u.id AS user_id,
	u.created_at
FROM crappy_data_db.users u
LEFT JOIN crappy_data_db.orders o ON u.id = o.user_id
WHERE o.user_id IS NULL
ORDER BY o.user_id

Not bad.


---

## Submission Instructions

1. Task 1 — Self-referencing org chart (3/5)
2. Task 2 — PIVOT monthly transactions by type (4/5)
3. Task 3 — Anti-join three ways (3/5)
n### Task Archive: 2026-03-17 (Week 14, Day 2)n
# Daily SQL Practice Tasks

**Generated:** 2026-03-16
**Week 14, Day 1 Focus:** Self-Referencing CTE (Type B) + PIVOT Scaffolded (Step A) + Anti-Join NULL Edge Case

---

## Task 1: Self-Referencing Recursive CTE — User Referral Chain

**Scenario:**
The `users` table has a `referred_by` column... except it doesn't in our schema. So we'll use a self-contained CTE with inline data to make this runnable.

The following CTE provides the data — treat it as your source table called `referrals`:

```sql
WITH referrals (id, name, referred_by) AS (
    VALUES
    (1, 'Alice',   NULL),
    (2, 'Bob',     1),
    (3, 'Carol',   1),
    (4, 'Dave',    2),
    (5, 'Eve',     2),
    (6, 'Frank',   4),
    (7, 'Grace',   3)
)
```

Build a recursive CTE on top of this that traverses the referral chain to unlimited depth.

**Expected Output Columns:**
- `id` (integer)
- `name` (text)
- `referred_by` (integer)
- `path` (text) — e.g. `'Alice'`, `'Alice->Bob'`, `'Alice->Bob->Dave->Frank'`
- `depth` (integer) — 1 for root, 2 for direct referrals, etc.

**Requirements:**
- Anchor: start with the root (referred_by IS NULL)
- Recursive: JOIN referrals back to the CTE on referrals.referred_by = cte.id
- Build path by appending name at each step
- Track depth starting at 1
- No LEVEL + 1 termination needed — stops naturally
- Order by path ASC

**Difficulty Rating:** 3/5

WITH RECURSIVE referrals (id, name, referred_by) AS (
    VALUES
    (1, 'Alice',   NULL),
    (2, 'Bob',     1),
    (3, 'Carol',   1),
    (4, 'Dave',    2),
    (5, 'Eve',     2),
    (6, 'Frank',   4),
    (7, 'Grace',   3)
),
HIERARCHY AS (
SELECT
	1 AS id,
	'Alice' AS name,
	NULL::TEXT AS referred_by,
	'Alice' AS PATH,
	1 AS DEPTH
UNION ALL
SELECT
	r.id,
	r.name,
	h.name,
	h.PATH || ' < ' || r.name,
	h.DEPTH + 1
FROM HIERARCHY h
JOIN referrals r ON h.id = r.referred_by
)
SELECT * FROM hierarchy


Nice, but I'd like to implement similar logic of data in our database next time and actually use it instead. We can create a table and add relevant fields in users or wherever. I just need your guidelines and cooperation.




---

## Task 2: PIVOT — Scaffolded Introduction (Step A + Step B)

PIVOT is a new concept. This task walks you through it in two steps.

### Step A — Understand the unpivoted shape

Write a query that produces the raw unpivoted data we want to pivot:

```
month | type | transaction_count
```

- Use `transactions` table
- Group by `DATE_TRUNC('month', created_at)` and `type`
- Order by `month ASC`, `type ASC`

Run this and look at the output. Notice how each type is a separate row per month. **This is what we want to rotate into columns.**

---

### Step B — Write one pivot column manually

Now extend Step A into a pivot. Write a query that produces:

```
month | deposit_count
```

Just one column for now — `deposit_count` = number of transactions where `type = 'deposit'` per month.

Use this pattern:
```sql
COUNT(*) FILTER (WHERE type = 'deposit') AS deposit_count
```

or equivalently:
```sql
SUM(CASE WHEN type = 'deposit' THEN 1 ELSE 0 END) AS deposit_count
```

**Expected Output Columns:**
- `month` (date)
- `deposit_count` (bigint)

Order by `month ASC`.

**Note:** Step C (all 5 columns at once) comes tomorrow once this pattern is clear.

**Difficulty Rating:** 2/5

WITH transactions_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', t.created_at) AS month_
FROM crappy_data_db.transactions t
)
SELECT
	tm.month_,
	COUNT(*) FILTER (WHERE tm.type = 'deposit') AS deposit_count
FROM transactions_months tm
GROUP BY tm.month_
ORDER BY month_

Yeah, now it makes a lot of sense and I get the basic idea.

I was easily able to do all 5 columns now, once I get the basic pattern. This is great honestly and feels like a very useful concept to use that saves me the need to use GROUP BY type. Wondering, how memory efficient is this, as it looks awesome. Definitely want to practice and learn this pattern in more advanced scenarios etc.

WITH transactions_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', t.created_at) AS month_
FROM crappy_data_db.transactions t
)
SELECT
	tm.month_,
	COUNT(*) FILTER (WHERE tm.type = 'deposit') AS deposit_count,
	COUNT(*) FILTER (WHERE tm.type = 'transfer') AS transfer_count,
	COUNT(*) FILTER (WHERE tm.type = 'withdrawal') AS withdrawal_count,
	COUNT(*) FILTER (WHERE tm.type = 'purchase') AS purchase_count,
	COUNT(*) FILTER (WHERE tm.type = 'payment') AS payment_count
FROM transactions_months tm
GROUP BY tm.month_
ORDER BY month_


---

## Task 3: Anti-Join — The NULL Trap in NOT IN

**Scenario:**
Yesterday you wrote three anti-join approaches. Today we explore when one of them silently breaks.

The `orders` table has a `user_id` column that is `NOT NULL` — so `NOT IN` works correctly there. But what if `user_id` could be NULL?

**Part A:** Write this query and run it:
```sql
SELECT id FROM users
WHERE id NOT IN (SELECT user_id FROM orders WHERE user_id IS NULL OR user_id IS NOT NULL)
```
What do you expect it to return? What does it actually return? Write your observation as a comment.

In this case we'd get data without any issues, so we'd get all users who did not make any orders, as I'm 100% sure that by design there are NO NULL id's in orders.
Also, I'd expect that we'd only have IS NOT NULL condition, I don't see the point of having IS NULL or IS NOT NULL, it's basically useless.


**Part B:** Now fix Part A using `NOT EXISTS` instead, so it correctly returns users not in orders regardless of NULLs.

SELECT id FROM crappy_data_db.users u
WHERE NOT EXISTS 
(SELECT 
	* 
FROM crappy_data_db.orders o
WHERE o.user_id = u.id)

**Part C:** Fix it again using `LEFT JOIN ... WHERE IS NULL`.

**Expected insight:** `NOT IN` returns **zero rows** when the subquery contains even one NULL — because `x NOT IN (..., NULL, ...)` evaluates to UNKNOWN, not TRUE, for every row. `NOT EXISTS` and `LEFT JOIN` are NULL-safe.

**Difficulty Rating:** 3/5

SELECT 
	u.id
FROM crappy_data_db.users u
LEFT JOIN crappy_data_db.orders o ON u.id = o.user_id
WHERE o.user_id IS NULL

This pattern feels quite unnatural to use at this point though.


---

## Submission Instructions

1. Task 1 — Self-referencing referral chain (3/5)
2. Task 2 — PIVOT Step A + Step B (2/5)
3. Task 3 — Anti-join NULL trap (3/5)
n### Task Archive: 2026-03-18 (Week 14, Day 3)n
# Daily SQL Practice Tasks

**Generated:** 2026-03-17
**Week 14, Day 2 Focus:** PIVOT Step C + Self-Referencing CTE on Real Data + Anti-Join in Complex Context

---

## Task 1: PIVOT Step C — Full Revenue Pivot by Transaction Type

**Scenario:**
You've mastered the single-column PIVOT. Now write the full pivot — but this time using **revenue** (SUM of amount) instead of counts, and include a `total_revenue` column as well.

**Expected Output Columns:**
- `month` (date) — truncated to month
- `deposit_revenue` (numeric) — rounded to 2 decimals
- `withdrawal_revenue` (numeric) — rounded to 2 decimals
- `payment_revenue` (numeric) — rounded to 2 decimals
- `transfer_revenue` (numeric) — rounded to 2 decimals
- `purchase_revenue` (numeric) — rounded to 2 decimals
- `total_revenue` (numeric) — sum of all types, rounded to 2 decimals

**Requirements:**
- Use `transactions` table
- Use `SUM(amount) FILTER (WHERE type = '...')` pattern
- Exclude NULL amounts
- Order by `month ASC`

**Difficulty Rating:** 3/5

I was wondering if you wanted total revenue for a given month or whole revenue for all time, but I assumed monthly revenues are expected

WITH transactions_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', t.created_at) AS month_
FROM crappy_data_db.transactions t
)
SELECT 
	month_,
	SUM(amount) FILTER (WHERE type = 'deposit') AS deposit_revenue,
	SUM(amount) FILTER (WHERE type = 'withdrawal') AS withdrawal_revenue,
	SUM(amount) FILTER (WHERE type = 'payment') AS payment_revenue,
	SUM(amount) FILTER (WHERE type = 'transfer') AS transfer_revenue,
	SUM(amount) FILTER (WHERE TYPE = 'purchase') AS purchase_revenue,
	SUM(amount) AS total_revenue
FROM transactions_months
GROUP BY month_
ORDER BY month_

---

## Task 2: Self-Referencing CTE — Product Category Tree

**Scenario:**
The `product_categories` table has a `parent_id` column — except it doesn't currently in our schema. Use this inline VALUES table as your data source (it's self-contained and runnable):

```sql
WITH categories (id, name, parent_id) AS (
    VALUES
    (1, 'All Products',    NULL),
    (2, 'Electronics',     1),
    (3, 'Clothing',        1),
    (4, 'Phones',          2),
    (5, 'Laptops',         2),
    (6, 'Men',             3),
    (7, 'Women',           3),
    (8, 'iPhone',          4),
    (9, 'Samsung',         4),
    (10, 'T-Shirts',       6)
)
```

Traverse this tree recursively to unlimited depth. Show each category's full path from root.

**Expected Output Columns:**
- `id` (integer)
- `name` (text)
- `parent_id` (integer)
- `path` (text) — e.g. `'All Products -> Electronics -> Phones -> iPhone'`
- `depth` (integer) — 1 for root

**Requirements:**
- Anchor: root node (`parent_id IS NULL`)
- Recursive: JOIN categories back to CTE on `categories.parent_id = cte.id`
- Path separator: ` -> ` (with spaces)
- Natural termination — no LEVEL limit needed
- Order by `path ASC`

**Difficulty Rating:** 3/5


WITH RECURSIVE categories (id, name, parent_id) AS (
    VALUES
    (1, 'All Products',    NULL),
    (2, 'Electronics',     1),
    (3, 'Clothing',        1),
    (4, 'Phones',          2),
    (5, 'Laptops',         2),
    (6, 'Men',             3),
    (7, 'Women',           3),
    (8, 'iPhone',          4),
    (9, 'Samsung',         4),
    (10, 'T-Shirts',       6)
),
hierarchy AS (
SELECT 
	1 AS id,
	'All Products' AS name,
	NULL::TEXT AS parent_id,
	'All Products' AS PATH,
	1 AS DEPTH
UNION ALL
SELECT
	c.id,
	c.name,
	h.name,
	h.PATH || ' < ' || c.name,
	h.DEPTH + 1
FROM hierarchy h
JOIN categories c ON h.id = c.parent_id
)
SELECT * FROM hierarchy


---

## Task 3: Anti-Join — Products Never Ordered

**Scenario:**
The inventory team wants to identify products that have never appeared in any order. These are candidates for removal from the catalogue.

Solve this using **all three approaches** (NOT IN, NOT EXISTS, LEFT JOIN IS NULL), but this time add a twist: the `orders_products` table links orders to products — so the subquery/join is one step removed from `products`.

Then answer: **which approach do you prefer here and why?**

**Expected Output Columns:**
- `product_id` (integer)
- `product_name` (text)
- `price` (numeric)

**Requirements:**
- Use `products` and `orders_products` tables
- Order by `product_id ASC`

**Difficulty Rating:** 3/5

1. SELECT id AS product_id FROM crappy_data_db.products p
WHERE id NOT IN (SELECT product_id FROM crappy_data_db.orders_products op WHERE op.product_id IS NOT NULL)

Very clean, although such products do not exist - thsi is probably the simplest option, and I consider it the most effective

2. SELECT id AS product_id 
FROM crappy_data_db.products p
WHERE NOT EXISTS
(SELECT * 
FROM crappy_data_db.orders_products op
WHERE op.product_id = p.id
)

Same, it also feels quite good.


3. SELECT p.id AS product_id 
FROM crappy_data_db.products p
LEFT JOIN crappy_data_db.orders_products op ON p.id = op.product_id 
WHERE op.product_id IS NULL


Quite sad that none of these actually give results this time, as there are no such products.
I'd pick options 1-2. 2 is good because it is NULL-proof, but I think option 1 is also NULL-proof, as long as we use IS NOT NULL condition in WHERE of our subquery.


---

## Submission Instructions

1. Task 1 — Full revenue PIVOT with total column (3/5)
2. Task 2 — Self-referencing category tree CTE (3/5)
3. Task 3 — Anti-join on products never ordered, three ways (3/5)
n### Task Archive: 2026-03-19 (Week 14, Day 4)n
# Daily SQL Practice Tasks

**Generated:** 2026-03-18
**Week 14, Day 3 Focus:** Self-Referencing CTE (Anchor Fix) + Time-Proximity Gaps-and-Islands Edge Case + PIVOT Reinforcement

---

## Task 1: Self-Referencing CTE — Fix the Anchor

**Scenario:**
Use the same category tree as yesterday:

```sql
WITH RECURSIVE categories (id, name, parent_id) AS (
    VALUES
    (1, 'All Products',  NULL::int),
    (2, 'Electronics',   1),
    (3, 'Clothing',      1),
    (4, 'Phones',        2),
    (5, 'Laptops',       2),
    (6, 'Men',           3),
    (7, 'Women',         3),
    (8, 'iPhone',        4),
    (9, 'Samsung',       4),
    (10, 'T-Shirts',     6)
)
```

This time, write the anchor by **selecting from the `categories` CTE** with `WHERE parent_id IS NULL` — do not hardcode any id or name. The query must work correctly even if the root node changes.

**Expected Output Columns:**
- `id` (integer)
- `name` (text)
- `depth` (integer) — 1 for root
- `path` (text) — separator ` -> `

**Requirements:**
- Anchor: `SELECT ... FROM categories WHERE parent_id IS NULL`
- Recursive: `JOIN categories ON categories.parent_id = cte.id`
- Natural termination
- Order by `path ASC`

**Difficulty Rating:** 2/5


WITH RECURSIVE categories (id, name, parent_id) AS (
    VALUES
    (1, 'All Products',  NULL::int),
    (2, 'Electronics',   1),
    (3, 'Clothing',      1),
    (4, 'Phones',        2),
    (5, 'Laptops',       2),
    (6, 'Men',           3),
    (7, 'Women',         3),
    (8, 'iPhone',        4),
    (9, 'Samsung',       4),
    (10, 'T-Shirts',     6)
),
hierarchy AS (
SELECT 
	*,
	name AS path
FROM categories
WHERE parent_id IS NULL
UNION ALL
SELECT
	c.id,
	c.name,
	c.parent_id,
	h.PATH || '->' || c.name
FROM HIERARCHY h
JOIN categories c ON c.parent_id = h.id
)
SELECT * FROM hierarchy

---

## Task 2: Time-Proximity Gaps-and-Islands — Session Windows

**Scenario:**
You have a stream of user events (transactions) and want to group them into "sessions" — bursts of activity where consecutive events are within 60 minutes of each other. If the gap between two consecutive events for the same user exceeds 60 minutes, it's a new session.

Use this inline data as your source:

```sql
WITH events (user_id, event_time) AS (
    VALUES
    (1, '2024-01-01 08:00'::timestamp),
    (1, '2024-01-01 08:30'::timestamp),
    (1, '2024-01-01 08:58'::timestamp),
    (1, '2024-01-01 09:03'::timestamp),
    (1, '2024-01-01 10:30'::timestamp),
    (1, '2024-01-01 11:45'::timestamp),
    (2, '2024-01-01 09:00'::timestamp),
    (2, '2024-01-01 09:50'::timestamp),
    (2, '2024-01-01 11:00'::timestamp)
)
```

Group events into sessions. The 08:58 and 09:03 events are only 5 minutes apart — they belong to the **same session** even though they straddle the hour boundary.

**Expected Output Columns:**
- `user_id` (integer)
- `session_id` (integer) — sequential session number per user (1, 2, 3...)
- `session_start` (timestamp) — first event in the session
- `session_end` (timestamp) — last event in the session
- `event_count` (bigint) — number of events in the session

**Requirements:**
- Use LAG to get the previous event time per user
- A new session starts when the gap to the previous event exceeds 60 minutes (or it's the user's first event)
- Use `SUM() OVER` to create a session group key (cumulative count of session-starts)
- Then GROUP BY the session key to produce one row per session
- Order by `user_id ASC`, `session_start ASC`

**Difficulty Rating:** 5/5

WITH events (user_id, event_time) AS (
    VALUES
    (1, '2024-01-01 08:00'::timestamp),
    (1, '2024-01-01 08:30'::timestamp),
    (1, '2024-01-01 08:58'::timestamp),
    (1, '2024-01-01 09:03'::timestamp),
    (1, '2024-01-01 10:30'::timestamp),
    (1, '2024-01-01 11:45'::timestamp),
    (2, '2024-01-01 09:00'::timestamp),
    (2, '2024-01-01 09:50'::timestamp),
    (2, '2024-01-01 11:00'::timestamp)
),
users_prev_events AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY user_id ORDER BY event_time) AS session_id,
	LAG(event_time) OVER (PARTITION BY user_id) AS prev_event_time
FROM events
),
users_events_new_sessions AS (
SELECT 
	*,
	CASE
		WHEN prev_event_time IS NULL OR event_time - prev_event_time > INTERVAL '60 minutes' THEN 1
		ELSE 0
	END AS is_new_session
FROM users_prev_events
),
users_events_session_keys AS (
SELECT 
	*,
	SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY event_time) AS session_key
FROM users_events_new_sessions
)
SELECT
	user_id,
	session_key,
	MIN(event_time) AS session_start,
	MAX(event_time) AS session_end,
	COUNT(*) AS event_count
FROM users_events_session_keys
GROUP BY user_id, session_key
ORDER BY user_id, session_start
	

Feels great and WE DEFINITELY NEED TO PRACTICE THIS PATTERN IN ALL DIFFERENT CONTEXTS

---

## Task 3: PIVOT — Order Count by Delivery Status per Month

**Scenario:**
Build a pivot showing how many orders were in each delivery status per month.

**Expected Output Columns:**
- `month` (date) — truncated to month
- `pending` (bigint)
- `delivered` (bigint)
- `total_orders_with_delivery` (bigint) — total across all statuses

**Requirements:**
- Use `orders` and `deliveries` tables
- Join on `order_id`
- Use `COUNT(*) FILTER (WHERE status = '...')` pattern
- Order by `month ASC`

**Difficulty Rating:** 3/5

WITH orders_deliveries_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', d.created_at) AS delivery_month
FROM crappy_data_db.orders o
JOIN crappy_data_db.deliveries d ON o.id = d.order_id
)
SELECT 
	delivery_month,
	COUNT(*) FILTER (WHERE status = 'pending') AS pending,
	COUNT(*) FILTER (WHERE status = 'delivered') AS delivered,
	COUNT(DISTINCT(order_id)) AS total_orders_with_delivery
FROM orders_deliveries_months
GROUP BY delivery_month
ORDER BY delivery_month

I've added DISTINCT here for total_orders_with_delivery, as otherwise we'd simply get duplicated orders as their status changes since there are many more than just these two statuses.


---

## Submission Instructions

1. Task 1 — Self-referencing CTE with non-hardcoded anchor (2/5)
2. Task 2 — Time-proximity session grouping (5/5)
3. Task 3 — Delivery status PIVOT by month (3/5)
n### Task Archive: 2026-03-20 (Week 14, Day 5)n
# Daily SQL Practice Tasks

**Generated:** 2026-03-19
**Week 14, Day 4 Focus:** Time-Proximity Sessions (New Context) + PIVOT Reinforcement + Anti-Join Complex

---

## Task 1: Time-Proximity Gaps-and-Islands — Trade Clustering

**Scenario:**
A trading desk wants to group trades into "bursts" — clusters of consecutive trades where each trade occurs within 30 minutes of the previous one for the same user. If the gap exceeds 30 minutes, it's a new burst.

Use this inline data:

```sql
WITH trades (user_id, trade_time, amount) AS (
    VALUES
    (1, '2024-01-15 09:02'::timestamp, 1500.00),
    (1, '2024-01-15 09:18'::timestamp, 2300.00),
    (1, '2024-01-15 09:45'::timestamp, 800.00),
    (1, '2024-01-15 10:20'::timestamp, 4100.00),
    (1, '2024-01-15 10:48'::timestamp, 950.00),
    (1, '2024-01-15 11:30'::timestamp, 3200.00),
    (2, '2024-01-15 08:55'::timestamp, 600.00),
    (2, '2024-01-15 09:10'::timestamp, 1200.00),
    (2, '2024-01-15 09:38'::timestamp, 900.00),
    (2, '2024-01-15 11:05'::timestamp, 2800.00)
)
```

**Expected Output Columns:**
- `user_id` (integer)
- `burst_id` (bigint) — sequential burst number per user (1, 2, 3...)
- `burst_start` (timestamp)
- `burst_end` (timestamp)
- `trade_count` (bigint)
- `total_amount` (numeric) — sum of amounts in the burst

**Requirements:**
- Gap threshold: 30 minutes
- Use LAG → is_new_burst flag → SUM() OVER session key pattern
- `burst_id` should be sequential per user (1, 2, 3...) — use RANK() or DENSE_RANK() over session_key
- Order by `user_id ASC`, `burst_start ASC`

**Difficulty Rating:** 4/5

WITH trades (user_id, trade_time, amount) AS (
    VALUES
    (1, '2024-01-15 09:02'::timestamp, 1500.00),
    (1, '2024-01-15 09:18'::timestamp, 2300.00),
    (1, '2024-01-15 09:45'::timestamp, 800.00),
    (1, '2024-01-15 10:20'::timestamp, 4100.00),
    (1, '2024-01-15 10:48'::timestamp, 950.00),
    (1, '2024-01-15 11:30'::timestamp, 3200.00),
    (2, '2024-01-15 08:55'::timestamp, 600.00),
    (2, '2024-01-15 09:10'::timestamp, 1200.00),
    (2, '2024-01-15 09:38'::timestamp, 900.00),
    (2, '2024-01-15 11:05'::timestamp, 2800.00)
),
users_trades AS (
SELECT 
	*,
	LAG(trade_time) OVER (PARTITION BY user_id ORDER BY trade_time) AS prev_trade_time
FROM trades
),
users_streak_starts AS (
SELECT 
	*,
	CASE WHEN prev_trade_time IS NULL OR trade_time - prev_trade_time > INTERVAL '30 MINUTES' THEN 1 ELSE 0 END AS is_start 
FROM users_trades
),
users_streak_keys AS (
SELECT 
	*,
	SUM(is_start) OVER (PARTITION BY user_id ORDER BY trade_time) AS burst_id
FROM users_streak_starts
)
SELECT
	user_id,
	burst_id,
	MIN(trade_time) AS burst_start,
	MAX(trade_time) AS burst_end,
	COUNT(*) AS trade_count,
	SUM(amount) AS total_amount
FROM users_streak_keys _keys
GROUP BY user_id, burst_id
ORDER BY user_id, burst_start


Works again, but oh man, that pattern feels so unintuitive now. I really needed to check the yesterday's progress to use it again. We need to practice it more, as it's golden

---

## Task 2: PIVOT — User Age Group Revenue Breakdown

**Scenario:**
The marketing team wants a monthly revenue breakdown by user age group. Classify users into age buckets and pivot them as columns.

Age groups:
- `under_30`: age < 30
- `age_30_to_50`: age BETWEEN 30 AND 50
- `over_50`: age > 50
- `unknown`: age IS NULL

**Expected Output Columns:**
- `month` (date) — truncated to month
- `under_30` (numeric) — total order revenue, rounded to 2 decimals
- `age_30_to_50` (numeric)
- `over_50` (numeric)
- `unknown` (numeric)
- `total_revenue` (numeric)

**Requirements:**
- Use `users` and `orders` tables
- Join on `user_id`
- Use `SUM(amount) FILTER (WHERE ...)` pattern with CASE for age bucketing
- Order by `month ASC`

**Difficulty Rating:** 4/5

WITH users_orders AS (
SELECT 
	*,
	DATE_TRUNC('Month', o.created_at) AS month_
FROM crappy_data_db.users u
JOIN crappy_data_db.orders o ON u.id = o.user_id
)
SELECT 
	month_,
	COALESCE(SUM(amount) FILTER (WHERE age < 30), 0) AS under_30,
	COALESCE(SUM(amount) FILTER (WHERE age >= 30 AND age <= 50), 0) AS age_30_to_50,
	COALESCE(SUM(amount) FILTER (WHERE age > 50), 0) AS over_50,
	COALESCE(SUM(amount) FILTER (WHERE age IS NULL), 0) AS unknown,
	SUM(amount) AS total_revenue
FROM users_orders
GROUP BY month_
ORDER BY month_

This pattern is super handy and much more efficient than classic CASE WHEN FILTERING and then grouping - love it. Definitely gotta practice it and use in a lot of differnet contexts and advanced scenarios.

This talk felt easy.

---

## Task 3: Complex Anti-Join — Users Who Ordered but Never Had a Delivery

**Scenario:**
The operations team wants to find users who placed at least one order but none of their orders ever received a delivery record. These are potential fulfilment failures.

**Expected Output Columns:**
- `user_id` (integer)
- `order_count` (bigint) — total orders placed by this user
- `total_order_value` (numeric) — rounded to 2 decimals

**Requirements:**
- Use `users`, `orders`, and `deliveries` tables
- A user qualifies if they have orders but NONE of those orders appear in `deliveries`
- Solve using `NOT EXISTS` (preferred) — no need for all three approaches this time
- Order by `total_order_value DESC`

**Difficulty Rating:** 4/5\\

SELECT * 
FROM crappy_data_db.orders o
WHERE NOT EXISTS
(SELECT * FROM crappy_data_db.deliveries d
 WHERE d.order_id = o.id)

 That's the first and most logical step.
 Such orders DO NOT EXIST, SO THERE'S ABSOLUTELY NO reason to proceed further.

 IF THERE WERE such orders/users, it would be as simple as finding their total order count and order value.

---

## Submission Instructions

1. Task 1 — Trade burst clustering (4/5)
2. Task 2 — Age group revenue PIVOT (4/5)
3. Task 3 — Users with orders but no deliveries (4/5)
n### Task Archive: 2026-03-23 (Week 15, Day 1)n
# Daily SQL Practice Tasks

**Generated:** 2026-03-20
**Week 14, Day 5 Focus:** Time-Proximity Drill + PIVOT Complex + Self-Referencing CTE Reinforcement

---

## Task 1: Time-Proximity Gaps-and-Islands — Chat Message Bursts

**Scenario:**
The support team wants to group chat messages into "conversation bursts" — clusters of messages within the same ticket where consecutive messages arrive within 10 minutes of each other. A gap > 10 minutes means a new burst starts.

For each burst, show the ticket_id, burst number, start/end time, message count, and how many unique senders were involved.

**Expected Output Columns:**
- `ticket_id` (bigint)
- `burst_id` (bigint) — sequential burst number per ticket (1, 2, 3...)
- `burst_start` (timestamp)
- `burst_end` (timestamp)
- `message_count` (bigint)
- `unique_senders` (bigint) — distinct non-NULL user_ids in the burst

**Requirements:**
- Use `chat_messages` table
- Gap threshold: 10 minutes between consecutive messages within the same ticket
- Use LAG → is_new_burst flag → SUM() OVER burst_key → GROUP BY pattern
- `burst_id` should be sequential per ticket using RANK() or DENSE_RANK()
- Only include bursts with at least 2 messages
- Order by `ticket_id ASC`, `burst_start ASC`

**Difficulty Rating:** 4/5


WITH ticket_msgs AS (
SELECT 
	*,
	LAG(cm.created_at) OVER (PARTITION BY cm.ticket_id ORDER BY cm.created_at) AS prev_msg_time
FROM crappy_data_db.chat_messages cm
WHERE cm.message_type = 'text'
),
ticket_burst_starts AS (
SELECT 
	*,
	CASE WHEN prev_msg_time IS NULL OR created_at - prev_msg_time > INTERVAL '10 Minutes' THEN 1 ELSE 0
	END AS burst_start
FROM ticket_msgs
),
ticket_burst_ids AS (
SELECT 
	*,
	SUM(burst_start) OVER (PARTITION BY ticket_id ORDER BY created_at) AS burst_id
FROM ticket_burst_starts
),
ticket_msg_bursts AS (
SELECT
	ticket_id,
	burst_id,
	MIN(created_at) AS burst_start,
	MAX(created_at) AS burst_end,
	COUNT(*) AS message_count,
	COUNT(DISTINCT(user_id, author_id)) AS unique_senders
FROM ticket_burst_ids
GROUP BY ticket_id, burst_id
)
SELECT * FROM ticket_msg_bursts
WHERE message_count > 2


I've checked and sorting is also on point.

---

## Task 2: Self-Referencing CTE — Find All Subordinates of a Manager

**Scenario:**
Using the same employee inline data as before, write a query that finds **all direct and indirect subordinates** of a given manager — for example, all employees who report (directly or indirectly) to employee id = 2 (Violet Green).

Use this data:

```sql
WITH RECURSIVE employees (id, first_name, last_name, manager_id) AS (
    VALUES
    (1, 'Madeline', 'Ray',     NULL::int),
    (2, 'Violet',   'Green',   1),
    (3, 'Alton',    'Vasquez', 1),
    (4, 'Geoffrey', 'Delgado', 1),
    (5, 'Allen',    'Garcia',  2),
    (6, 'Marian',   'Daniels', 2),
    (7, 'Tricia',   'Wong',    3),
    (8, 'Bruce',    'Grant',   3),
    (9, 'Darin',    'Burke',   4),
    (10,'Bob',      'Freeman', 5)
)
```

**Expected Output Columns:**
- `id` (integer)
- `first_name` (text)
- `last_name` (text)
- `depth` (integer) — levels below the starting manager (1 = direct report)
- `path` (text) — from the starting manager down

**Requirements:**
- Anchor: start with direct reports of manager_id = 2 (not the manager themselves)
- Recursive: keep joining employees to the CTE on manager_id = cte.id
- Natural termination
- Order by `path ASC`

**Difficulty Rating:** 3/5


WITH RECURSIVE employees (id, first_name, last_name, manager_id) AS (
    VALUES
    (1, 'Madeline', 'Ray',     NULL::int),
    (2, 'Violet',   'Green',   1),
    (3, 'Alton',    'Vasquez', 1),
    (4, 'Geoffrey', 'Delgado', 1),
    (5, 'Allen',    'Garcia',  2),
    (6, 'Marian',   'Daniels', 2),
    (7, 'Tricia',   'Wong',    3),
    (8, 'Bruce',    'Grant',   3),
    (9, 'Darin',    'Burke',   4),
    (10,'Bob',      'Freeman', 5)
),
hierarchy AS (
SELECT 
	id,
	first_name,
	last_name,
	manager_id,
	1 AS DEPTH,
	manager_id || '->' || id AS path
FROM employees
WHERE manager_id = 2
UNION ALL
SELECT
	e.id,
	e.first_name,
	e.last_name,
	h.id,
	h.DEPTH + 1,
	h.PATH || '->' || e.id
FROM hierarchy h
JOIN employees e ON h.id = e.manager_id
)
SELECT * FROM hierarchy




---

## Task 3: PIVOT — Support Ticket Priority Distribution by Status

**Scenario:**
The support team wants a cross-tab of ticket priorities across statuses — how many tickets of each priority exist for each status.

**Expected Output Columns:**
- `status` (text)
- `low` (bigint)
- `medium` (bigint)
- `high` (bigint)
- `urgent` (bigint)
- `total` (bigint)

**Requirements:**
- Use `chat_tickets` table
- Use `COUNT(*) FILTER (WHERE priority = '...')` pattern
- Order by `status ASC`

**Difficulty Rating:** 3/5

SELECT 
	status,
	COUNT(*) FILTER (WHERE priority = 'low') AS low,
	COUNT(*) FILTER (WHERE priority = 'medium') AS medium,
	COUNT(*) FILTER (WHERE priority = 'high') AS high,
	COUNT(*) FILTER (WHERE priority = 'urgent') AS urgent,
	COUNT(*) AS total
FROM crappy_data_db.chat_tickets
GROUP BY status
ORDER BY status

Wow, this is so easy and useful - everything in such a short query. Definitely more of this in more complex scenarios

---

## Submission Instructions

1. Task 1 — Chat message burst clustering (4/5)
2. Task 2 — All subordinates of a manager (3/5)
3. Task 3 — Ticket priority PIVOT by status (3/5)
n### Task Archive: 2026-03-24 (Week 15, Day 2)n
# Daily SQL Practice Tasks

**Generated:** 2026-03-23
**Week 15, Day 1 Focus:** Time-Proximity Variant + PIVOT Complex + Anti-Join with Subquery Twist

---

## Task 1: Time-Proximity Gaps-and-Islands — Order Bursts per User

**Scenario:**
The operations team wants to identify "ordering bursts" — periods where a user places multiple orders in quick succession. Define a burst as a sequence of orders where each consecutive order arrives within **2 hours** of the previous one (per user).

For each burst show:

**Expected Output Columns:**
- `user_id` (integer)
- `burst_id` (bigint) — sequential per user (1, 2, 3...)
- `burst_start` (timestamp)
- `burst_end` (timestamp)
- `order_count` (bigint)
- `burst_revenue` (numeric) — total order amount in the burst, rounded to 2 decimals

**Requirements:**
- Use `orders` table, exclude NULL amounts
- Gap threshold: 2 hours between consecutive orders per user
- LAG → is_new_burst flag → SUM() OVER burst_key → GROUP BY pattern
- Only include bursts with at least 2 orders
- Order by `user_id ASC`, `burst_start ASC`

**Difficulty Rating:** 4/5

WITH users_orders AS (
SELECT 
	*,
	LAG(o.created_at) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_order_time
FROM crappy_data_db.orders o
),
users_streak_beginnings AS (
SELECT 
	*,
	CASE WHEN prev_order_time IS NULL OR created_at - prev_order_time > INTERVAL '2 HOURS' THEN 1 ELSE 0 END AS is_new_streak
FROM users_orders
),
users_streak_keys AS (
SELECT 
	*,
	SUM(is_new_streak) OVER (PARTITION BY user_id ORDER BY created_at) AS streak_key
FROM users_streak_beginnings
),
users_order_streaks AS (
SELECT 
	user_id,
	streak_key AS burst_id,
	MIN(created_at) AS burst_start,
	MAX(created_at) AS burst_end,
	COUNT(*) AS order_count,
	SUM(amount) AS burst_revenue
FROM users_streak_keys
GROUP BY user_id, streak_key
)
SELECT 
*
FROM users_order_streaks
WHERE order_count > 1
ORDER BY user_id, burst_start


Very useful task - this is definitely a pattern I want to practice.
Here there were only 12 such bursts in total, but still - very useful pattern to practice in more and more context and more and more advanced tasks with different data.


---

## Task 2: PIVOT — User Age Group × Order Frequency Matrix

**Scenario:**
The analytics team wants a cross-tab matrix showing how many users fall into each combination of age group and order frequency bucket.

Age groups:
- `under_30`: age < 30
- `30_to_50`: age between 30 and 50
- `over_50`: age > 50

Order frequency buckets (total orders per user):
- `one_time`: exactly 1 order
- `occasional`: 2–4 orders
- `regular`: 5+ orders

**Expected Output Columns:**
- `age_group` (text)
- `one_time` (bigint)
- `occasional` (bigint)
- `regular` (bigint)
- `total_users` (bigint)

**Requirements:**
- Use `users` and `orders` tables
- Exclude users with NULL age
- Users with 0 orders are NOT included (only users who appear in orders)
- Use conditional aggregation for the pivot columns
- Order by `age_group ASC`

**Difficulty Rating:** 4/5

WITH users_orders_cnt AS (
SELECT 
	user_id,
	COUNT(*) AS orders_cnt,
	CASE WHEN COUNT(*) = 1 THEN 'one_time' WHEN COUNT (*) BETWEEN 2 AND 4 THEN 'occasional' ELSE 'regular' END AS frequency_bucket
FROM crappy_data_db.orders o
GROUP BY o.user_id
),
users_orders_age AS (
SELECT 
	*,
	CASE WHEN u.age < 30 THEN 'under_30' WHEN age BETWEEN 30 AND 50 THEN '30_to_50' ELSE 'over_50' END AS age_group
FROM users_orders_cnt uo
JOIN crappy_data_db.users u ON uo.user_id = u.id
)
SELECT 
	age_group,
	COUNT(*) FILTER (WHERE frequency_bucket = 'one_time') AS one_time,
	COUNT(*) FILTER (WHERE frequency_bucket = 'occasional') AS occasional,
	COUNT(*) FILTER (WHERE frequency_bucket = 'regular') AS regular,
	COUNT(*) AS total_users
FROM users_orders_age
GROUP BY age_group
ORDER BY age_group

I really struggled with that today - this matrix did feel unintuitive.

---

## Task 3: Anti-Join — Users Who Ordered but Never Had a Delivered Order

**Scenario:**
The customer success team wants to find users who have placed at least one order, but none of their orders have ever been successfully delivered (no delivery record with `status = 'delivered'`).

Solve this using **NOT EXISTS** only — this is the safest pattern when the subquery involves NULLable joins.

**Expected Output Columns:**
- `user_id` (integer)
- `total_orders` (bigint)
- `first_order_date` (date)

**Requirements:**
- Use `users`, `orders`, `deliveries` tables
- A "delivered order" = an order that has at least one delivery record with `status = 'delivered'`
- Only include users who have at least 1 order
- Order by `total_orders DESC`, `first_order_date ASC`

**Difficulty Rating:** 4/5


WITH delivered_users AS (
SELECT
DISTINCT o2.user_id
FROM crappy_data_db.orders o1
JOIN crappy_data_db.deliveries d
ON d.status = 'delivered' AND d.order_id = o1.id
JOIN crappy_data_db.orders o2 ON o1.user_id = o2.user_id
),
non_delivered_users AS (
SELECT 
	o.user_id
FROM crappy_data_db.orders o
WHERE NOT EXISTS
(SELECT * FROM delivered_users du WHERE du.user_id = o.user_id)
)
SELECT 
	ndu.user_id,
	COUNT(*) AS total_orders,
	MIN(o.created_at) AS first_order_date
FROM non_delivered_users ndu
JOIN crappy_data_db.orders o ON ndu.user_id = o.user_id
GROUP BY ndu.user_id


---

## Submission Instructions

1. Task 1 — Order burst clustering (4/5)
2. Task 2 — Age group × order frequency PIVOT matrix (4/5)
3. Task 3 — Anti-join: users with no delivered orders (4/5)
n### Task Archive: 2026-03-25 (Week 15, Day 3)n
# Daily SQL Practice Tasks

**Generated:** 2026-03-24
**Week 15, Day 2 Focus:** Time-Proximity Variant + PIVOT + Self-Referencing CTE + Anti-Join Complex

---

## Task 1: Time-Proximity — Support Ticket Response Bursts

**Scenario:**
The support team wants to identify "response bursts" — periods within a ticket where messages arrive rapidly. Define a burst as a sequence of messages within the same ticket where each consecutive message arrives within **15 minutes** of the previous one.

For each burst show:

**Expected Output Columns:**
- `ticket_id` (bigint)
- `burst_id` (bigint) — sequential per ticket (1, 2, 3...)
- `burst_start` (timestamp)
- `burst_end` (timestamp)
- `message_count` (bigint)
- `duration_minutes` (numeric) — minutes from burst_start to burst_end, rounded to 1 decimal

**Requirements:**
- Use `chat_messages` table, `message_type = 'text'` only
- Gap threshold: 15 minutes between consecutive messages per ticket
- LAG → is_new_burst flag → SUM() OVER → GROUP BY pattern
- Only include bursts with at least 2 messages
- Order by `ticket_id ASC`, `burst_start ASC`

**Difficulty Rating:** 4/5

WITH tickets_msg_prev_responses AS (
SELECT 
	*,
	LAG(cm.created_at) OVER (PARTITION BY ticket_id ORDER BY created_at) AS prev_ticket_response
FROM crappy_data_db.chat_messages cm
WHERE message_type = 'text'
),
msgs_is_streaks AS (
SELECT 
	*,
	CASE WHEN prev_ticket_response IS NULL OR created_at - prev_ticket_response > INTERVAL '15 Minutes' THEN 1 ELSE 0 END AS is_new_streak
FROM tickets_msg_prev_responses
),
msgs_streak_ids AS (
SELECT 
	*,
	SUM(is_new_streak) OVER (PARTITION BY ticket_id ORDER BY created_at) AS streak_id
FROM msgs_is_streaks
)
SELECT
	ticket_id,
	streak_id,
	MIN(created_at) AS streak_start,
	MAX(created_at) AS streak_end,
	COUNT(*) AS message_count,
	EXTRACT('Minute' FROM MAX(created_at) - MIN(created_at)) AS duration_minutes
FROM msgs_streak_ids
GROUP BY ticket_id, streak_id
ORDER BY ticket_id, streak_start


I've changed the names from burst to streak, as it sounds more natural and matching here. DO NOT take away points for that.

---

## Task 2: Self-Referencing CTE — Find All Ancestors of a Node

**Scenario:**
Given the same category tree as before, write a query that finds **all ancestors** of a given node — traversing **upward** from child to parent, instead of the usual top-down direction.

Use this inline data:

```sql
WITH RECURSIVE categories (id, name, parent_id) AS (
    VALUES
    (1, 'All Products',  NULL::int),
    (2, 'Electronics',   1),
    (3, 'Clothing',      1),
    (4, 'Phones',        2),
    (5, 'Laptops',       2),
    (6, 'Men',           3),
    (7, 'Women',         3),
    (8, 'iPhone',        4),
    (9, 'Samsung',       4),
    (10, 'T-Shirts',     6)
)
```

Find all ancestors of node `id = 8` (iPhone). The result should include: Phones → Electronics → All Products.

**Expected Output Columns:**
- `id` (integer)
- `name` (text)
- `parent_id` (integer)
- `depth` (integer) — 1 = direct parent, 2 = grandparent, etc.

**Requirements:**
- Anchor: start with the direct parent of node 8 (`WHERE id = (SELECT parent_id FROM categories WHERE id = 8)`)
- Recursive: JOIN categories on `categories.id = cte.parent_id`
- Natural termination when parent_id IS NULL (root reached)
- Order by `depth ASC`

**Difficulty Rating:** 4/5

WITH RECURSIVE categories (id, name, parent_id) AS (
    VALUES
    (1, 'All Products',  NULL::int),
    (2, 'Electronics',   1),
    (3, 'Clothing',      1),
    (4, 'Phones',        2),
    (5, 'Laptops',       2),
    (6, 'Men',           3),
    (7, 'Women',         3),
    (8, 'iPhone',        4),
    (9, 'Samsung',       4),
    (10, 'T-Shirts',     6)
),
HIERARCHY AS (
SELECT 
	*,
	1 AS depth
FROM categories
WHERE id = 8
UNION ALL
SELECT 
c.id,
c.name,
c.parent_id,
h.DEPTH + 1
FROM HIERARCHY h
JOIN categories c ON h.parent_id = c.id
)
SELECT * FROM HIERARCHY
ORDER BY DEPTH

---

## Task 3: PIVOT + Anti-Join — Monthly Revenue from Users Who Never Had a Delivered Order

**Scenario:**
Combine what you've learned: first identify users who have never had a delivered order (anti-join), then pivot their monthly order revenue by the delivery status of their orders.

**Expected Output Columns:**
- `month` (date) — truncated to month
- `pending_revenue` (numeric) — rounded to 2 decimals
- `total_revenue` (numeric) — rounded to 2 decimals
- `user_count` (bigint) — distinct users contributing that month

**Requirements:**
- Use `users`, `orders`, `deliveries`
- First isolate users with no delivered orders using NOT EXISTS
- Then join to their orders and deliveries to pivot revenue by status
- Only include `status = 'pending'` for the pivot column (it will be the only status for these users)
- Order by `month ASC`

**Difficulty Rating:** 5/5

WITH users_without_delivered_orders AS (
SELECT o2.user_id 
FROM crappy_data_db.orders o2
WHERE NOT EXISTS (
SELECT 
	DISTINCT(user_id)
FROM crappy_data_db.deliveries d 
JOIN crappy_data_db.orders o ON d.order_id = o.id AND o.user_id = o2.user_id
WHERE d.status = 'delivered'
)
),
users_orders AS (
SELECT 
	uw.user_id AS USER,
	*,
	DATE_TRUNC('Month', o.created_at) AS month_
FROM users_without_delivered_orders uw
JOIN crappy_data_db.orders o ON uw.user_id = o.user_id
JOIN crappy_data_db.deliveries d ON o.id = d.order_id
)
SELECT 
	month_,
	SUM(amount) FILTER (WHERE status = 'pending') AS pending_revenue,
	SUM(amount) AS total_revenue,
	COUNT(DISTINCT(uo.user)) AS user_count
FROM users_orders uo
GROUP BY month_
ORDER BY month_

---

## Submission Instructions

1. Task 1 — Support ticket response bursts (4/5)
2. Task 2 — Self-referencing CTE traversal upward (4/5)
3. Task 3 — PIVOT + anti-join combined (5/5)
n### Task Archive: 2026-03-26 (Week 15, Day 4)n
# Daily SQL Practice Tasks

**Generated:** 2026-03-25
**Week 15, Day 3 Focus:** Time-Proximity Edge Cases + PIVOT Complex + Anti-Join Combo

---

## Task 1: Time-Proximity — User Transaction Sessions with Edge Cases

**Scenario:**
Group each user's transactions into sessions where consecutive transactions are within **30 minutes** of each other. This dataset is designed to test edge cases — pay attention to transactions that straddle the hour boundary.

Use this inline data:

```sql
WITH events (user_id, event_time, amount) AS (
    VALUES
    (1, '2024-01-01 08:00'::timestamp, 100),
    (1, '2024-01-01 08:25'::timestamp, 200),
    (1, '2024-01-01 08:52'::timestamp, 150),
    (1, '2024-01-01 09:18'::timestamp, 300),
    (1, '2024-01-01 10:00'::timestamp, 50),
    (1, '2024-01-01 10:20'::timestamp, 75),
    (2, '2024-01-01 09:00'::timestamp, 500),
    (2, '2024-01-01 09:28'::timestamp, 200),
    (2, '2024-01-01 10:05'::timestamp, 100)
)
```

Note: user 1's events at 08:52 and 09:18 are 26 minutes apart — same session. Events at 09:18 and 10:00 are 42 minutes apart — new session.

**Expected Output Columns:**
- `user_id` (integer)
- `session_id` (bigint) — sequential per user (1, 2, 3...)
- `session_start` (timestamp)
- `session_end` (timestamp)
- `event_count` (bigint)
- `total_amount` (numeric)
- `duration_minutes` (numeric) — use EPOCH, rounded to 1 decimal

**Requirements:**
- LAG → is_new_session flag → SUM() OVER → GROUP BY pattern
- Use `EXTRACT(EPOCH FROM ...) / 60` for duration
- Order by `user_id ASC`, `session_start ASC`

**Difficulty Rating:** 4/5

WITH events (user_id, event_time, amount) AS (
    VALUES
    (1, '2024-01-01 08:00'::timestamp, 100),
    (1, '2024-01-01 08:25'::timestamp, 200),
    (1, '2024-01-01 08:52'::timestamp, 150),
    (1, '2024-01-01 09:18'::timestamp, 300),
    (1, '2024-01-01 10:00'::timestamp, 50),
    (1, '2024-01-01 10:20'::timestamp, 75),
    (2, '2024-01-01 09:00'::timestamp, 500),
    (2, '2024-01-01 09:28'::timestamp, 200),
    (2, '2024-01-01 10:05'::timestamp, 100)
),
events_lag AS (
SELECT 
	*,
	LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) AS prev_event_time
FROM events
),
events_new_sessions AS (
SELECT 
	*,
	CASE WHEN prev_event_time IS NULL OR event_time - prev_event_time > INTERVAL '30 Minutes' THEN 1 ELSE 0 END AS is_new_session
FROM events_lag
),
event_session_ids AS (
SELECT
	*,
	SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY event_time) AS session_id
FROM events_new_sessions
)
SELECT 
	user_id,
	session_id,
	MIN(event_time) AS session_start,
	MAX(event_time) AS session_end,
	COUNT(*) AS event_count,
	SUM(amount) AS total_amount,
	EXTRACT(EPOCH FROM MAX(event_time) - MIN(event_time)) / 60 AS duration_minutes 
FROM event_session_ids
GROUP BY user_id, session_id
ORDER BY USER_id, session_start


Again, I love this pattern and find it very useful.




---

## Task 2: PIVOT — Delivery Status Revenue by Product Category

**Scenario:**
The operations team wants a revenue breakdown showing, for each product category, how much revenue is associated with each delivery status.

**Expected Output Columns:**
- `category_name` (text)
- `pending_revenue` (numeric) — rounded to 2 decimals
- `delivered_revenue` (numeric) — rounded to 2 decimals
- `total_revenue` (numeric) — rounded to 2 decimals

**Requirements:**
- Use `orders`, `orders_products`, `products`, `product_categories`, `deliveries`
- Revenue = `quantity × price`
- Join orders to deliveries to get status
- PIVOT on delivery status using `SUM(...) FILTER (WHERE status = '...')`
- Order by `total_revenue DESC`

**Difficulty Rating:** 4/5

SELECT 
	pc."name" AS category_name,
	SUM(p.price * op.quantity) FILTER (WHERE d.status = 'pending') AS pending_revenue,
	SUM(p.price * op.quantity) FILTER (WHERE d.status = 'delivered') AS delivered_revenue,
	SUM(p.price * op.quantity) AS total_revenue
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.products p ON op.product_id = p.id
JOIN crappy_data_db.product_categories pc ON p.category_id = pc.id
JOIN crappy_data_db.deliveries d ON d.order_id = op.order_id
GROUP BY pc."name"
ORDER BY total_revenue DESC

I followed your instructions at first, but I quickly realized that this is weird AND VERY DANGEROUS to calculate the total revenue with the deliveries JOINED, as effectively it multiplies the total revenue by every delivery status that IS THERE. IT'S ERRATIC WHAT YOU WANTED ME TO DO THERE. I've added another CTE in a reverse logic WITHOUT ADDING DELIVERIES to avoid that bias to calculate total_revenue.


WITH pending_delivered_revenues AS (
SELECT 
	pc."name" AS category_name,
	SUM(p.price * op.quantity) FILTER (WHERE d.status = 'pending') AS pending_revenue,
	SUM(p.price * op.quantity) FILTER (WHERE d.status = 'delivered') AS delivered_revenue
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.products p ON op.product_id = p.id
JOIN crappy_data_db.product_categories pc ON p.category_id = pc.id
JOIN crappy_data_db.deliveries d ON d.order_id = op.order_id
GROUP BY pc."name"
)
SELECT 
	pdr.category_name,
	pdr.delivered_revenue,
	pdr.pending_revenue,
	SUM(p.price * op.quantity) AS total_revenue
FROM pending_delivered_revenues pdr
JOIN crappy_data_db.product_categories pc ON pc."name" = pdr.category_name
JOIN crappy_data_db.products p ON p.category_id = pc.id
JOIN crappy_data_db.orders_products op ON p.id = op.product_id
GROUP BY pdr.category_name, pdr.delivered_revenue, pdr.pending_revenue

It turns out that total_revenue is equal to pending_revenue, which makes total sense.
delivered_revenue is smaller and it also makes sense as not all of the orders were delivered successfully.

Also, mind that I AM AWARE THAT YOU WANTED the results rounded to 2 decimals, AND THEY ARE ROUNDED TO 2 DECIMALS, so I didn't add the unnecessary ROUND there.

---

## Task 3: Anti-Join — Categories With No Sales in the Last 3 Months

**Scenario:**
The product team wants to identify product categories that have had zero sales in the last 3 months (relative to the most recent order date in the dataset — do not use `NOW()`).

**Expected Output Columns:**
- `category_id` (integer)
- `category_name` (text)
- `last_sale_date` (date) — most recent sale date for this category across all time, NULL if never sold

**Requirements:**
- Use `product_categories`, `products`, `orders_products`, `orders`
- "Last 3 months" = within 3 months before the most recent order date in the dataset
- Use `NOT EXISTS` to identify categories with no sales in that window
- Include `last_sale_date` — if a category has older sales but none recently, show when it last sold
- Order by `last_sale_date ASC NULLS FIRST`

**Difficulty Rating:** 5/5

THERE ARE NO SUCH CATEGORIES.
I've used a different approach here but it works just fine.


WITH categories_last_orders AS (
SELECT 
	pc.name AS category_name,
	MAX(o.created_at) AS last_order_time
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.orders o ON op.order_id = o.id
JOIN crappy_data_db.products p ON p.id = op.product_id
JOIN crappy_data_db.product_categories pc ON pc.id = p.category_id
GROUP BY pc.name
),
qualified_orders AS (
SELECT 
	* 
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.orders o ON op.order_id = o.id
JOIN crappy_data_db.products p ON p.id = op.product_id
JOIN crappy_data_db.product_categories pc ON pc.id = p.category_id
JOIN categories_last_orders clo ON clo.category_name = pc."name"
WHERE last_order_time - o.created_at < INTERVAL '3 Months'
)
SELECT 
	DISTINCT category_id, category_name, last_order_time
FROM qualified_orders

---

## Submission Instructions

1. Task 1 — Transaction session grouping with edge cases (4/5)
2. Task 2 — Delivery status revenue PIVOT by category (4/5)
3. Task 3 — Anti-join: categories with no recent sales (5/5)
n### Task Archive: 2026-03-27 (Week 15, Day 5)n
# Daily SQL Practice Tasks

**Generated:** 2026-03-26
**Week 15, Day 4 Focus:** Time-Proximity Variant + PIVOT Multi-Dimension + Anti-Join Complex

---

## Task 1: Time-Proximity — Order Bursts per User with Variable Threshold

**Scenario:**
Group each user's orders into bursts where consecutive orders are placed within **3 days** of each other. This tests the pattern at day granularity rather than minutes.

Use the `orders` table from the real database.

**Expected Output Columns:**
- `user_id` (integer)
- `burst_id` (bigint) — sequential per user (1, 2, 3...)
- `burst_start` (date)
- `burst_end` (date)
- `order_count` (bigint)
- `total_revenue` (numeric) — rounded to 2 decimals
- `duration_days` (integer) — days between burst_start and burst_end

**Requirements:**
- Use `DATE(created_at)` to work at day granularity
- LAG → is_new_burst flag (gap > 3 days) → SUM() OVER → GROUP BY
- Only show bursts with at least 2 orders
- Order by `user_id ASC`, `burst_start ASC`

**Difficulty Rating:** 4/5

WITH orders_users AS (
SELECT 
	*,
	LAG(o.created_at) OVER (PARTITION BY user_id ORDER BY o.created_at) AS prev_order_time
FROM crappy_data_db.orders o
),
orders_users_is_streak AS (
SELECT 
	*,
	CASE WHEN prev_order_time IS NULL OR created_at - prev_order_time > INTERVAL '3 Days' THEN 1 ELSE 0 END AS is_new_streak
FROM orders_users
),
orders_users_streaks_id AS (
SELECT 
	*,
	SUM(is_new_streak) OVER (PARTITION BY user_id ORDER BY created_at) AS streak_id
FROM orders_users_is_streak
),
users_order_streaks AS (
SELECT 
	user_id,
	streak_id,
	MIN(created_at) AS streak_start,
	MAX(created_at) AS streak_end,
	COUNT(*) AS order_count,
	SUM(amount) AS total_revenue,
	ROUND(EXTRACT(EPOCH FROM MAX(created_at) - MIN(created_at)) / 86400, 2) AS duration_days
FROM orders_users_streaks_id
GROUP BY user_id, streak_id
)
SELECT * FROM users_order_streaks
WHERE order_count >= 2
ORDER BY user_id, streak_start


Lovely pattern, feels like I've mastered it already more or less.
Also I changed the column names a bit, as streak sounds a bit more natural imo than burst in this context.

---

## Task 2: PIVOT — User Age Group × Transaction Type Revenue Matrix

**Scenario:**
The finance team wants a full matrix showing total transaction revenue broken down by both age group (rows) and transaction type (columns).

**Expected Output Columns:**
- `age_group` (text) — `'under_30'`, `'30_to_50'`, `'over_50'`
- `deposit` (numeric) — rounded to 2 decimals
- `withdrawal` (numeric) — rounded to 2 decimals
- `payment` (numeric) — rounded to 2 decimals
- `transfer` (numeric) — rounded to 2 decimals
- `purchase` (numeric) — rounded to 2 decimals
- `total` (numeric) — rounded to 2 decimals

**Requirements:**
- Use `transactions` and `users` tables
- Join on `user_id`, exclude NULL ages and NULL amounts
- Age groups: `age < 30` → `'under_30'`, `30-50` → `'30_to_50'`, `> 50` → `'over_50'`
- Use `SUM(amount) FILTER (WHERE type = '...')` per column
- Order by `age_group ASC`

**Difficulty Rating:** 4/5

WITH users_age_groups AS (
SELECT 
	id AS users_user_id,
	CASE WHEN age < 30 THEN 'under_30'
	WHEN age >= 30 AND age <= 50 THEN '30_to_50' ELSE 'over_50' END AS age_group
FROM crappy_data_db.users u
WHERE age IS NOT NULL
)
SELECT 
	age_group,
	sum(t.amount) FILTER (WHERE TYPE = 'deposit') AS deposit_sum,
	sum(t.amount) FILTER (WHERE TYPE = 'withdrawal') AS withdrawal_sum,
	sum(t.amount) FILTER (WHERE TYPE = 'payment') AS payment_sum,
	sum(t.amount) FILTER (WHERE TYPE = 'transfer') AS transfer_sum,
	sum(t.amount) FILTER (WHERE TYPE = 'purchase') AS purchase_sum,
	sum(t.amount) AS total
FROM crappy_data_db.transactions t
JOIN users_age_groups uag ON t.user_id = uag.users_user_id
WHERE t.amount IS NOT NULL
GROUP BY age_group


Works perfectly imo, no needed to round anything, as the values are already rounded



---

## Task 3: Anti-Join — Users Who Ordered But Never Sent a Support Ticket

**Scenario:**
The CX team wants to identify "silent" customers — users who have placed at least one order but have never opened a support ticket. These users may have had issues they never reported.

**Expected Output Columns:**
- `user_id` (integer)
- `total_orders` (bigint)
- `total_order_revenue` (numeric) — rounded to 2 decimals
- `first_order_date` (date)
- `last_order_date` (date)

**Requirements:**
- Use `orders` and `chat_tickets` tables
- Use `NOT EXISTS` to exclude users who have any ticket
- Only include users with at least 1 order
- Order by `total_orders DESC`, `total_order_revenue DESC`

**Difficulty Rating:** 4/5

WITH ticketless_users_with_orders AS (
SELECT
	DISTINCT o.user_id
FROM crappy_data_db.orders o
WHERE NOT EXISTS
(
SELECT 
	user_id 
FROM crappy_data_db.chat_tickets ct
WHERE ct.user_id = o.user_id
)
)
SELECT 
	tu.user_id,
	COUNT(o.id) AS total_orders,
	SUM(o.amount) AS total_order_revenue,
	MIN(o.created_at) AS first_order_date,
	MAX(o.created_at) AS last_order_date
FROM ticketless_users_with_orders tu
JOIN crappy_data_db.orders o ON tu.user_id = o.user_id
GROUP BY tu.user_id
ORDER BY total_orders DESC, total_order_revenue DESC


---

## Submission Instructions

1. Task 1 — Order bursts at day granularity (4/5)
2. Task 2 — Age group × transaction type revenue matrix (4/5)
3. Task 3 — Anti-join: ordered but never ticketed users (4/5)
n### Task Archive: 2026-03-30 (Week 16, Day 1)n
# Daily SQL Practice Tasks

**Generated:** 2026-03-27
**Week 15, Day 5 Focus:** Review Day — Gaps & Islands + Window Functions + Recursive CTE (Type A)

---

## Task 1: Gaps & Islands — Monthly Active User Streaks

**Scenario:**
Find each user's consecutive months of activity (at least 1 order placed). Report streaks of 2+ consecutive months.

**Expected Output Columns:**
- `user_id` (integer)
- `streak_start` (text) — first month in streak, formatted as `'YYYY-MM'`
- `streak_end` (text) — last month in streak, formatted as `'YYYY-MM'`
- `streak_length` (bigint) — number of consecutive months

**Requirements:**
- Use `orders` table
- Derive active months using `DATE_TRUNC('month', created_at)`
- Use the classic gaps-and-islands pattern: `ROW_NUMBER()` subtraction to group consecutive months
- Only include streaks of 2+ months
- Order by `streak_length DESC`, `user_id ASC`

**Difficulty Rating:** 3/5


WITH orders_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
FROM crappy_data_db.orders
),
users_months_orders AS (
SELECT 
	user_id,
	month_,
	COUNT(*) AS orders_cnt
FROM orders_months
GROUP BY user_id, month_
ORDER BY user_id
),
orders_months_rn AS (
SELECT 
	*,
	ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY month_) AS rn
FROM users_months_orders
),
users_streak_ids AS (
SELECT 
	*,
	month_ - rn * INTERVAL '1 Month' AS streak_id
FROM orders_months_rn
),
users_streaks AS (
SELECT 
	user_id,
	MIN(month_) AS streak_start,
	MAX(month_) AS streak_end,
	COUNT(*) AS streak_length
	FROM users_streak_ids
GROUP BY user_id, streak_id
ORDER BY streak_length DESC, user_id
)
SELECT * FROM users_streaks WHERE streak_length >= 2


---

## Task 2: Window Functions — Top Spender Per Product Category

**Scenario:**
For each product category, find the top 3 users by total spend. Include their rank and total spend.

**Expected Output Columns:**
- `category_name` (text)
- `user_id` (integer)
- `rank` (bigint)
- `total_spend` (numeric) — rounded to 2 decimals

**Requirements:**
- Use `orders`, `orders_products`, `products`, `product_categories`
- Revenue = `quantity × price`
- Use `RANK()` window function partitioned by category
- Only return ranks 1–3
- Order by `category_name ASC`, `rank ASC`

**Difficulty Rating:** 3/5


WITH users_categories_spendings AS (
SELECT 
	pc.name AS category_name,
	o.user_id,
	SUM(p.price * op.quantity) AS category_spent
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.products p ON OP.product_id = p.id
JOIN crappy_data_db.product_categories pc ON pc.id = p.category_id
JOIN crappy_data_db.orders o ON o.id = op.order_id
GROUP BY pc.name, o.user_id
),
users_spending_rank AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY category_name ORDER BY category_spent DESC) AS rank
FROM users_categories_spendings
)
SELECT * FROM users_spending_rank 
WHERE RANK <= 3
ORDER BY category_name, rank

I thought category_spent IS WAY MORE logical, as total_spent indicates that these are total user's spendings. Anyway, it's good

---


## Task 3: Recursive CTE (Type A) — 3-Level Category Hierarchy Summary

**Scenario:**
The product team wants a summary of the category hierarchy: for each top-level category, show its direct subcategories and the total number of products at leaf level.

**Expected Output Columns:**
- `top_level` (text) — name of root category (parent_id IS NULL)
- `sub_level` (text) — name of direct child category
- `product_count` (bigint) — total products belonging to the sub_level category

**Requirements:**
- Use `product_categories` and `products` tables
- Fixed 3-level structure: root → subcategory → products
- Build with CTEs: root categories, subcategories, then join to products
- Order by `top_level ASC`, `product_count DESC`

**Difficulty Rating:** 3/5

Stupid question, THERE IS NO SUCH STRUCTURE IN MY DATABASE.
AVOIDING IT, DON'T PUNISH ME FOR IT!

---

## Submission Instructions

1. Task 1 — Monthly active user streaks (3/5)
2. Task 2 — Top 3 spenders per category (3/5)
3. Task 3 — 3-level category hierarchy summary (3/5)
n### Task Archive: 2026-03-31 (Week 16, Day 2)n
# Daily SQL Practice Tasks

**Generated:** 2026-03-30
**Week 16, Day 1 Focus:** Type B Recursive CTE + Anti-Join NULL Trap + Time-Proximity Variant

---

## Task 1: Type B Recursive CTE — Full Referral Chain Traversal

**Scenario:**
Use this inline referral data to traverse the full referral tree — unlimited depth, natural termination:

```sql
WITH RECURSIVE referrals (id, name, referred_by) AS (
    VALUES
    (1,  'Alice',   NULL::int),
    (2,  'Bob',     1),
    (3,  'Carol',   1),
    (4,  'Dave',    2),
    (5,  'Eve',     2),
    (6,  'Frank',   3),
    (7,  'Grace',   4),
    (8,  'Hank',    4),
    (9,  'Ivy',     6),
    (10, 'Jack',    9)
)
```

For each person, show their full path from root and their depth level.

**Expected Output Columns:**
- `id` (integer)
- `name` (text)
- `referred_by` (integer)
- `depth` (integer) — 1 for root
- `path` (text) — e.g. `'Alice -> Bob -> Dave -> Grace'`

**Requirements:**
- Anchor: `WHERE referred_by IS NULL` — do NOT hardcode the root
- Recursive: `JOIN referrals ON referrals.referred_by = cte.id`
- Path separator: ` -> `
- Natural termination — no LEVEL limit
- Order by `path ASC`

**Difficulty Rating:** 3/5

WITH RECURSIVE referrals (id, name, referred_by) AS (
    VALUES
    (1,  'Alice',   NULL::int),
    (2,  'Bob',     1),
    (3,  'Carol',   1),
    (4,  'Dave',    2),
    (5,  'Eve',     2),
    (6,  'Frank',   3),
    (7,  'Grace',   4),
    (8,  'Hank',    4),
    (9,  'Ivy',     6),
    (10, 'Jack',    9)
),
HIERARCHY AS (
SELECT 
	*,
	1 AS DEPTH,
	'Alice' AS path
FROM referrals 
WHERE referred_by IS NULL
UNION ALL
SELECT 
	r.id,
	r.name,
	r.referred_by,
	h.DEPTH + 1,
	h.PATH || '->' || R.NAME
FROM HIERARCHY h JOIN referrals r ON h.id = r.referred_by
)
SELECT * FROM hierarchy


---

## Task 2: Anti-Join NULL Trap — The NOT IN Failure

**Scenario:**
This task demonstrates why `NOT IN` breaks silently when the subquery contains NULLs.

Use this inline data:

```sql
WITH all_users (user_id, name) AS (
    VALUES (1, 'Alice'), (2, 'Bob'), (3, 'Carol'), (4, 'Dave'), (5, 'Eve')
),
orders (order_id, user_id) AS (
    VALUES (101, 1), (102, 2), (103, NULL), (104, 4)
)
```

**Step A:** Write `NOT IN` to find users with no orders. Observe the result — it returns 0 rows despite user 3 (Carol) and user 5 (Eve) clearly having no orders.

WITH all_users (user_id, name) AS (
    VALUES (1, 'Alice'), (2, 'Bob'), (3, 'Carol'), (4, 'Dave'), (5, 'Eve')
),
orders (order_id, user_id) AS (
    VALUES (101, 1), (102, 2), (103, NULL), (104, 4)
)
SELECT DISTINCT user_id FROM all_users
WHERE user_id NOT IN (SELECT DISTINCT user_id FROM orders)

It shows no users, as you've said

BUT:

WITH all_users (user_id, name) AS (
    VALUES (1, 'Alice'), (2, 'Bob'), (3, 'Carol'), (4, 'Dave'), (5, 'Eve')
),
orders (order_id, user_id) AS (
    VALUES (101, 1), (102, 2), (103, NULL), (104, 4)
)
SELECT DISTINCT user_id FROM all_users
WHERE user_id NOT IN (SELECT DISTINCT user_id FROM orders WHERE user_id IS NOT NULL)

Now it correctly shows id 3 and 5, so this is just one caveat to remember, that we need IS NOT NULL here!


**Step B:** Write `NOT EXISTS` for the same question. Observe it returns Carol and Eve correctly.

WITH all_users (user_id, name) AS (
    VALUES (1, 'Alice'), (2, 'Bob'), (3, 'Carol'), (4, 'Dave'), (5, 'Eve')
),
orders (order_id, user_id) AS (
    VALUES (101, 1), (102, 2), (103, NULL), (104, 4)
)
SELECT DISTINCT au.user_id FROM all_users au
WHERE NOT EXISTS (SELECT DISTINCT user_id FROM orders WHERE user_id = au.user_id)


**Step C:** Write `LEFT JOIN ... WHERE IS NULL` for the same question.

WITH all_users (user_id, name) AS (
    VALUES (1, 'Alice'), (2, 'Bob'), (3, 'Carol'), (4, 'Dave'), (5, 'Eve')
),
orders (order_id, user_id) AS (
    VALUES (101, 1), (102, 2), (103, NULL), (104, 4)
)
SELECT DISTINCT au.user_id FROM all_users au
LEFT JOIN orders o ON au.user_id = o.user_id WHERE o.user_id IS NULL

But honestly, THIS solution seems the most unnatural to me. It looks weird and unnatural. I'd rather omit it in the future.


**Step D:** Explain in a comment WHY `NOT IN` fails here.

I already explained, that not necessarily it has to fail.
It fails because NULL in the subquery disqualifies all of user_ids from the original table.

**Expected Output** (Steps B and C):
- `user_id` (integer)
- `name` (text)

**Difficulty Rating:** 3/5

---

## Task 3: Time-Proximity — Support Ticket Message Bursts on Real Data

**Scenario:**
Group `chat_messages` per ticket into conversation bursts where consecutive messages (any type) are within **20 minutes** of each other. Find tickets with at least 3 bursts.

**Expected Output Columns:**
- `ticket_id` (bigint)
- `burst_count` (bigint) — number of bursts in this ticket
- `total_messages` (bigint)
- `first_message_at` (timestamp)
- `last_message_at` (timestamp)

**Requirements:**
- Use `chat_messages` table
- Include all message types (no filter on message_type)
- LAG → is_new_burst flag (gap > 20 min) → SUM() OVER → aggregate per ticket
- Only include tickets with burst_count >= 3
- Order by `burst_count DESC`, `total_messages DESC`

**Difficulty Rating:** 4/5

WITH ticket_messages AS (
SELECT 
	*,
	LAG(cm.created_at) OVER (PARTITION BY ticket_id ORDER BY created_at) AS prev_ticket_time
FROM crappy_data_db.chat_messages cm
WHERE cm.message_type = 'text'
),
ticket_streak_starts AS (
SELECT 
	*,
	CASE WHEN prev_ticket_time IS NULL OR created_at - prev_ticket_time > INTERVAL '20 Minutes' THEN 1 ELSE 0 END AS is_start
FROM ticket_messages
),
ticket_streak_ids AS (
SELECT 
	*,
	SUM(is_start) OVER (PARTITION BY ticket_id ORDER BY created_at) AS streak_id
FROM ticket_streak_starts
),
ticket_streaks AS (
SELECT 
	ticket_id,
	streak_id,
	COUNT(*) AS total_messages,
	MIN(created_at) AS first_message_at,
	MAX(created_at) AS last_message_at
FROM ticket_streak_ids
GROUP BY ticket_id, streak_id
)
SELECT 
	ticket_id,
	COUNT(*) AS streak_count,
	SUM(total_messages) AS total_messages,
	MIN(first_message_at) AS first_message_at,
	MAX(last_message_at) AS last_message_at
FROM ticket_streaks
GROUP BY ticket_id
ORDER BY streak_count DESC, total_messages DESC

This was a bit unintuitive.
We didn't HAVE A SINGLE ticket with more than 2 streaks, so I kept data the way it is.


---

## Submission Instructions

1. Task 1 — Type B recursive CTE referral chain (3/5)
2. Task 2 — NOT IN NULL trap — all four steps (3/5)
3. Task 3 — Chat message bursts on real data (4/5)
n### Task Archive: 2026-04-01 (Week 16, Day 3)n
# Daily SQL Practice Tasks

**Generated:** 2026-03-31
**Week 16, Day 2 Focus:** NULLIF + Advanced Window Functions + Complex GROUP BY

---

## Task 1: NULLIF — Clean Revenue Averages

**Scenario:**
The finance team wants average transaction amounts per type, but the data contains some transactions with amount = 0 (placeholder entries that should be excluded from averages, not treated as real zero-value transactions).

Calculate the average transaction amount per type, excluding both NULLs and zero-amount entries.

Then show what the average would be **including** zeros — so both values appear side by side to demonstrate the difference.

**Expected Output Columns:**
- `type` (text)
- `avg_excl_zeros` (numeric) — average excluding 0-amount entries, rounded to 2 decimals
- `avg_incl_zeros` (numeric) — average including 0-amount entries (but still excluding NULLs), rounded to 2 decimals
- `zero_count` (bigint) — how many zero-amount entries exist for this type
- `transaction_count` (bigint) — total non-NULL transactions for this type

**Requirements:**
- Use `transactions` table
- Use `AVG(NULLIF(amount, 0))` for `avg_excl_zeros`
- Use `AVG(amount)` for `avg_incl_zeros` (NULLs excluded automatically by AVG)
- Order by `type ASC`

**Difficulty Rating:** 3/5

SELECT 
	TYPE,
	ROUND(AVG(nullif(amount, 0)), 2) AS avg_excl_zeros,
	ROUND(AVG(t.amount), 2) AS avg_incl_zeros,
	COUNT(*) AS transactions_count
FROM crappy_data_db.transactions t
GROUP BY TYPE
ORDER BY type


---

## Task 2: Advanced Window Functions — Transaction Percentile Distribution

**Scenario:**
The analytics team wants a full percentile breakdown of transaction amounts per type. For each transaction, show:
- Its percentile rank within its type (0.0 to 1.0)
- Which quartile it belongs to (1–4)
- How far it deviates from the type's mean in standard deviations (z-score)
- The running cumulative amount within its type ordered by amount ASC

**Expected Output Columns:**
- `id` (integer)
- `type` (text)
- `amount` (numeric)
- `percentile_rank` (numeric) — `PERCENT_RANK()` rounded to 3 decimals
- `quartile` (integer) — `NTILE(4)`
- `z_score` (numeric) — `(amount - AVG) / STDDEV`, rounded to 2 decimals
- `cumulative_amount` (numeric) — running SUM within type ordered by amount ASC, rounded to 2 decimals

**Requirements:**
- Use `transactions` table, exclude NULL amounts
- All window functions partitioned by `type`, ordered by `amount ASC`
- Exclude rows where `STDDEV = 0` (all amounts identical — z-score undefined)
- Order by `type ASC`, `amount ASC`

**Difficulty Rating:** 4/5

WITH transactions_perc_rank_quartiles AS (
SELECT 
	id,
	TYPE,
	amount,
	ROUND(percent_rank() OVER (PARTITION BY TYPE ORDER BY amount)::numeric, 3) AS percentile_rank,
	ntile(4) OVER (PARTITION BY TYPE ORDER BY amount DESC) AS quartile,
	stddev(amount) OVER (PARTITION BY TYPE) AS std_dev
FROM crappy_data_db.transactions t
ORDER BY percentile_rank DESC
),
transaction_types_avgs AS (
SELECT
	TYPE,
	ROUND(AVG(amount), 2) AS avg_type_amt
FROM transactions_perc_rank_quartiles
GROUP BY TYPE
)
SELECT 
	tpr.id,
	tpr.TYPE,
	tpr.amount,
	tpr.percentile_rank,
	tpr.quartile,
	(tpr.amount - tta.avg_type_amt) / std_dev AS z_score,
	SUM(tpr.amount) OVER (PARTITION BY tpr.TYPE ORDER BY tpr.amount) AS cumulative_amount
FROM transaction_types_avgs tta
JOIN transactions_perc_rank_quartiles tpr ON tta."type"  = tpr."type"


---

## Task 3: Complex GROUP BY — Order Revenue by Country and Age Group

**Scenario:**
The growth team wants to understand revenue distribution across countries and user age groups, but only for countries with at least 3 distinct ordering users.

**Expected Output Columns:**
- `country` (text)
- `age_group` (text) — `'under_30'`, `'30_to_50'`, `'over_50'`
- `order_count` (bigint)
- `total_revenue` (numeric) — rounded to 2 decimals
- `avg_revenue_per_order` (numeric) — rounded to 2 decimals, use `NULLIF` to guard against division by zero
- `pct_of_country_revenue` (numeric) — this age group's revenue as % of total country revenue, rounded to 1 decimal

**Requirements:**
- Use `orders` and `users` tables only
- Exclude NULL countries and NULL ages
- Only include countries where `COUNT(DISTINCT user_id) >= 3` — apply via HAVING
- `pct_of_country_revenue` requires a window SUM partitioned by country
- Order by `country ASC`, `total_revenue DESC`

**Difficulty Rating:** 4/5

WITH countries_user_cnt AS (
SELECT 
	country,
	COUNT(*) AS user_cnt
FROM crappy_data_db.users u
WHERE country IS NOT NULL
GROUP BY country
),
users_age_groups AS (
SELECT 
	u.id,
	u.country,
	CASE WHEN u.age < 30 THEN 'under_30' WHEN u.age >= 30 AND u.age <= 50 THEN '30_to_50' ELSE 'over_50' END AS age_group
FROM countries_user_cnt cu
JOIN crappy_data_db.users u ON cu.country = u.country
WHERE cu.user_cnt >= 3
),
orders_countries_age_stats AS (
SELECT 
	ua.country,
	ua.age_group,
	COUNT(o.id) AS order_count,
	SUM(o.amount) AS total_revenue,
	ROUND(AVG(NULLIF(o.amount, 0))::NUMERIC, 2) AS avg_revenue_per_order
FROM users_age_groups ua
JOIN crappy_data_db.orders o ON ua.id = o.user_id
GROUP BY ua.country, ua.age_group
),
countries_total_revenues AS (
SELECT 
	country,
	ROUND(SUM(total_revenue)::NUMERIC, 2) AS total_country_revenue
FROM orders_countries_age_stats
GROUP BY country
)
SELECT 
	oc.country,
	oc.age_group,
	oc.order_count,
	oc.total_revenue,
	oc.avg_revenue_per_order,
	ct.total_country_revenue,
	ROUND(oc.total_revenue::NUMERIC / ct.total_country_revenue * 100::NUMERIC, 1) AS pct_of_country_revenue
FROM orders_countries_age_stats oc
JOIN countries_total_revenues ct ON oc.country = ct.country
ORDER BY country, total_revenue DESC

I didn't need to use HAVING here, I used simple counts at the beginning instead.

---

## Submission Instructions

1. Task 1 — NULLIF clean averages (3/5)
2. Task 2 — Transaction percentile distribution (4/5)
3. Task 3 — Revenue by country and age group (4/5)
n### Task Archive: 2026-04-02 (Week 16, Day 4)n
# Daily SQL Practice Tasks

**Generated:** 2026-04-01
**Week 16, Day 3 Focus:** NULLIF in Context + Type B Recursive CTE + Time-Proximity on Real Data

---

## Task 1: NULLIF — Safe Division in Order Metrics

**Scenario:**
The operations team wants per-user order metrics, but some users have orders with NULL amounts. A careless average would silently exclude those orders, skewing the per-user stats. They also want a conversion rate (orders with amount > 0 divided by total orders) — which requires safe division.

**Expected Output Columns:**
- `user_id` (integer)
- `total_orders` (bigint) — all orders including NULL amounts
- `orders_with_amount` (bigint) — orders where amount IS NOT NULL
- `avg_order_value` (numeric) — average of non-NULL amounts only (AVG handles this naturally)
- `conversion_rate` (numeric) — `orders_with_amount / NULLIF(total_orders, 0)` as a ratio, rounded to 3 decimals
- `has_null_amounts` (boolean) — true if any order has a NULL amount

**Requirements:**
- Use `orders` table
- Use `NULLIF(total_orders, 0)` in the division to guard against division by zero
- Only include users with at least 2 orders
- Order by `total_orders DESC`

**Difficulty Rating:** 3/5

WITH users_orders_metrics AS (
SELECT 
user_id,
COUNT(*) AS total_orders,
COUNT(*) FILTER (WHERE amount IS NOT NULL) AS orders_with_amount,
ROUND(AVG(NULLIF(amount, 0))::NUMERIC, 2) AS avg_order_amt
FROM crappy_data_db.orders o
GROUP BY user_id
)
SELECT 
	*,
	round(orders_with_amount::NUMERIC / total_orders, 3) AS conversion_rate,
	CASE WHEN orders_with_amount / total_orders < 1 THEN TRUE ELSE FALSE END AS has_null_amounts
FROM users_orders_metrics
WHERE total_orders >= 2
ORDER BY total_orders DESC


---

## Task 2: Type B Recursive CTE — Full Org Chart with Subordinate Count

**Scenario:**
Use this inline org chart data:

```sql
WITH employees (id, name, manager_id, department) AS (
    VALUES
    (1,  'CEO',        NULL::int, 'Executive'),
    (2,  'CTO',        1,         'Tech'),
    (3,  'CFO',        1,         'Finance'),
    (4,  'VP Eng',     2,         'Tech'),
    (5,  'VP Data',    2,         'Tech'),
    (6,  'FP&A Lead',  3,         'Finance'),
    (7,  'Eng Lead 1', 4,         'Tech'),
    (8,  'Eng Lead 2', 4,         'Tech'),
    (9,  'Data Lead',  5,         'Tech'),
    (10, 'Analyst',    6,         'Finance')
)
```

Traverse the full hierarchy and for each person show their depth, path, and how many direct reports they have.

**Expected Output Columns:**
- `id` (integer)
- `name` (text)
- `depth` (integer) — 1 for CEO
- `path` (text) — e.g. `'CEO -> CTO -> VP Eng -> Eng Lead 1'`
- `direct_reports` (bigint) — count of employees whose manager_id = this person's id

**Requirements:**
- Anchor: `WHERE manager_id IS NULL` — no hardcoding
- Recursive: JOIN employees back to CTE on `employees.manager_id = cte.id`
- `direct_reports`: compute via a subquery or LEFT JOIN to the same employees table
- Path separator: ` -> `
- Order by `path ASC`

**Difficulty Rating:** 4/5


WITH RECURSIVE employees (id, name, manager_id, department) AS (
    VALUES
    (1,  'CEO',        NULL::int, 'Executive'),
    (2,  'CTO',        1,         'Tech'),
    (3,  'CFO',        1,         'Finance'),
    (4,  'VP Eng',     2,         'Tech'),
    (5,  'VP Data',    2,         'Tech'),
    (6,  'FP&A Lead',  3,         'Finance'),
    (7,  'Eng Lead 1', 4,         'Tech'),
    (8,  'Eng Lead 2', 4,         'Tech'),
    (9,  'Data Lead',  5,         'Tech'),
    (10, 'Analyst',    6,         'Finance')
),
HIERARCHY AS (
SELECT 
	*,
	1 AS DEPTH,
	name AS PATH
FROM employees
WHERE manager_id IS NULL
UNION ALL
SELECT
	e.id,
	e.name,
	e.manager_id,
	e.department,
	h.DEPTH + 1,
	h.PATH || '->' || e.name
FROM HIERARCHY h JOIN employees e ON h.id = e.manager_id
),
direct_reports AS (
SELECT 
	manager_id,
	COUNT(*) AS direct_reports
FROM HIERARCHY
GROUP BY manager_id
)
SELECT 
	id,
	name,
	DEPTH,
	PATH,
	COALESCE(direct_reports, 0) AS direct_reports
FROM HIERARCHY h
LEFT JOIN direct_reports dr ON h.id = dr.manager_id

That was an unusual approach and I had to think for a while, but it wasn't too difficult so I figured it out - nice.

---

## Task 3: NULLIF + Window Functions — Transaction Anomaly Detection

**Scenario:**
The fraud team wants to flag transactions where the amount is unusually high relative to that user's typical behavior. Specifically, flag transactions where the amount is more than 2x the user's average — but handle users who have only one transaction (stddev = 0 or NULL) gracefully using NULLIF.

**Expected Output Columns:**
- `id` (integer)
- `user_id` (integer)
- `amount` (numeric)
- `user_avg` (numeric) — user's average transaction amount
- `ratio` (numeric) — `amount / NULLIF(user_avg, 0)`, rounded to 2 decimals
- `is_anomaly` (boolean) — true if ratio > 2.0

**Requirements:**
- Use `transactions` table, exclude NULL amounts and NULL user_ids
- Compute `user_avg` as a window AVG partitioned by user_id
- Use `NULLIF(user_avg, 0)` in the ratio to guard against division by zero
- Order by `ratio DESC NULLS LAST`

**Difficulty Rating:** 4/5

WITH transactions_w_avg AS (
SELECT 
	*,
	ROUND(AVG(amount) OVER (PARTITION BY t.user_id)::NUMERIC, 2) AS user_avg
FROM crappy_data_db.transactions t
WHERE amount IS NOT NULL -- a way simpler method to handle NULL amounts 
AND user_id IS NOT NULL-- NO NEED TO HANDLE NULL user_ids, HONESTLY, but here you are
),
users_transactions_cnt AS (
SELECT 
	user_id,
	COUNT(*) AS transactions_cnt
FROM transactions_w_avg
GROUP BY user_id
)
SELECT 
	tw.id,
	tw.user_id,
	tw.amount,
	tw.user_avg,
	ROUND(tw.amount / tw.user_avg, 2) AS ratio,
	CASE WHEN ROUND(tw.amount / tw.user_avg, 2) > 2.0 THEN TRUE ELSE FALSE END AS is_anomaly
FROM transactions_w_avg tw
JOIN users_transactions_cnt ut ON tw.user_id = ut.user_id AND ut.transactions_cnt > 0

Your instructions ARE CONTRADICTING EACH OTHER and unclear.
I've used a very simple approach to handle everything properly and knowing the data I know this is all correct.

---

## Submission Instructions

1. Task 1 — NULLIF safe division in order metrics (3/5)
2. Task 2 — Type B recursive CTE org chart with direct reports (4/5)
3. Task 3 — NULLIF + window functions for anomaly detection (4/5)

### Task Archive: 2026-04-03 (Week 16, Day 5)
# Daily SQL Practice Tasks

**Generated:** 2026-04-02
**Week 16, Day 4 Focus:** PERCENT_RANK + Complex GROUP BY + FIRST_VALUE with offset + Type A Recursive CTE

---

## Task 1: PERCENT_RANK — User Spending Percentile by Country

**Scenario:**
The growth team wants to understand how each user's total order spend ranks within their country. They need the absolute spend, the percentile rank, and a spending tier label — so they can target marketing campaigns at mid-tier spenders who are close to becoming top performers.

**Expected Output Columns:**
- `user_id` (integer)
- `country` (varchar)
- `total_spent` (double precision) — sum of all order amounts for this user
- `pct_rank` (numeric) — PERCENT_RANK() within country, rounded to 3 decimals
- `spending_tier` (text) — `'top'` if pct_rank >= 0.75, `'mid'` if >= 0.4, `'low'` otherwise

**Requirements:**
- Use `orders` JOIN `users` — only include orders where amount IS NOT NULL and country IS NOT NULL
- Compute total_spent per user, then rank within country
- Only include users with at least 3 orders
- Order by `country ASC`, `pct_rank DESC`

**Difficulty Rating:** 4/5


WITH users_country_spent AS (
SELECT 
	o.user_id,
	u.country,
	SUM(o.amount) AS total_spent
FROM crappy_data_db.orders o 
JOIN crappy_data_db.users u ON o.user_id = u.id
WHERE u.country IS NOT NULL
GROUP BY o.user_id, u.country
),
users_countries_pct_rank AS (
SELECT 
	*, 
	ROUND(PERCENT_RANK() OVER (PARTITION BY country ORDER BY total_spent)::NUMERIC, 3) AS pct_rank
FROM users_country_spent
ORDER BY country
)
SELECT 
	*,
	CASE WHEN pct_rank >= 0.75 THEN 'top' WHEN pct_rank >= 0.4 THEN 'mid' ELSE 'low' END AS spending_tier
FROM users_countries_pct_rank
ORDER BY country, pct_rank DESC


---

## Task 2: Complex GROUP BY — Product Category Revenue with Conditional Aggregation

**Scenario:**
The product team wants a breakdown of each category's revenue performance split by order size. They define "large orders" as amount > 300 and "small orders" as amount <= 300. They want to see how the revenue mix differs across categories and flag categories where large-order revenue exceeds small-order revenue.

**Expected Output Columns:**
- `category_name` (varchar)
- `total_revenue` (numeric) — sum of (price × quantity) across all orders in this category
- `large_order_revenue` (numeric) — revenue from items where the parent order amount > 300
- `small_order_revenue` (numeric) — revenue from items where the parent order amount <= 300
- `large_dominates` (boolean) — true if large_order_revenue > small_order_revenue

**Requirements:**
- Use `product_categories`, `products`, `orders_products`, `orders` — join them properly
- Only include orders where amount IS NOT NULL
- Only include categories with at least 50 total line items (orders_products rows)
- Order by `total_revenue DESC`

**Difficulty Rating:** 4/5

WITH categories_order_revenues AS (
SELECT 
	pc."name" AS category_name,
	SUM(p.price * op.quantity) AS total_revenue,
	SUM(p.price * op.quantity) FILTER (WHERE o.amount > 300) AS large_orders_revenue,
	SUM(p.price * op.quantity) FILTER (WHERE o.amount <= 300) AS small_orders_revenue
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.products p ON op.product_id = p.id
JOIN crappy_data_db.product_categories pc ON pc.id = p.category_id
JOIN crappy_data_db.orders o ON op.order_id = o.id
WHERE o.amount IS NOT NULL
GROUP BY pc."name" 
)
SELECT 
	*,
	large_orders_revenue > small_orders_revenue AS large_dominates
FROM categories_order_revenues


Here, there's no need to exclude any categories, as there are only 3, plus I'm 100% sure every single one had at least 50 line items, for sure!

I could do it with more CTEs using GROUP BY, but I preferred to use pivots, as it's simpler, much clearer and very easy to read - it makes the most sense here.

---

## Task 3: Type A Recursive CTE — Monthly Category Revenue with Running Totals

**Scenario:**
The finance team wants a month-by-month revenue summary per product category, plus a running total that accumulates revenue within each category across months. They also want to know the best-revenue month for each category (the month where revenue was highest).

**Expected Output Columns:**
- `category_name` (varchar)
- `year` (integer)
- `month` (integer)
- `monthly_revenue` (numeric) — sum of price × quantity for that category in that month
- `running_total` (numeric) — cumulative revenue for this category up to and including this month
- `best_month_revenue` (numeric) — highest monthly_revenue ever recorded for this category (same value repeated per category)

**Requirements:**
- Use `product_categories`, `products`, `orders_products`, `orders`
- Only include rows where price IS NOT NULL and amount IS NOT NULL
- Only include categories that appear in at least 3 distinct months of data
- Order by `category_name ASC`, `year ASC`, `month ASC`

**Note:** This task does not require a recursive CTE — solve it purely with window functions. The "Type A" label here refers to the fixed aggregation pattern (monthly aggregation → window over that result), not a recursive hierarchy.

**Difficulty Rating:** 4/5

WITH categories_months AS (
SELECT 
	*,
	pc.name AS category_name,
	DATE_TRUNC('Month', o.created_at) AS month_
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.products p ON op.product_id = p.id
JOIN crappy_data_db.product_categories pc ON pc.id = p.category_id
JOIN crappy_data_db.orders o ON op.order_id = o.id
),
categories_monthly_revenues AS (
SELECT 
	category_name,
	month_,
	SUM(price * quantity) AS monthly_revenue
FROM categories_months
GROUP BY category_name, month_
ORDER BY month_
)
SELECT 
	*,
	sum(monthly_revenue) OVER (PARTITION BY category_name ORDER BY month_) AS running_total,
	MAX(monthly_revenue) OVER (PARTITION BY category_name) AS best_month_revenue
FROM categories_monthly_revenues

Please note that the year column is redundant here, as I've used date_trunc it already contains the year in it and it's properly sorted with ascending dates order - it makes the most sense and we're using 1 column instead of 2, which is way clearer.

---

## Submission Instructions

1. Task 1 — PERCENT_RANK user spending percentile by country (4/5)
2. Task 2 — Complex GROUP BY with conditional aggregation on order size (4/5)
3. Task 3 — Monthly category revenue with running totals and best-month window (4/5)

### Task Archive: 2026-04-07 (Week 17, Day 1)
# Daily SQL Practice Tasks

**Generated:** 2026-04-03
**Week 16, Day 5 Focus:** Time-proximity gaps-and-islands (session detection) + STDDEV anomaly scoring + Anti-join with NULL trap

---

## Task 1: Time-Proximity Gaps-and-Islands — User Session Detection (5/5)

**Scenario:**
The analytics team wants to group a user's daily sessions into "activity bursts" — consecutive days where the user had at least one session, with no gap of more than 2 days between them. Each burst should get a unique burst ID per user, and the team wants to see how long each burst lasted (in days) and the total sessions within it.

**Expected Output Columns:**
- `user_id` (integer)
- `burst_id` (integer) — sequential burst number per user (1, 2, 3…)
- `burst_start` (date) — first date of the burst
- `burst_end` (date) — last date of the burst
- `burst_days` (integer) — number of calendar days from start to end inclusive
- `total_sessions` (integer) — sum of count_sessions across all days in this burst

**Requirements:**
- Use `user_sessions_daily`
- Only include days where `count_sessions > 0`
- A new burst begins when the gap from the previous active day (for that user) exceeds 2 days
- Use LAG to detect gap, SUM OVER to build burst key, then GROUP BY to collapse
- Order by `user_id ASC`, `burst_start ASC`

**Hint:** The pattern is: LAG to get previous active date → compare gap → flag new burst → cumulative SUM of flags = burst group key → GROUP BY that key.

**Difficulty Rating:** 5/5

WITH user_sessions_lag AS (
SELECT 
	*,
	LAG(date) OVER (PARTITION BY user_id ORDER BY date) AS prev_date
FROM crappy_data_db.user_sessions_daily usd
),
streak_is_start AS (
SELECT 
	*,
	date - prev_date,
	CASE WHEN prev_date IS NULL OR date - prev_date <= 2 THEN 0 ELSE 1 END AS is_start
FROM user_sessions_lag
),
sessions_streak_ids AS (
SELECT 
	*,
	SUM(is_start) OVER (PARTITION BY user_id ORDER BY date) AS streak_id
FROM streak_is_start
)
SELECT 
	user_id,
	streak_id,
	MIN(date) AS streak_start,
	MAX(date) AS streak_end,
	COUNT(*) AS streak_days,
	SUM(count_sessions) AS total_sessions
FROM sessions_streak_ids
GROUP BY user_id, streak_id
ORDER BY user_id, streak_start

I changed column names as streak is more natural than burst.

---

## Task 2: STDDEV-Based Outlier Scoring — Transaction Volatility per User

**Scenario:**
The risk team wants to measure how volatile each user's transaction amounts are, and identify which individual transactions are statistical outliers (more than 2 standard deviations from the user's mean). They want a z-score for each transaction so analysts can rank transactions by how unusual they are.

**Expected Output Columns:**
- `id` (integer) — transaction id
- `user_id` (integer)
- `amount` (numeric)
- `user_avg` (numeric) — user's average transaction amount, rounded to 2 decimals
- `user_stddev` (numeric) — user's stddev of transaction amounts, rounded to 2 decimals
- `z_score` (numeric) — `(amount - user_avg) / NULLIF(user_stddev, 0)`, rounded to 2 decimals
- `is_outlier` (boolean) — true if ABS(z_score) > 2.0

**Requirements:**
- Use `transactions`, exclude NULL amounts and NULL user_ids
- Only include users with at least 5 transactions (to make stddev meaningful)
- NULLIF(user_stddev, 0) prevents division by zero for users with identical amounts
- Order by `ABS(z_score) DESC NULLS LAST`

**Difficulty Rating:** 4/5


WITH transactions_avg_std AS (
SELECT 
	*,
	ROUND(AVG(t.amount) OVER (PARTITION BY t.user_id), 2) AS user_avg,
	ROUND(STDDEV(t.amount) OVER (PARTITION BY t.user_id), 2) AS user_std
FROM crappy_data_db.transactions t 
),
transactions_cnt_users AS (
SELECT t.user_id, 
COUNT(*) AS transactions_cnt
FROM crappy_data_db.transactions t
GROUP BY t.user_id
)
SELECT 
	tas.id,
	tas.user_id,
	tas.amount,
	tas.user_avg,
	tas.user_std AS user_stddev,
	ABS(ROUND((tas.amount - tas.user_avg) / NULLIF(tas.user_std, 0), 2)) AS z_score,
	ABS(ROUND((tas.amount - tas.user_avg) / NULLIF(tas.user_std, 0), 2)) > 2.0 AS is_outlier
FROM transactions_avg_std tas
JOIN transactions_cnt_users tcu ON tas.user_id = tcu.user_id AND tcu.transactions_cnt >= 5
ORDER BY z_score DESC NULLS LAST


---

## Task 3: Anti-Join — Users Who Ordered But Never Transacted

**Scenario:**
The finance reconciliation team suspects there are users who placed orders but have no corresponding transactions on record. Find all users who have at least one order but zero transactions — using all three anti-join approaches: NOT IN, NOT EXISTS, and LEFT JOIN ... WHERE IS NULL.

**Expected Output Columns (for each approach):**
- `user_id` (integer)

**Requirements:**
- Use `orders` and `transactions` tables
- Write three separate queries producing the same result:
  1. Using `NOT IN` — then add a comment explaining when this breaks
  2. Using `NOT EXISTS`
  3. Using `LEFT JOIN ... WHERE IS NULL`
- For the NOT IN version: add a SQL comment explaining the NULL trap (what happens if any `user_id` in `transactions` is NULL, and why NOT IN silently returns 0 rows)
- Order by `user_id ASC` in all three

**Difficulty Rating:** 3/5

1. SELECT DISTINCT o.user_id
FROM crappy_data_db.orders o
WHERE o.user_id NOT IN
(SELECT t.user_id 
FROM crappy_data_db.transactions t
WHERE t.user_id IS NOT NULL
)

IT DOES NOT BREAK, AS I USED 'IS NOT NULL' to prevent from breaking :)).


2.

SELECT DISTINCT o.user_id
FROM crappy_data_db.orders o
WHERE NOT EXISTS
(SELECT t.user_id 
FROM crappy_data_db.transactions t
WHERE t.user_id  = o.user_id
)

3. 
SELECT DISTINCT o.user_id
FROM crappy_data_db.orders o
LEFT JOIN crappy_data_db.transactions t ON o.user_id = t.user_id
WHERE t.id IS NULL

I'm not a fan of this pattern though.

Got 35 rows in EVERY single query.



---

## Submission Instructions

1. Task 1 — Time-proximity burst detection with LAG + cumulative SUM (5/5)
2. Task 2 — STDDEV z-score outlier scoring with NULLIF guard (4/5)
3. Task 3 — Anti-join triple: NOT IN / NOT EXISTS / LEFT JOIN IS NULL (3/5)

### Task Archive: 2026-04-08 (Week 17, Day 2)
# Daily SQL Practice Tasks

**Generated:** 2026-04-07
**Week 17, Day 1 Focus:** Light review — JOINs, GROUP BY, HAVING, basic window functions

---

## Task 1: Multi-Table JOIN — Order Revenue by Country and Gender

**Scenario:**
The marketing team wants a simple breakdown of total order revenue and order count, grouped by country and gender. They want to see which country + gender segments are the most valuable.

**Expected Output Columns:**
- `country` (varchar)
- `gender` (varchar)
- `order_count` (integer)
- `total_revenue` (numeric) — sum of order amounts, rounded to 2 decimals

**Requirements:**
- Use `orders` JOIN `users`
- Exclude rows where country IS NULL, gender IS NULL, or amount IS NULL
- Only include segments with at least 10 orders (HAVING)
- Order by `total_revenue DESC`

**Difficulty Rating:** 2/5

SELECT 
	u.country,
	u.gender,
	COUNT(o.id) AS order_count,
	SUM(o.amount) AS total_revenue
FROM crappy_data_db.orders o
JOIN crappy_data_db.users u ON o.user_id = u.id
WHERE u.country IS NOT NULL AND u.gender IS NOT NULL AND o.amount IS NOT NULL
GROUP BY u.country, u.gender
ORDER BY total_revenue DESC

Values were already rounded as intended

---

## Task 2: HAVING with Multiple Conditions — Active High-Value Users

**Scenario:**
The retention team wants to find users who are both frequent and high-spending: at least 5 orders AND average order value above 300. They also want to know the date of their most recent order.

**Expected Output Columns:**
- `user_id` (integer)
- `order_count` (integer)
- `avg_order_value` (numeric) — rounded to 2 decimals
- `total_spent` (numeric) — rounded to 2 decimals
- `last_order_date` (timestamp)

**Requirements:**
- Use `orders` only
- Exclude NULL amounts
- HAVING: order_count >= 5 AND avg_order_value > 300
- Order by `total_spent DESC`

**Difficulty Rating:** 2/5

WITH user_orders_metrics AS (
SELECT 
	user_id,
	COUNT(*) AS order_cnt,
	round(AVG(amount)::NUMERIC, 2) AS avg_order_amt,
	SUM(amount) AS total_spent,
	MAX(created_at) AS last_order_date
FROM crappy_data_db.orders o
WHERE amount IS NOT NULL
GROUP BY user_id
)
SELECT * FROM user_orders_metrics
WHERE order_cnt >= 5 AND avg_order_amt > 300
ORDER BY total_spent DESC

---

## Task 3: Window Function Review — Rank Users by Sessions Within City

**Scenario:**
The engagement team wants to rank users by their total session count, partitioned by city — so they can see who the most active user is within each city.

**Expected Output Columns:**
- `user_id` (integer)
- `city` (varchar)
- `total_sessions` (integer) — sum of count_sessions across all dates
- `city_rank` (integer) — RANK() within city, 1 = most active

**Requirements:**
- Use `user_sessions_daily` JOIN `users`
- Exclude rows where city IS NULL
- Only show users where `city_rank = 1` (top user per city)
- Order by `city ASC`

**Difficulty Rating:** 3/5

WITH users_cities_session_cnts AS (
SELECT 
	usd.user_id,
	u.city,
	SUM(usd.count_sessions) AS total_sessions
FROM crappy_data_db.user_sessions_daily usd
JOIN crappy_data_db.users u ON usd.user_id = u.id
WHERE U.CITY IS NOT null
GROUP BY USD.user_id, u.city
),
sessions_city_ranks AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY city ORDER BY total_sessions DESC) AS city_rank
FROM users_cities_session_cnts
)
SELECT * FROM sessions_city_ranks
WHERE city_rank = 1
ORDER BY city


---

## Submission Instructions

1. Task 1 — JOIN + GROUP BY + HAVING: order revenue by country and gender (2/5)
2. Task 2 — HAVING with multiple conditions: active high-value users (2/5)
3. Task 3 — RANK() partitioned by city, filter to top user per city (3/5)

### Task Archive: 2026-04-09 (Week 17, Day 3)
# Daily SQL Practice Tasks

**Generated:** 2026-04-08
**Week 17, Day 2 Focus:** YoY comparison + LAG offset + Cohort retention (5/5) + NTILE segmentation

---

## Task 1: Year-over-Year Revenue Comparison per Country

**Scenario:**
The finance team wants to compare order revenue between 2024 and 2025 for each country. They want the absolute revenue for each year side by side, the difference, and the percentage change.

**Expected Output Columns:**
- `country` (varchar)
- `revenue_2024` (numeric) — total order revenue in 2024, rounded to 2 decimals
- `revenue_2025` (numeric) — total order revenue in 2025, rounded to 2 decimals
- `revenue_diff` (numeric) — revenue_2025 minus revenue_2024, rounded to 2 decimals
- `pct_change` (numeric) — percentage change from 2024 to 2025, rounded to 1 decimal. Use NULLIF to guard against countries with no 2024 revenue.

**Requirements:**
- Use `orders` JOIN `users`
- Exclude NULL amounts and NULL countries
- Only include countries that have revenue in **both** years
- Order by `pct_change DESC`

**Difficulty Rating:** 3/5

WITH orders_users AS (
SELECT 
	*,
	EXTRACT('Year' FROM o.created_at) AS year_
FROM crappy_data_db.users u
JOIN crappy_data_db.orders o ON u.id = o.user_id
),
orders_countries_revenues AS (
SELECT 
	country,
	SUM(amount) FILTER (WHERE year_ = 2025) AS revenue_2025,
	round(SUM(amount) FILTER (WHERE year_ = 2024)::NUMERIC, 2) AS revenue_2024
FROM orders_users
GROUP BY country
)
SELECT 
	*,
	COALESCE(revenue_2025, 0) - COALESCE(revenue_2024, 0) AS revenue_diff,
	ROUND((COALESCE(revenue_2025::NUMERIC, 0) - COALESCE(revenue_2024, 0)) / revenue_2024::NUMERIC * 100::NUMERIC, 1) AS pct_change
FROM orders_countries_revenues
WHERE revenue_2025 IS NOT NULL AND revenue_2024 IS NOT NULL
ORDER BY pct_change DESC



---

## Task 2: Cohort Retention — Did Users Order in the Month After Registration? (5/5)

**Scenario:**
The growth team wants to measure first-month retention: of all users who registered in a given month, what percentage placed at least one order in the following calendar month?

For example: users who registered in 2024-10 — how many of them placed an order in 2024-11?

**Expected Output Columns:**
- `registration_month` (date) — truncated to month (e.g. 2024-10-01)
- `cohort_size` (integer) — total users registered in that month
- `retained_users` (integer) — users who placed at least one order in the month immediately following their registration month
- `retention_rate` (numeric) — retained_users / cohort_size as a percentage, rounded to 1 decimal

**Requirements:**
- Use `users` and `orders`
- Registration month comes from `users.created_at`
- "Following month" = DATE_TRUNC('month', created_at) + INTERVAL '1 month'
- A user is "retained" if they have at least one order where DATE_TRUNC('month', order.created_at) = their following month
- Only include cohorts with at least 5 users
- Order by `registration_month ASC`

**Difficulty Rating:** 5/5

WITH users_registration_dates AS (
SELECT 
	u.id AS user_id,
	DATE_TRUNC('Month', u.created_at) AS registration_month,
	DATE_TRUNC('Month', u.created_at) + INTERVAL '1 Month' AS next_month
FROM crappy_data_db.users u
WHERE u.created_at IS NOT NULL
),
cohorts AS (
SELECT 
	urd.registration_month,
	COUNT(DISTINCT(urd.user_id)) AS cohort_size,
	COUNT(DISTINCT(urd.user_id)) FILTER (WHERE DATE_TRUNC('Month', o.created_at) = next_month) AS retained_users
FROM users_registration_dates urd
JOIN crappy_data_db.orders o ON urd.user_id = o.user_id
GROUP BY urd.registration_month
)
SELECT 
	*,
	ROUND(retained_users::numeric / cohort_size::NUMERIC * 100, 1) AS retention_rate
FROM cohorts
WHERE cohort_size >= 5


Not that big of a deal for me - manageable task.

---

## Task 3: NTILE Segmentation — Transaction Amount Quartiles

**Scenario:**
The analytics team wants to divide all transactions into 4 equal buckets by amount, then report the min, max, and average amount per bucket — to understand how transaction values are distributed.

**Expected Output Columns:**
- `quartile` (integer) — 1 to 4 (1 = lowest amounts)
- `min_amount` (numeric) — minimum amount in this quartile, rounded to 2 decimals
- `max_amount` (numeric) — maximum amount in this quartile, rounded to 2 decimals
- `avg_amount` (numeric) — average amount in this quartile, rounded to 2 decimals
- `transaction_count` (integer)

**Requirements:**
- Use `transactions`, exclude NULL amounts
- NTILE(4) ordered by amount ASC (so quartile 1 = lowest)
- Order by `quartile ASC`

**Difficulty Rating:** 3/5

WITH transactions_quartiles AS (
SELECT 
	*,
	NTILE(4) OVER (ORDER BY t.amount) AS quartile
FROM crappy_data_db.transactions t
)
SELECT 
	quartile,
	ROUND(MIN(amount), 2) AS min_amount,
	ROUND(MAX(amount), 2) AS max_amount,
	ROUND(AVG(amount), 2) AS avg_amount,
	COUNT(*) AS transaction_count
FROM transactions_quartiles
GROUP BY quartile
ORDER BY quartile

This is just too easy.

---

## Submission Instructions

1. Task 1 — YoY revenue comparison per country with NULLIF pct_change guard (3/5)
2. Task 2 — Cohort retention: % of users who ordered in month after registration (5/5)
3. Task 3 — NTILE(4) transaction quartiles with min/max/avg per bucket (3/5)

### Task Archive: 2026-04-10 (Week 17, Day 4)
# Daily SQL Practice Tasks

**Generated:** 2026-04-09
**Week 17, Day 3 Focus:** Easy-medium session — LAG for change detection + self-join affinity + window running total

---

## Task 1: LAG — Detect Users Whose Order Value Dropped

**Scenario:**
The retention team wants to find users whose most recent order was lower in value than their previous order — a potential sign of disengagement.

**Expected Output Columns:**
- `user_id` (integer)
- `prev_amount` (numeric) — amount of the second-to-last order, rounded to 2 decimals
- `last_amount` (numeric) — amount of the most recent order, rounded to 2 decimals
- `drop` (numeric) — prev_amount minus last_amount, rounded to 2 decimals

**Requirements:**
- Use `orders`, exclude NULL amounts
- Use LAG to get the previous order amount per user (ordered by created_at ASC)
- Only return users where the last order amount is lower than the previous one
- Only include users with at least 2 orders
- Order by `drop DESC`

**Difficulty Rating:** 3/5

WITH users_orders AS (
SELECT 
	*,
	LAG(o.amount) OVER (PARTITION BY user_id ORDER BY created_at DESC) AS prev_amount,
	rank() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
FROM crappy_data_db.orders o
ORDER BY user_id, created_at DESC
)
SELECT 
	user_id,
	prev_amount,
	amount AS last_amount,
	prev_amount - amount AS drop
FROM users_orders
WHERE rn = 2
ORDER BY DROP desc

I used slightly different, but clearer logic.


---

## Task 2: Self-Join — Products Frequently Bought Together

**Scenario:**
The recommendations team wants to find pairs of products that appear together in the same order at least 3 times — to power a "frequently bought together" feature.

**Expected Output Columns:**
- `product_a` (integer) — lower product_id of the pair
- `product_b` (integer) — higher product_id of the pair
- `times_bought_together` (integer)

**Requirements:**
- Use `orders_products`
- Self-join on `order_id`, with `op1.product_id < op2.product_id` to avoid duplicates
- Only include pairs appearing together in at least 3 distinct orders
- Order by `times_bought_together DESC`

**Difficulty Rating:** 3/5

SELECT
	op1.product_id AS product_a,
	op2.product_id AS product_b,
	COUNT(*) AS times_bought_together
FROM crappy_data_db.orders_products op1
JOIN crappy_data_db.orders_products op2 ON op1.order_id = op2.order_id
WHERE op1.product_id > op2.product_id
GROUP BY op1.product_id, op2.product_id
HAVING COUNT(*) >= 3
ORDER BY times_bought_together DESC


---

## Task 3: Running Total — Cumulative Revenue per User Over Time

**Scenario:**
The finance team wants to see how each user's cumulative order spend grows over time — one row per order, with the running total up to and including that order.

**Expected Output Columns:**
- `user_id` (integer)
- `order_id` (integer)
- `created_at` (timestamp)
- `amount` (numeric)
- `cumulative_spent` (numeric) — running total of amount for this user up to this order, rounded to 2 decimals

**Requirements:**
- Use `orders`, exclude NULL amounts
- SUM OVER partitioned by user_id, ordered by created_at ASC
- Only include users with at least 3 orders
- Order by `user_id ASC`, `created_at ASC`

**Difficulty Rating:** 3/5

SELECT 
	o.user_id,
	o.id AS order_id,
	o.created_at,
	o.amount,
	SUM(o.amount) OVER (PARTITION BY o.user_id ORDER BY o.created_at) AS cumulative_spent
FROM crappy_data_db.orders o


Very easy.


---

## Submission Instructions

1. Task 1 — LAG to detect order value drop per user (3/5)
2. Task 2 — Self-join: products bought together at least 3 times (3/5)
3. Task 3 — Running cumulative spend per user with SUM OVER (3/5)

### Task Archive: 2026-04-13 (Week 18, Day 1)
# Daily SQL Practice Tasks

**Generated:** 2026-04-10
**Week 17, Day 4 Focus:** Type A recursive CTE (3-level) + FIRST_VALUE frame spec + complex multi-window scoring (5/5)

---

## Task 1: Type A Recursive CTE — 3-Level Product Category Revenue Rollup

**Scenario:**
The finance team wants a 3-level revenue rollup: individual product → product category → grand total. Each level should show the label, the revenue at that level, and the percentage of the grand total.

**Expected Output Columns:**
- `level` (integer) — 1 = product, 2 = category, 3 = grand total
- `label` (varchar) — product name, category name, or 'Grand Total'
- `revenue` (numeric) — sum of price × quantity for this node, rounded to 2 decimals
- `pct_of_total` (numeric) — revenue as % of grand total, rounded to 1 decimal

**Requirements:**
- Use `products`, `product_categories`, `orders_products`
- Only include rows where price IS NOT NULL
- Structure as three pre-aggregated CTEs (product_revenue, category_revenue, grand_total), then UNION ALL them together with a level column
- Order by `level ASC`, `revenue DESC`

**Difficulty Rating:** 4/5

WITH product_revenues AS (
SELECT 
	p.id AS product_id,
	p.category_id AS category_id,
	p.name AS LABEL,
	SUM(p.price * op.quantity) AS REVENUE
FROM crappy_data_db.products p
JOIN crappy_data_db.orders_products op ON op.product_id = p.id
GROUP BY p.id, p.name
),
categories_revenues AS (
SELECT 
	pc.id AS category_id,
	pc.name AS category_name,
	SUM(op.quantity * p.price) AS category_revenue
FROM crappy_data_db.products p
JOIN crappy_data_db.orders_products op ON op.product_id = p.id
JOIN crappy_data_db.product_categories pc ON p.category_id = pc.id
GROUP BY pc.id, pc.name
),
total_revenue AS (
SELECT 
	SUM(p.price * op.quantity) AS grand_total
FROM crappy_data_db.products p
JOIN crappy_data_db.orders_products op ON op.product_id = p.id
)
SELECT 
	1 AS LEVEL,
	pr.LABEL AS LABEL,
	pr.revenue AS revenue,
	ROUND(revenue / tr.grand_total * 100, 2) AS pct_of_total
FROM product_revenues pr
JOIN total_revenue tr ON tr.grand_total > pr.revenue
UNION ALL
SELECT 
	2 AS level,
	cr.category_name,
	cr.category_revenue,
	ROUND(cr.category_revenue / tr.grand_total * 100, 2)
FROM categories_revenues cr
JOIN total_revenue tr ON tr.grand_total > cr.category_revenue
UNION ALL
SELECT 
	3 AS LEVEL,
	'grand_total',
	tr.grand_total,
	ROUND(tr.grand_total / tr.grand_total * 100, 2)
FROM total_revenue tr

This was quite weird, but I've done it.


---

## Task 2: FIRST_VALUE — Each User's First and Most Recent Transaction Type

**Scenario:**
The product team wants to know, for each user, what transaction type they started with and what their most recent transaction type was — to detect whether users' behaviour has shifted over time.

**Expected Output Columns:**
- `user_id` (integer)
- `first_type` (text) — transaction type of the user's earliest transaction
- `last_type` (text) — transaction type of the user's most recent transaction
- `total_transactions` (integer)
- `shifted` (boolean) — true if first_type != last_type

**Requirements:**
- Use `transactions`, exclude NULL types and NULL user_ids
- Use `FIRST_VALUE(type ORDER BY created_at ASC)` for first_type
- Use `FIRST_VALUE(type ORDER BY created_at DESC)` for last_type
- Collapse to one row per user — use a CTE with window functions, then GROUP BY or DISTINCT
- Only include users with at least 3 transactions
- Order by `user_id ASC`

**Difficulty Rating:** 3/5

WITH users_transactions AS (
SELECT 
	*,
	FIRST_VALUE(type) OVER (PARTITION BY user_id ORDER BY t.created_at) first_transaction_type,
	FIRST_VALUE(type) OVER (PARTITION BY user_id ORDER BY t.created_at DESC) AS last_transaction_type
FROM crappy_data_db.transactions t
)
SELECT 
	user_id,
	first_transaction_type,
	last_transaction_type,
	COUNT(*) AS transactions,
	first_transaction_type = last_transaction_type AS shifted
FROM users_transactions
GROUP BY user_id, 	first_transaction_type, last_transaction_type
HAVING COUNT(*) >= 3
ORDER BY user_id


---

## Task 3: Multi-Window Scoring — User Engagement Score (5/5)

**Scenario:**
The growth team wants a composite engagement score for each user, combining three signals:
1. **Order frequency score**: NTILE(4) on total order count — quartile 4 = most orders
2. **Spend score**: NTILE(4) on total order amount — quartile 4 = highest spend
3. **Session score**: NTILE(4) on total session count — quartile 4 = most sessions

Final `engagement_score` = sum of the three NTILE values (max 12, min 3).

They then want to rank users by engagement_score and flag the top 10% using PERCENT_RANK.

**Expected Output Columns:**
- `user_id` (integer)
- `order_freq_score` (integer) — NTILE(4) on order count
- `spend_score` (integer) — NTILE(4) on total spend
- `session_score` (integer) — NTILE(4) on session count
- `engagement_score` (integer) — sum of the three scores
- `engagement_pct_rank` (numeric) — PERCENT_RANK() on engagement_score, rounded to 3 decimals
- `is_top_10pct` (boolean) — true if engagement_pct_rank >= 0.90

**Requirements:**
- Use `orders`, `user_sessions_daily`, and `users` (to get the full user list as base)
- A user with no orders gets order_freq_score = 1, spend_score = 1 (treat missing as lowest tier)
- A user with no sessions gets session_score = 1
- NTILE must be computed over all users (not just those with orders/sessions)
- Only include users where is_active = TRUE
- Order by `engagement_score DESC`, `user_id ASC`

**Difficulty Rating:** 5/5

WITH orders_metrics AS (
SELECT 
	user_id,
	SUM(o.amount) AS total_spent,
	COUNT(*) AS order_cnt
FROM crappy_data_db.orders o
JOIN crappy_data_db.users u ON o.user_id = u.id
WHERE u.is_active = True
GROUP BY o.user_id
),
sessions_totals AS (
SELECT 
	user_id,
	SUM(count_sessions) AS total_sessions
FROM crappy_data_db.user_sessions_daily usd
GROUP BY user_id
),
users_combined_metrics AS (
SELECT 
	om.user_id,
	NTILE(4) OVER (ORDER BY om.order_cnt) AS order_freq_score,
	NTILE(4) OVER (ORDER BY om.total_spent) AS spend_score,
	NTILE(4) OVER (ORDER BY st.total_sessions) AS session_score
FROM orders_metrics om
JOIN sessions_totals st ON om.user_id = st.user_id
),
users_engagement_score AS (
SELECT 
	*,
	order_freq_score + spend_score + session_score AS engagement_score,
	ROUND(percent_rank() OVER (ORDER BY (order_freq_score + spend_score + session_score))::NUMERIC, 3) AS engagement_pct_rank
FROM users_combined_metrics
)
SELECT 
	*,
	engagement_pct_rank >= 0.9 AS is_top_10pct
FROM users_engagement_score
ORDER BY engagement_score DESC

---

## Submission Instructions

1. Task 1 — Type A 3-level revenue rollup with UNION ALL (4/5)
2. Task 2 — FIRST_VALUE first and last transaction type per user (3/5)
3. Task 3 — Composite engagement score from three NTILE signals + PERCENT_RANK (5/5)

### Task Archive: 2026-04-14 (Week 18, Day 2)
# Daily SQL Practice Tasks

**Generated:** 2026-04-13
**Week 18, Day 1 Focus:** LEFT JOIN + COALESCE base population + LEAD

---

## Task 1: LEFT JOIN + COALESCE — All Active Users with Order Metrics

**Scenario:**
The growth team wants a full list of all users with their order stats — but users who have never placed an order must still appear in the result with zeros, not be silently dropped.

**Expected Output Columns:**
- `user_id` (integer)
- `country` (varchar)
- `order_count` (integer) — 0 if no orders
- `total_spent` (numeric) — 0.00 if no orders
- `avg_order_value` (numeric) — NULL if no orders (can't average zero orders)

**Requirements:**
- Base: all users from `users` table
- LEFT JOIN aggregated order metrics onto the user base
- Use COALESCE to convert NULL order_count and total_spent to 0
- avg_order_value should remain NULL for users with no orders — do not COALESCE it to 0
- Exclude NULL countries
- Order by `total_spent DESC`

**Difficulty Rating:** 3/5

SELECT 
	u.id AS user_id,
	u.country,
	COALESCE(COUNT(o.id), 0) AS order_count,
	ROUND(COALESCE(SUM(o.amount), 0)::NUMERIC, 2) AS total_spent,
	ROUND(AVG(o.amount)::NUMERIC, 2) AS avg_order_value
FROM crappy_data_db.users u
LEFT JOIN crappy_data_db.orders o ON u.id = o.user_id
WHERE u.country IS NOT NULL
GROUP BY u.id, u.country
ORDER BY total_spent DESC




---

## Task 2: LEAD — Days Until Next Order per User

**Scenario:**
The retention team wants to understand ordering cadence: for each order, how many days until the same user placed their next order? Orders with no subsequent order should show NULL.

**Expected Output Columns:**
- `user_id` (integer)
- `order_id` (integer)
- `order_date` (date) — DATE(created_at)
- `next_order_date` (date) — date of the user's next order, NULL if none
- `days_until_next` (integer) — next_order_date minus order_date in days, NULL if no next order

**Requirements:**
- Use `orders`, exclude NULL amounts
- Use LEAD(DATE(created_at)) OVER (PARTITION BY user_id ORDER BY created_at ASC)
- Only include users with at least 2 orders
- Order by `user_id ASC`, `order_date ASC`

**Difficulty Rating:** 3/5

WITH orders_dates AS (
SELECT
	*,
	DATE(created_at) AS order_date,
	LEAD(DATE(created_at)) OVER (PARTITION BY user_id ORDER BY created_at) AS next_order_date
FROM crappy_data_db.orders o
),
orders_cnt AS (
SELECT 
user_id,
COUNT(*) AS order_cnt
FROM crappy_data_db.orders o
GROUP BY user_id
)
SELECT 
	od.user_id,
	od.id AS order_id,
	od.order_date,
	od.next_order_date,
	oc.order_cnt,
	CASE WHEN od.next_order_date IS NULL THEN NULL ELSE od.next_order_date - od.order_date END AS days_until_next 
FROM orders_dates od
JOIN orders_cnt oc ON od.user_id = oc.user_id AND oc.order_cnt >= 2
ORDER BY od.user_id, od.order_date


I've added order_cnt to make sure data prints properly. It's all good.

---

## Task 3: LEFT JOIN Chain — Active Users with Orders and Transactions (5/5)

**Scenario:**
The finance team wants to understand engagement depth: for every active user, show their order count, transaction count, and total transaction amount — even if they have no orders or no transactions. Then flag users who have orders but zero transactions (potential payment issues).

**Expected Output Columns:**
- `user_id` (integer)
- `order_count` (integer) — 0 if none
- `transaction_count` (integer) — 0 if none
- `total_transaction_amount` (numeric) — 0.00 if none
- `has_orders_no_transactions` (boolean) — true if order_count > 0 AND transaction_count = 0

**Requirements:**
- Base: all users from `users` table
- LEFT JOIN order metrics (pre-aggregated) onto users
- LEFT JOIN transaction metrics (pre-aggregated) onto users
- Both joins must be LEFT — no active user should be dropped
- COALESCE all counts and amounts to 0
- Order by `has_orders_no_transactions DESC`, `order_count DESC`

**Difficulty Rating:** 5/5

That's marked as 5/5 difficulty, but that was very easy for me.

WITH users_metrics AS (
SELECT 
	u.id AS user_id,
	COALESCE(COUNT(o.id), 0) AS order_count,
	COALESCE(COUNT(t.id), 0) AS transaction_count,
	COALESCE(SUM(t.amount), 0.00) AS total_transaction_amount
FROM crappy_data_db.users u
LEFT JOIN crappy_data_db.orders o ON u.id = o.user_id
LEFT JOIN crappy_data_db.transactions t ON u.id = t.user_id
GROUP BY u.id
)
SELECT 
	*,
	(order_count > 0) AND (transaction_count = 0) AS has_orders_no_transactions
FROM users_metrics 
ORDER BY has_orders_no_transactions DESC, order_count DESC




---

## Submission Instructions

1. Task 1 — LEFT JOIN + COALESCE: all active users with order metrics, zeros for no orders (3/5)
2. Task 2 — LEAD: days until next order per user (3/5)
3. Task 3 — LEFT JOIN chain: active users with orders + transactions, flag payment gap (5/5)

### Task Archive: 2026-04-15 (Week 18, Day 3)
# Daily SQL Practice Tasks

**Generated:** 2026-04-14
**Week 18, Day 2 Focus:** Light session — window functions review + basic aggregation

---

## Task 1: Top Product per Category by Revenue

**Scenario:**
The product team wants to know the single best-selling product (by revenue) within each category.

**Expected Output Columns:**
- `category_name` (varchar)
- `product_name` (varchar)
- `revenue` (numeric) — sum of price × quantity, rounded to 2 decimals

**Requirements:**
- Use `product_categories`, `products`, `orders_products`
- Only include rows where price IS NOT NULL
- Use RANK() or ROW_NUMBER() partitioned by category, then filter to rank = 1
- Order by `revenue DESC`

**Difficulty Rating:** 2/5

WITH products_sales AS (
SELECT 
	pc."name" AS category_name,
	p.name AS product_name,
	SUM(p.price * op.quantity) AS revenue
FROM crappy_data_db.orders_products op
JOIN crappy_data_db.products p ON p.id = op.product_id
JOIN crappy_data_db.product_categories pc ON p.category_id = pc.id
WHERE price IS NOT NULL
GROUP BY pc.name, p.name
),
categories_rank AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY category_name ORDER BY revenue DESC) AS sales_rank
FROM products_sales
)
SELECT * FROM categories_rank WHERE sales_rank = 1
ORDER BY revenue DESC



---

## Task 2: Transaction Count and Amount by Type and Month

**Scenario:**
The finance team wants a monthly breakdown of transaction volume and total amount per transaction type.

**Expected Output Columns:**
- `year` (integer)
- `month` (integer)
- `type` (text)
- `transaction_count` (integer)
- `total_amount` (numeric) — rounded to 2 decimals

**Requirements:**
- Use `transactions`, exclude NULL types and NULL amounts
- Extract year and month from created_at
- Order by `year ASC`, `month ASC`, `type ASC`

**Difficulty Rating:** 2/5


WITH transactions_y_m AS (
SELECT 
	*,
	date_trunc('Year', created_at) AS YEAR,
	date_trunc('Month', created_at) AS month
FROM crappy_data_db.transactions t
)
SELECT 
	YEAR,
	MONTH,
	TYPE,
	COUNT(*) AS transaction_count,
	SUM(amount) AS total_amount
FROM transactions_y_m
GROUP BY YEAR, MONTH, TYPE
ORDER BY YEAR, month, type


---

## Task 3: Users with Above-Average Order Spend

**Scenario:**
Find users whose total order spend is above the overall average total spend across all users.

**Expected Output Columns:**
- `user_id` (integer)
- `total_spent` (numeric) — rounded to 2 decimals
- `overall_avg` (numeric) — the overall average, same value on every row, rounded to 2 decimals

**Requirements:**
- Use `orders`, exclude NULL amounts
- Compute total_spent per user, then compare against the average of those totals
- Include `overall_avg` as a window or scalar subquery so it's visible in output
- Order by `total_spent DESC`

**Difficulty Rating:** 3/5

WITH users_spents AS (
SELECT 
	user_id,
	ROUND(SUM(o.amount)::numeric, 2) AS total_spent
FROM crappy_data_db.orders o
GROUP BY user_id
)
SELECT 
	user_id,
	total_spent,
	ROUND((SELECT AVG(total_spent) FROM users_spents), 2) AS overall_avg
FROM users_spents
ORDER BY total_spent DESC



---

## Submission Instructions

1. Task 1 — Top product per category by revenue using RANK (2/5)
2. Task 2 — Monthly transaction breakdown by type (2/5)
3. Task 3 — Users with above-average total spend (3/5)

### Task Archive: 2026-04-16 (Week 18, Day 4)
# Daily SQL Practice Tasks

**Generated:** 2026-04-15
**Week 18, Day 3 Focus:** Funnel analysis (5/5) + gaps-and-islands monthly streaks + PERCENT_RANK with conditional bucket

---

## Task 1: Funnel Analysis — Registration → Order → Transaction (5/5)

**Scenario:**
The growth team wants to measure how many users complete each stage of the engagement funnel:
- **Stage 1**: Registered (all users)
- **Stage 2**: Placed at least one order
- **Stage 3**: Made at least one transaction

They want the absolute count at each stage and the drop-off rate from the previous stage.

**Expected Output Columns:**
- `stage` (integer) — 1, 2, or 3
- `stage_name` (text) — 'Registered', 'Ordered', 'Transacted'
- `user_count` (integer) — number of users reaching this stage
- `dropoff_pct` (numeric) — % of users lost vs previous stage, rounded to 1 decimal. Stage 1 = 0.0.

**Requirements:**
- Base population: all users from `users`
- Stage 2: users who have at least one row in `orders`
- Stage 3: users who have at least one row in `transactions`
- Use LEFT JOIN + COUNT DISTINCT to build each stage count — do NOT use subqueries in WHERE
- `dropoff_pct` = (prev_stage_count - current_stage_count) / prev_stage_count * 100
- Use LAG to compute dropoff from the previous stage row
- Final result: 3 rows, one per stage
- Order by `stage ASC`

**Difficulty Rating:** 5/5


WITH RECURSIVE registered_count AS (
SELECT
	'Registered' AS stage_name,
	COUNT(u.id) AS user_count
FROM crappy_data_db.users u
),
ordered_count AS (
SELECT 
	'Ordered' AS stage_name,
	COUNT(DISTINCT(o.user_id)) AS user_count
FROM crappy_data_db.orders o
),
transacted_count AS (
SELECT 
	'Transacted' AS stage_name,
	COUNT(DISTINCT(t.user_id)) AS user_count
FROM crappy_data_db.transactions t
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	rc.stage_name,
	rc.user_count,
	0.0 AS dropoff_pct
FROM registered_count rc
UNION ALL
SELECT
	h.LEVEL + 1,
	COALESCE(oc.stage_name, tc.stage_name),
	COALESCE(oc.user_count, tc.user_count),
	COALESCE(ROUND((1 - oc.user_count / h.user_count::NUMERIC) * 100, 1), ROUND((1 - Tc.user_count / h.user_count::NUMERIC) * 100, 1))
FROM HIERARCHY h
LEFT JOIN ordered_count oc ON h.LEVEL = 1
LEFT JOIN transacted_count tc ON h.LEVEL = 2
WHERE H.LEVEL < 3
)
SELECT * FROM hierarchy


I used recursive cte instead, a pattern we've practiced before. Imo it's very effective here.



---

## Task 2: Gaps-and-Islands — Monthly Order Streaks per User

**Scenario:**
The retention team wants to find users who placed orders in consecutive calendar months — and measure how long their longest ordering streak was (in months).

**Expected Output Columns:**
- `user_id` (integer)
- `streak_start` (date) — first month of the streak (DATE_TRUNC to month)
- `streak_end` (date) — last month of the streak
- `streak_length` (integer) — number of consecutive months in the streak

**Requirements:**
- Use `orders`, exclude NULL amounts
- Collapse to one row per user per month first (a user with 3 orders in the same month counts as 1 active month)
- Use the ROW_NUMBER subtraction trick to identify streak groups
- Only return each user's **longest** streak (if tied, return the earliest)
- Only include users with a streak of at least 2 months
- Order by `streak_length DESC`, `user_id ASC`

**Difficulty Rating:** 4/5


WITH orders_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month
FROM crappy_data_db.orders o
),
orders_prev_months AS (
SELECT 
	*,
	LAG(month) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_order_month
FROM orders_months
),
orders_new_streak_markers AS (
SELECT 
	*,
	CASE WHEN prev_order_month IS NULL OR prev_order_month - MONTH > INTERVAL '1' Month THEN 1 ELSE 0 END AS is_new_streak
FROM orders_prev_months
),
orders_streak_keys AS (
SELECT
	*,
	SUM(is_new_streak) OVER (PARTITION BY user_id ORDER BY created_at) AS streak_key
FROM orders_new_streak_markers 
),
users_streaks AS (
SELECT 
	user_id,
	MIN(month) AS streak_start,
	MAX(month) AS streak_end,
	COUNT(DISTINCT(month)) AS streak_length
FROM orders_streak_keys
GROUP BY user_id, streak_key
HAVING COUNT(DISTINCT(month)) >= 2
ORDER BY streak_length DESC, user_id
)
SELECT * FROM users_streaks

I used different pattern to identify patterns properly, IMO it's more universal and works in more contexts, but prove me wrong. I didn't read your instructions as I wanted to think myself, but I followed your objective and requirements, so you should credit me properly for that. Imo it's a better solution as well.

---

## Task 3: PERCENT_RANK — Transaction Amount Buckets with Percentile Labels

**Scenario:**
The analytics team wants to label each transaction with a human-readable percentile band based on amount — so analysts can quickly see if a transaction is in the bottom, middle, or top of the distribution.

**Expected Output Columns:**
- `id` (integer)
- `user_id` (integer)
- `amount` (numeric)
- `pct_rank` (numeric) — PERCENT_RANK() on amount, rounded to 3 decimals
- `band` (text) — 'bottom 25%' if pct_rank < 0.25, 'middle 50%' if < 0.75, 'top 25%' otherwise

**Requirements:**
- Use `transactions`, exclude NULL amounts and NULL user_ids
- PERCENT_RANK ordered by amount ASC (global, no partition)
- Order by `amount DESC`

**Difficulty Rating:** 3/5

WITH transactions_pct_rank AS (
SELECT 
	*,
	round(percent_rank() OVER (ORDER BY amount)::NUMERIC, 3) AS pct_rank
FROM crappy_data_db.transactions t 
WHERE t.amount IS NOT NULL AND t.user_id IS NOT NULL
)
SELECT 
	id,
	user_id,
	amount,
	pct_rank,
	CASE 
	WHEN pct_rank < 0.25 THEN 'bottom 25%' 
	WHEN pct_rank < 0.75 THEN 'middle 50%' ELSE 'top 25%' 
	END AS band
FROM transactions_pct_rank
ORDER BY amount DESC


---

## Submission Instructions

1. Task 1 — Funnel analysis: registered → ordered → transacted with dropoff % (5/5)
2. Task 2 — Monthly order streaks per user, longest streak only (4/5)
3. Task 3 — PERCENT_RANK transaction amount bands (3/5)

### Task Archive: 2026-04-17 (Week 18, Day 5)
# Daily SQL Practice Tasks

**Generated:** 2026-04-16
**Week 18, Day 4 Focus:** Type B recursive CTE + LAG with offset + complex multi-condition aggregation (5/5)

---

## Task 1: Type B Recursive CTE — Full Organisational Hierarchy

**Scenario:**
Given the employee dataset below, traverse the full org chart from the CEO down to every leaf node. Show each employee's depth in the hierarchy, their full reporting path, and how many people report directly to them.

```sql
-- Use this inline dataset (paste into your query as a CTE):
-- id | name          | manager_id | department
--  1 | CEO           | NULL       | Executive
--  2 | CTO           | 1          | Tech
--  3 | CFO           | 1          | Finance
--  4 | VP Eng        | 2          | Tech
--  5 | VP Data       | 2          | Tech
--  6 | FP&A Lead     | 3          | Finance
--  7 | Eng Lead 1    | 4          | Tech
--  8 | Eng Lead 2    | 4          | Tech
--  9 | Data Lead     | 5          | Tech
-- 10 | Analyst       | 6          | Finance
-- 11 | Engineer 1    | 7          | Tech
-- 12 | Engineer 2    | 8          | Tech
```

**Expected Output Columns:**
- `id` (integer)
- `name` (text)
- `depth` (integer) — 1 = CEO, increases with each level
- `path` (text) — e.g. `CEO -> CTO -> VP Eng`
- `direct_reports` (integer) — number of people who directly report to this person (0 for leaf nodes)

**Requirements:**
- Anchor: `WHERE manager_id IS NULL` — never hardcode the root node
- Recursive join: employees on `manager_id = cte.id`
- Compute `direct_reports` in a separate CTE after the recursion, then LEFT JOIN back
- Order by `depth ASC`, `id ASC`

**Difficulty Rating:** 4/5

WITH RECURSIVE employees (id, name, manager_id, department) AS (
    VALUES
    (1,  'CEO',        NULL, 'Executive'),
    (2,  'CTO',        1,    'Tech'),
    (3,  'CFO',        1,    'Finance'),
    (4,  'VP Eng',     2,    'Tech'),
    (5,  'VP Data',    2,    'Tech'),
    (6,  'FP&A Lead',  3,    'Finance'),
    (7,  'Eng Lead 1', 4,    'Tech'),
    (8,  'Eng Lead 2', 4,    'Tech'),
    (9,  'Data Lead',  5,    'Tech'),
    (10, 'Analyst',    6,    'Finance'),
    (11, 'Engineer 1', 7,    'Tech'),
    (12, 'Engineer 2', 8,    'Tech')
),
hierarchy AS (
SELECT
	id,
	name,
	1 AS DEPTH,
	manager_id,
	name AS PATH,
	department
FROM employees WHERE manager_id IS NULL
UNION ALL
SELECT 
	e.id,
	e.name,
	h.DEPTH + 1,
	e.manager_id,
	h.PATH || '->' || e.name,
	e.department
FROM employees e 
JOIN HIERARCHY H ON e.manager_id = h.id
),
direct_report_cte AS (
SELECT
	manager_id,
	COUNT(*) AS direct_reports
FROM HIERARCHY
GROUP BY manager_id
)
SELECT 
	h.id,
	h.name,
	h.DEPTH,
	h.PATH,
	COALESCE(dr.direct_reports, 0)
FROM HIERARCHY h
LEFT JOIN direct_report_cte dr ON h.id = dr.manager_id

It's already ordered in proper order.


---

## Task 2: LAG with Offset — Compare Orders to 3 Orders Ago

**Scenario:**
The analytics team wants to understand long-term order value trends per user. For each order, show the amount from 3 orders ago for the same user, and the difference — to detect gradual spend changes over time.

**Expected Output Columns:**
- `user_id` (integer)
- `order_id` (integer)
- `created_at` (timestamp)
- `amount` (numeric)
- `amount_3_orders_ago` (numeric) — amount from 3 orders prior for this user, NULL if fewer than 3 prior orders
- `diff` (numeric) — amount minus amount_3_orders_ago, NULL if no comparison available

**Requirements:**
- Use `orders`, exclude NULL amounts
- Use `LAG(amount, 3)` partitioned by user_id, ordered by created_at ASC
- Only include users with at least 4 orders
- Order by `user_id ASC`, `created_at ASC`

**Difficulty Rating:** 3/5


WITH users_order_cnt AS (
SELECT
	o.user_id,
	COUNT(*) AS order_cnt
FROM crappy_data_db.orders o
GROUP BY o.user_id
),
users_orders AS (
SELECT 
	o.user_id,
	o.id AS order_id,
	o.created_at,
	o.amount,
	LAG(o.amount, 3) OVER (PARTITION BY o.user_id ORDER BY o.created_at) AS amount_3_orders_ago
FROM crappy_data_db.orders o
JOIN users_order_cnt oc ON o.user_id = oc.user_id AND oc.order_cnt >= 4
)
SELECT 
	*,
	amount - amount_3_orders_ago AS diff
FROM users_orders

---

## Task 3: Complex Aggregation — User Engagement Tier Report (5/5)

**Scenario:**
The product team wants a single summary report combining order behaviour, session behaviour, and ticket behaviour per user — with tier labels based on composite thresholds.

**Expected Output Columns:**
- `user_id` (integer)
- `order_count` (integer)
- `total_sessions` (integer)
- `ticket_count` (integer)
- `engagement_tier` (text):
  - `'power'` — order_count >= 10 AND total_sessions >= 50
  - `'active'` — order_count >= 5 OR total_sessions >= 30
  - `'casual'` — everyone else

**Requirements:**
- Base: all users from `users`
- LEFT JOIN pre-aggregated order metrics (from `orders`)
- LEFT JOIN pre-aggregated session metrics (from `user_sessions_daily`)
- LEFT JOIN pre-aggregated ticket metrics (from `chat_tickets`)
- COALESCE all counts to 0
- Tier logic: evaluate `'power'` first, then `'active'`, then `'casual'`
- Order by `engagement_tier ASC`, `order_count DESC`

**Difficulty Rating:** 5/5

WITH users_order_cnt AS (
SELECT 
	o.user_id,
	COUNT(o.id) AS order_count
FROM crappy_data_db.orders o
GROUP BY o.user_id
),
users_total_sessions AS (
SELECT 
	usd.user_id,
	SUM(usd.count_sessions) AS total_sessions
FROM crappy_data_db.user_sessions_daily usd
GROUP BY usd.user_id
),
users_ticket_cnt AS (
SELECT 
	user_id,
	COUNT(id) AS ticket_cnt
FROM crappy_data_db.chat_tickets ct
GROUP BY user_id
),
users_metrics AS (
SELECT 
	u.id AS user_id,
	COALESCE(uoc.order_count, 0) AS order_count,
	COALESCE(usc.total_sessions, 0) AS total_sessions,
	COALESCE(utc.ticket_cnt, 0) AS ticket_count
FROM crappy_data_db.users u
LEFT JOIN users_order_cnt uoc ON u.id = uoc.user_id
LEFT JOIN users_total_sessions usc ON u.id = usc.user_id
LEFT JOIN users_ticket_cnt utc ON u.id = utc.user_id
)
SELECT 
	*,
CASE 
WHEN order_count >= 10 AND total_sessions >= 50 THEN 'power'
WHEN order_count >= 5 AND total_sessions >= 30 THEN 'active' ELSE 'casual' END AS engagement_tier
FROM users_metrics
ORDER BY engagement_tier, order_count DESC


---

## Submission Instructions

1. Task 1 — Type B recursive CTE: full org chart with depth, path, direct reports (4/5)
2. Task 2 — LAG(amount, 3): compare each order to 3 orders prior (3/5)
3. Task 3 — Triple LEFT JOIN aggregation + composite engagement tier (5/5)


### Task Archive: 2026-04-17 (Week 18, Day 5)

# Daily SQL Practice Tasks

**Generated:** 2026-04-17
**Week 18, Day 5 Focus:** Query optimization concepts + session gaps-and-islands + YoY window

---

## Task 1: Query Optimization — Rewrite a Slow Query

**Scenario:**
The following query is logically correct but poorly written — it uses a correlated subquery in the SELECT clause that re-executes for every row, and a WHERE subquery that also re-scans the table. Your job is to rewrite it to be more efficient using CTEs or JOINs, without changing the result.

**Original slow query:**
```sql
SELECT
    u.id AS user_id,
    u.country,
    (SELECT SUM(o.amount) FROM orders o WHERE o.user_id = u.id) AS total_spent,
    (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) AS order_count
FROM users u
WHERE (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) >= 5
ORDER BY total_spent DESC;
```

**Your task:**
1. Rewrite this query using a CTE + JOIN to eliminate the correlated subqueries
2. Add a brief SQL comment (1-2 lines) explaining why the original is slow and why your version is faster

**Expected Output Columns:** same as original — `user_id`, `country`, `total_spent`, `order_count`

**Difficulty Rating:** 3/5

**Student Solution:**
```sql
SELECT 
    o.user_id,
    u.country,
    SUM(o.amount) AS total_spent,
    COUNT(o.id) AS order_count
FROM crappy_data_db.users u
JOIN crappy_data_db.orders o ON o.user_id = u.id
GROUP BY o.user_id, u.country
HAVING COUNT(*) >= 5
ORDER BY total_spent DESC
```
EXPLAIN ANALYZE: 8.6ms → 0.3ms, buffers 127 → 29. Score: 10/10.

---

## Task 2: Time-Based Gaps — User Session Inactivity Periods (5/5)

**Scenario:**
Find all pairs of consecutive active days per user where the gap > 14 days.

**Expected Output Columns:** `user_id`, `last_active_date`, `next_active_date`, `gap_days`

**Difficulty Rating:** 5/5

**Student Solution:** Used LAG instead of LEAD (gap calculated backward not forward). Missing count_sessions > 0 filter. HAVING >= 14 vs > 14. Score: 8/10.

---

## Task 3: YoY Comparison with Window — Monthly Revenue Trend (4/5)

**Scenario:**
Monthly revenue with LAG(12) for same-month prior year comparison.

**Expected Output Columns:** `month`, `revenue`, `prev_year_revenue`, `yoy_diff`

**Difficulty Rating:** 4/5

**Student Solution:** LAG(total_revenue, 12) correct. GROUP BY YEAR, MONTH redundant (MONTH contains year). Score: 9/10.

**Day Score: 27/30**

---

## Submission Instructions

1. Task 1 — Query optimization rewrite (3/5)
2. Task 2 — LEAD-based inactivity gap detection > 14 days (5/5)
3. Task 3 — Monthly revenue with LAG(12) YoY comparison (4/5)

### Task Archive: 2026-04-20 (Week 19, Day 1)

# Daily SQL Practice Tasks

**Generated:** 2026-04-20
**Week 19, Day 1 Focus:** Anti-join patterns + conditional aggregation GROUP BY

---

## Task 1: Anti-Join — Users Who Never Placed an Order

**Scenario:**
The CRM team wants a list of users who have **never placed any order** — to target them with a first-purchase campaign.

Write this query using whichever anti-join approach you prefer (`NOT IN`, `NOT EXISTS`, or `LEFT JOIN ... WHERE IS NULL`). Output `user_id` only.

**Then answer in a comment:** Which of the three approaches breaks silently if `orders.user_id` contains NULLs, and why?

**Expected Output Columns:** `user_id`
**Tables:** `users`, `orders`
**Difficulty Rating:** 3/5

**Student Solution:**
```sql
SELECT 
    u.id AS user_id
FROM crappy_data_db.users u
WHERE NOT EXISTS
(SELECT o.user_id FROM crappy_data_db.orders o
WHERE o.user_id = u.id AND o.user_id IS NOT NULL
)
```

---

## Task 2: Conditional Aggregation — Transaction Type Breakdown per User

**Scenario:**
Per-user transaction pivot by type (deposit, withdrawal, purchase counts).

**Difficulty Rating:** 4/5

**Student Solution:**
```sql
WITH users_transactions_breakdown AS (
SELECT 
    user_id,
    COUNT(id) AS total_transactions,
    COUNT(id) FILTER (WHERE TYPE = 'deposit') AS deposit_count,
    COUNT(id) FILTER (WHERE TYPE = 'withdrawal') AS withdrawal_count,
    COUNT(id) FILTER (WHERE TYPE = 'purchase') AS purchase_count
FROM crappy_data_db.transactions t
GROUP BY user_id
```
(Session ended early — student was tired)

---

## Submission Instructions

1. Task 1 — Anti-join approach + NULL trap explanation (3/5)
2. Task 2 — Per-user transaction type breakdown (4/5)

### Task Archive: 2026-04-21 (Week 19, Day 2)

**Focus:** dominant_type via RANK + Type A recursive CTE rollup + NULLIF safe division
**Day Score: 25/30**

**Task 1 (dominant_type):** Three CTEs — per-type filter counts, long aggregation, RANK. Clean JOIN on rank=1. Alphabetical tie-breaking included. Bonus per-type counts included. 10/10.

**Task 2 (3-level rollup):** Used WITH RECURSIVE + self-joining HIERARCHY — wrong tool. Type A pattern is plain UNION ALL of three independent CTEs, no recursion. JOIN logic on label names fragile and incorrect. Student noted it "doesn't seem real" — correct instinct. 6/10.

**Task 3 (NULLIF):** Verified data has no zero-session rows, adapted correctly. Simple one-liner cleaner than CTE version. NULLIF not needed given actual data. 9/10.

### Task Archive: 2026-04-22 (Week 19, Day 3)

**Focus:** Type A fixed-hierarchy UNION ALL + PERCENT_RANK bands + gaps-and-islands monthly streaks
**Day Score: 26/30**

**Task 1 (Type A rollup):** Correct logic, still used WITH RECURSIVE + self-join unnecessarily. Pattern should be two independent CTEs + plain UNION ALL. 8/10.

**Task 2 (PERCENT_RANK):** Good structure. Minor typos: 075 instead of 0.75, 'bottom' instead of 'bottom 25%'. 9/10.

**Task 3 (Monthly streaks):** Solid gaps-and-islands. SUM(is_new_streak) group ID correct. COUNT(DISTINCT month) handles duplicates correctly. ROW_NUMBER needs alias to avoid runtime error. Student verified only 1 streak per user in data. 9/10.

### Task Archive: 2026-04-23 (Week 19, Day 4)

**Focus:** LAG between orders + self-join affinity + cohort retention
**Day Score: 29/30**

**Task 1 (LAG between orders):** LAG with DESC order + ABS for days difference — works correctly. No NULL amounts in data. 10/10.

**Task 2 (Self-join Jan+Feb):** Clean self-join on user_id with different month filters on each alias. DISTINCT correct. 10/10.

**Task 3 (Cohort retention):** Works. Used LEAD on distinct months instead of LEFT JOIN pattern from spec. INTERVAL '31 days' slightly loose — INTERVAL '1 month' is calendar-aware and cleaner. Could simplify with MIN(created_at) GROUP BY instead of FIRST_VALUE window. 9/10.

### Task Archive: 2026-04-24 (Week 19, Day 5)

**Focus:** STDDEV volatility + ticket response time + funnel analysis
**Day Score: 30/30**

**Task 1 (STDDEV volatility):** Used window functions instead of GROUP BY but result identical. Student verified all users have 2+ transactions so HAVING filter unnecessary. 10/10.

**Task 2 (Ticket response time):** Adapted to minutes (data showed 5-8 min averages, not hours). Filtered on author_id IS NOT NULL to isolate agent responses — smart business logic interpretation. 10/10.

**Task 3 (Funnel analysis):** Used pending status for "has delivery record" — correct, as pending is the initial status and avoids double-counting. funnel_step labels were numeric strings rather than descriptive text but logic was sound. 10/10.

### Task Archive: 2026-04-27 (Week 20, Day 1)

**Focus:** Type A CTE drill + NTILE + EXPLAIN ANALYZE query optimization
**Day Score: 30/30**

**Task 1 (Type A CTE):** Two independent CTEs, plain UNION ALL, no recursion. Pattern clicked naturally. 10/10.

**Task 2 (NTILE):** Two CTEs, NTILE(4) applied cleanly, CASE labels correct. 10/10.

**Task 3 (Query optimization):** 5.944ms → 0.383ms (15x faster). Correct explanation: correlated subquery re-runs AVG per row vs single aggregation pass. Included avg_transaction as bonus output. 10/10.

### Task Archive: 2026-04-29 (Week 20, Days 2+3)

**Focus:** FIRST_VALUE + rolling SUM + anti-join NULL trap + LAG offset + YoY + NULLIF
**Day Score: 51/60**

**Task 1 (FIRST_VALUE):** Clean, correct. FIRST_VALUE(amount ORDER BY created_at DESC) with PARTITION BY. 10/10.

**Task 2 (Rolling 3-order SUM):** ROWS BETWEEN 2 PRECEDING AND CURRENT ROW correct. 10/10.

**Task 3 (Anti-join NULL trap):** All three versions written but join direction inverted — queried from orders_products instead of products. Student requested settling on NOT EXISTS going forward. 6/10.

**Task 4 (LAG offset):** LAG(amount, 3) clean one-liner. 10/10.

**Task 5 (YoY NULLIF):** LAG(12) correct. NULLIF missing from denominator in division. 8/10.

**Task 6 (NULLIF + COALESCE):** COUNT(NULLIF(city, '')) correct and verified working on real data. COALESCE missing but data has no zero-city countries. 9/10.

### Task Archive: 2026-04-30 (Week 20, Day 4)

**Focus:** NOT EXISTS anti-join + YoY NULLIF + window frame comparison
**Day Score: 29/30**

**Task 1 (NOT EXISTS):** Correct direction — FROM orders WHERE NOT EXISTS (deliveries). All orders have deliveries in data. 10/10.

**Task 2 (YoY NULLIF):** NULLIF(lag(revenue,12), 0) in denominator correct. LAG(12) correct. ::NUMERIC cast for ROUND handled properly. 10/10.

**Task 3 (Window frames):** All three frames in one SELECT correct. Default RANGE frame for running_total works without ties. Explanation comment missing. 9/10.

### Task Archive: 2026-05-01 (Week 20, Day 5)

**Focus:** PERCENT_RANK revisit + cohort retention LEFT JOIN pattern
**Day Score: 18/20**

**Task 1 (PERCENT_RANK):** Clean CTE + boolean expression pct_rank >= 0.9 directly. No CASE WHEN needed. 10/10.

**Task 2 (Cohort retention):** Smart second-order approach via ROW_NUMBER instead of LEFT JOIN. Interval condition caught only 1 month instead of 3 months as specified. 8/10.

### Task Archive: 2026-05-04 (Week 21, Day 1)

**Focus:** job_db exploration - platform/seniority distribution + city dominance + VALUES CROSS JOIN ILIKE
**Day Score: 28/30**

**Task 1 (Platform × seniority):** Clean two-JOIN GROUP BY. 10/10.

**Task 2 (Top 5 cities per seniority):** ROW_NUMBER instead of RANK — safer for top-N. NULL filter in right place. 10/10.

**Task 3 (VALUES + CROSS JOIN + ILIKE):** Needed scaffolding. Used COUNT FILTER instead of WHERE which is smart. Pattern not locked in yet — needs more reps. 8/10.

### Task Archive: 2026-05-05 (Week 21, Day 2)

**Focus:** VALUES CROSS JOIN repeat + salary text parsing + platform share per seniority
**Day Score: 25/30**

**Task 1 (VALUES + CROSS JOIN):** Pattern clicked independently this time. COUNT FILTER clean. Missing seniority name JOIN — used seniority_id instead. 9/10.

**Task 2 (Salary parsing):** REGEXP_MATCH needed scaffolding — genuinely complex. Aggregation structure around it correct. REGEXP not expected to be independent yet. 8/10.

**Task 3 (Platform share):** Window SUM correct, percentage calculation correct. Column aliases swapped — platforma labelled as seniority and vice versa. 8/10.

### Task Archive: 2026-05-06 (Week 21, Day 3)

**Focus:** GROUP BY multi-dimension + VALUES CROSS JOIN 3rd rep + cumulative SUM on job data
**Day Score: 29/30**

**Task 1 (Work type per platform):** Clean JOIN, correct GROUP BY, NULLs filtered properly. 10/10.

**Task 2 (VALUES + CROSS JOIN):** Independent this time, seniority JOIN included correctly. Typo 'Statoniary' won't match data but not a logic error. 9/10.

**Task 3 (Cumulative offers):** CTE for daily counts, window SUM with explicit UNBOUNDED PRECEDING frame correct. 10/10.

### Task Archive: 2026-05-07 (Week 21, Day 4)

**Focus:** NOT EXISTS anti-join + YoY offer count LAG(12) + dominant work type per platform
**Day Score: 24/30**

**Task 1 (NOT EXISTS):** Direction inverted again — FROM oferty instead of FROM seniority. Condition `zarobki IS NULL AND zarobki LIKE '%PLN%'` can never be true simultaneously. Anti-join direction keeps failing: must start FROM the table being checked, NOT EXISTS against the excluding table. 4/10.

**Task 2 (YoY LAG(12)):** Three CTEs, LAG(12) correct, yoy_diff NULL propagation works. 10/10.

**Task 3 (Dominant work type):** Pattern fully independent. GROUP BY → RANK → filter rank=1. Joined to platforma in CTE 1 for the name. 10/10.

**Note:** Anti-join confusion traced to drilling three versions at once. Going forward: NOT EXISTS only, one rep at a time.

### Task Archive: 2026-05-08 (Week 21, Day 5)

**Focus:** NOT EXISTS clean drill + NTILE seniority tiers + self-join platform co-occurrence
**Day Score: 29/30**

**Task 1 (NOT EXISTS):** Correct direction — FROM platforma WHERE NOT EXISTS (oferty). Pattern finally clicked. 10/10.

**Task 2 (NTILE):** Two CTEs, NTILE(4) correct, CASE labels right. 10/10.

**Task 3 (Self-join):** Works. Used > in WHERE instead of < in JOIN ON — same result, slightly less efficient. 9/10.

### Task Archive: 2026-05-08 (Week 21, Day 5) - already archived above, see Week 21 recap

### Task Archive: 2026-05-11 (Week 22, Day 1)

**Focus:** crappy_data_db warm-up — GROUP BY + RANK within country + NOT EXISTS
**Day Score: 29/30**

**Task 1 (Country order stats):** CTE aggregation, all columns correct. COUNT(DISTINCT u.id) correct. 10/10.

**Task 2 (RANK within country):** CTE for spend, RANK() PARTITION BY country correct. Missing alias on RANK() column — cosmetic. 9/10.

**Task 3 (NOT EXISTS):** Correct direction — FROM users WHERE NOT EXISTS (chat_tickets). Pattern cemented. 10/10.

### Task Archive: 2026-05-12 (Week 22, Day 2)

**Focus:** EPOCH ticket resolution + cohort LEFT JOIN + STDDEV cross-table
**Day Score: 23/30**

**Task 1 (EPOCH resolution):** AVG(interval) then EXTRACT EPOCH — cleaner than per-row EPOCH then AVG. 10/10.

**Task 2 (Cohort LEFT JOIN):** LEFT JOIN pattern correct. Interval <= 3 months instead of < 4 months — misses boundary. GROUP BY in second CTE redundant with DISTINCT in final SELECT. 8/10.

**Task 3 (STDDEV cross-table):** STDDEV computed on orders instead of transactions (misread). Window function + GROUP BY conflict — should use STDDEV(amount) as GROUP BY aggregation, not OVER. Duplicate column alias. 5/10.

### Task Archive: 2026-05-13 (Week 22, Day 3)

**Focus:** STDDEV GROUP BY drill + PERCENT_RANK job_db + cross-schema city JOIN
**Day Score: 29/30**

**Task 1 (STDDEV GROUP BY):** Clean — plain STDDEV(amount) aggregation, no window. tx_count >= 2 filter in outer query correct. 10/10.

**Task 2 (PERCENT_RANK):** CTE for counts, PERCENT_RANK() OVER correct. NULL filter on platform name instead of platforma_id — same effect. 10/10.

**Task 3 (Cross-schema JOIN):** COUNT(DISTINCT data_wystawienia) as proxy for distinct offers — reasonable given no PK on oferty. Student correctly identified no perfect deduplication option exists without a surrogate key. 9/10.

### Task Archive: 2026-05-14 (Week 22, Day 4)

**Focus:** MoM LAG job_db + NTILE spend tiers cross-table + gaps-and-islands posting gaps
**Day Score: 29/30**

**Task 1 (LAG MoM):** Three CTEs, LAG(1) correct. ABS on mom_diff valid choice. 10/10.

**Task 2 (NTILE cross-table):** Skipped unnecessary CTE, joined directly to orders — cleaner. AVG per-order amount valid interpretation. 10/10.

**Task 3 (Gaps-and-islands):** Independently solved on new schema. EPOCH with average month seconds correct approach. Platform shown as ID not nazwa — missing JOIN to platforma. Only one qualifying gap in data. 9/10.

### Task Archive: 2026-05-15 (Week 22, Day 5)

**Focus:** Light Friday — GROUP BY + LAG + NOT EXISTS refresh
**Day Score: 30/30**

**Task 1 (Offer count by seniority + work type):** Clean JOIN, GROUP BY correct, NULLs filtered. 10/10.

**Task 2 (LAG prev amount):** LAG(amount) OVER PARTITION BY user_id correct. No NULL filter needed — verified no NULLs in transactions.amount. 10/10.

**Task 3 (NOT EXISTS 2025):** Correct direction — FROM seniority WHERE NOT EXISTS (oferty). Year filter correct. 10/10.

### Task Archive: 2026-05-18 (Week 23, Day 1)

**Focus:** GROUP BY + HAVING on job_db + MoM LAG per user
**Day Score: 20/20**

**Task 1 (GROUP BY + HAVING — high-volume platforms by contract type):** JOIN to platforma correct, WHERE filters NULLs, HAVING COUNT(*) >= 100 (spec said > 100, minor boundary, data-justified). GROUP BY and ORDER BY clean. 10/10.

**Task 2 (LAG MoM revenue per user):** Clean 3-CTE chain — DATE_TRUNC for month, SUM per user+month, LAG(1) partitioned by user, mom_diff at final SELECT. 10/10.


### Task Archive: 2026-05-19 (Week 23, Day 2)

**Focus:** Type A CTE + Cohort LEFT JOIN + FIRST_VALUE on job_db
**Day Score: 26/30**

**Task 1 (Type A CTE revenue rollup):** Three independent UNION ALL blocks correct. Redundant JOIN to orders table not needed (line revenue only needs orders_products + products + product_categories). No WHERE for NULL price/quantity as specified. Result correct. 10/10.

**Task 2 (Cohort 3-month retention):** Three issues — (1) missing lower bound: orders in registration month itself (month 0) incorrectly included; (2) WHERE o.id IS NOT NULL converts LEFT JOIN to INNER JOIN, drops cohorts with 0 retention; (3) formula was churn rate not retention rate (cohort_size - retained) / cohort_size vs retained / cohort_size. 6/10.

**Task 3 (FIRST_VALUE latest offer per platform):** Clean. DISTINCT deduplicates, FIRST_VALUE with DESC sort correct, INNER JOIN filters NULLs implicitly. 10/10.


### Task Archive: 2026-05-20 (Week 23, Day 3)

**Focus:** Cohort retention retry + cumulative SUM frame + cross-schema seniority
**Day Score: 27/30**

**Task 1 (Cohort 3-month retention retry):** LEFT JOIN structure correct, no WHERE killing it, date bounds correct via DATE_TRUNC comparison. Retention rate formula correct. COUNT(DISTINCT o.user_id) natural 0 for unmatched cohorts. 9/10 — minor: COUNT(DISTINCT(col)) style, DISTINCT is a clause not a function.

**Task 2 (Cumulative SUM with frame):** Perfect. Explicit ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW, PARTITION BY user_id, NULL filter in WHERE. 10/10.

**Task 3 (Cross-schema seniority by shared cities):** JOIN and GROUP BY correct. COUNT(DISTINCT o.pozycja) counts distinct titles not offers — should be COUNT(*) since oferty has no PK. Missing u.city IS NOT NULL filter. 8/10.


### Task Archive: 2026-05-21 (Week 23, Day 4)

**Focus:** dominant_type CTE pattern + PERCENT_RANK on job_db + NULLIF safe division
**Day Score: 26/30**

**Task 1 (dominant_type via two CTEs):** Pattern correct — aggregate first CTE, RANK second CTE, filter rank=1. Used ROW_NUMBER() instead of RANK() — ROW_NUMBER breaks tie-inclusion requirement. NULL filters placed in second CTE instead of first (after aggregation). 8/10.

**Task 2 (PERCENT_RANK within seniority):** Clean. Correct partition, order, ROUND with NUMERIC cast, NULL filters in WHERE. Extra seniority_id column not penalized. 10/10.

**Task 3 (NULLIF safe avg):** NULLIF applied to data values (NULLIF(amount, 0)) instead of denominator only. This treats zero amounts as invalid — zeros are valid order values. NULLIF belongs only in the division guard: total_revenue / NULLIF(valid_order_count, 0). COUNT/SUM naturally ignore NULLs. 8/10.


### Task Archive: 2026-05-22 (Week 23, Day 5)

**Focus:** Light Friday — dominant_type RANK fix + NULLIF denominator
**Day Score: 18/20**

**Task 1 (dominant_type RANK fix):** RANK() correct, ties preserved, two-CTE pattern clean. NULL filters in second CTE (minor style — functionally fine). 10/10.

**Task 2 (NULLIF denominator only):** SUM(NULLIF(amount, 0)) still present — zeros incorrectly excluded. Should be plain SUM(amount). NULLIF on denominator correct. 8/10.


### Task Archive: 2026-06-02 (Week 24, Day 1 — NQ Project begins)

**Focus:** daily_ohlcv_globex materialized view — Layer 1 foundation
**Notes:**
- First attempt used FIRST_VALUE window functions — ran 10 minutes on 56M rows
- Rewrote using aggregate-then-JOIN approach: MIN/MAX ts_event to find open/close times, join back to ticks for price. Significantly faster.
- Trade date label: ((ts_event AT TIME ZONE America/New_York) - INTERVAL 18 hours)::date + 1
- Globex session: 18:00 ET prev day to 17:00 ET close day
- Created as materialized view: nq_data.daily_ohlcv_globex
- Includes: trade_date, weekday, open, high, low, close, total_volume, buy_volume, sell_volume, tick_count


### Task Archive: 2026-06-03 (Week 24, Day 2)

**Focus:** RTH materialized view + daily range by weekday + buy/sell pressure by weekday
**Day Score: 30/30**

**Task 1 (daily_ohlcv_rth):** Clean. ::time cast for RTH filter, ::date for trade_date, aggregate-then-JOIN for open/close, N-side excluded. 10/10.

**Task 2 (daily range by weekday):** Used daily_ohlcv_rth instead of globex — better choice for trader analysis. Clean CTE, correct aggregation. Results: Fri/Thu highest avg range (~370pts), Mon lowest (~298pts). 10/10.

**Task 3 (buy/sell pressure by weekday):** Clean, NULLIF on denominator. Key finding: ratio 0.995-1.001 across all weekdays — near-perfect buy/sell symmetry at daily aggregate level. 10/10.


### Task Archive: 2026-06-04 (Week 24, Day 3)

**Focus:** First hour range vs full day + volume by hour + gap analysis
**Day Score: 27/30**

**Task 1 (RTH-RANGE-002 — first hour range vs full day):** Extended beyond spec to include rest-of-session OHLC and fh_pct_of_day. Correct aggregate-then-JOIN pattern. Duplicate rows from timestamp ties in point-lookup JOINs — needs DISTINCT ON fix. Key lesson: never JOIN raw ticks on range condition, always filter with ::time directly. 8/10.

**Task 2 (RTH-VOL-002 — volume by hour):** Clean two-pass aggregation. Two-step approach (per day then average) correct. Window function for pct correct. 10/10.

**Task 3 (RTH-GAP-001 — overnight gap):** LAG pattern correct, FILTER aggregation clean. avg_gap not rounded (minor). 9/10.


### Task Archive: 2026-06-05 (Week 24, Day 4)

[Solutions to be added after session]
