### Task Archive

(Archive of completed `tasks.md` snapshots will be appended here with a timestamp header.)

---

### Task Archive: 2025-12-04 (Previous Tasks - Agent-generated before student started)

### Tasks: Daily Exercise Sheet

Date: 2025-12-04

Instructions: Solve each problem using only schema table/column names found in `schema.md`. Provide the final SQL query, brief explanation, and expected sample output rows.

---

Title: Top Product Categories by Revenue (Last 30 Days)

Scenario/Business Question:
Calculate total revenue per product category for orders placed in the last 30 days. Use `orders`, `orders_products`, `products`, and `product_categories`. Exclude NULL category names.

Expected Output Columns:
- category_id
- category_name
- total_revenue

Difficulty Rating: 3

---

Title: Repeat Customers — Average Days Between Orders

Scenario/Business Question:
For users with at least 2 orders, compute average days between consecutive orders. Use window functions (LAG) over `orders` partitioned by `user_id`.

Expected Output Columns:
- user_id
- first_order_date
- last_order_date
- avg_days_between_orders

Difficulty Rating: 4

---

Title: 7-Day Rolling Sessions and Active Users

Scenario/Business Question:
Produce daily totals of sessions and number of active users (users with count_sessions > 0) and a 7-day rolling average of total sessions. Use `dates` and `user_sessions_daily`.

Expected Output Columns:
- date
- total_sessions
- active_users
- sessions_7d_avg

Difficulty Rating: 4

---

Title: Top Open Chat Tickets and Message Activity

Scenario/Business Question:
List currently open or assigned chat tickets (status NOT IN ('resolved','closed')) with message counts and last message timestamp. Use `chat_tickets` and `chat_messages`.

Expected Output Columns:
- ticket_id
- user_id
- priority
- status
- message_count
- last_message_at

Difficulty Rating: 3

---

Title: Top Users by Lifetime Value (Orders + Transactions)

Scenario/Business Question:
Compute lifetime value per user as sum of `orders.amount` plus `transactions.amount`. Show top 20 users by combined value. Use `users`, `orders`, `transactions`.

Expected Output Columns:
- user_id
- total_order_amount
- total_transaction_amount
- lifetime_value

Difficulty Rating: 5

---

Notes:
- Use `schema.md` as single source of truth for column names and nullability.
- Keep queries readable, use CTEs where helpful, and prefer explicit joins.

# Daily SQL Practice Tasks

**Generated:** 2025-12-04
**Week 1 Focus:** Advanced JOINs, Window Functions, CTEs, Aggregations

---

## Task 1: Self-Join — Session Buddies

**Scenario:**
The product analytics team wants to identify "session buddies" — pairs of users who had the exact same number of sessions on at least 3 different dates. This helps understand synchronized user behavior patterns.

**Expected Output Columns:**
- `user_id_1` (integer) — first user ID (should be less than user_id_2)
- `user_id_2` (integer) — second user ID
- `matching_days` (bigint) — count of dates where they had identical session counts

**Requirements:**
- Use `user_sessions_daily` table
- Self-join to compare users
- Only include pairs where matching_days >= 3
- Ensure user_id_1 < user_id_2 to avoid duplicate pairs
- Order by `matching_days` DESC, then `user_id_1` ASC

**Difficulty Rating:** 3/5

WITH users_sessions_comparison AS (
	SELECT
		usd_first.user_id AS user_id_1,
		usd_first.date AS date_1,
		usd_first.count_sessions AS sessions_1,
		usd_second.user_id AS user_id_2,
		usd_second.count_sessions AS sessions_2
	FROM user_sessions_daily usd_first
	JOIN user_sessions_daily usd_second ON usd_first.date = usd_second.date
	),
users_matching_days AS (
	SELECT
		user_id_1,
		user_id_2,
		COUNT(*) AS matching_days
	FROM users_sessions_comparison
	WHERE sessions_1 = sessions_2
	AND user_id_1 != user_id_2 
	GROUP BY user_id_1, user_id_2
	),
matching_check AS (
	SELECT * FROM users_matching_days 
	WHERE matching_days >= 3
	AND user_id_1 > user_id_2
	ORDER BY matching_days, user_id_1
	)
SELECT COUNT(*) FROM matching_check

#You gave me a great tip with user_id_1 > user_id_2, as I was wondering how to avoid duplicates. I got 889 pairs of users, hoping it's true. Your question is alright, probably not a very real scenario for business cases though.


---

## Task 2: First vs. Last Transaction Type Per User

**Scenario:**
The finance team wants to analyze user transaction journey patterns. Find all users whose first transaction type differs from their last transaction type, and show how many transactions they made in total.

**Expected Output Columns:**
- `user_id` (integer)
- `first_name` (varchar)
- `last_name` (varchar)
- `first_transaction_type` (text)
- `last_transaction_type` (text)
- `total_transactions` (bigint)

**Requirements:**
- Use `transactions` and `users` tables
- Apply window functions: FIRST_VALUE and LAST_VALUE
- Include only users with at least 2 transactions
- Filter for cases where first_transaction_type != last_transaction_type
- Order by `total_transactions` DESC

**Difficulty Rating:** 3/5

WITH users_transactions_types AS (
	SELECT
	 	*,
		 FIRST_VALUE(type) OVER (PARTITION BY user_id ORDER BY created_at) AS first_transaction_type,
		 FIRST_VALUE(type) OVER (PARTITION BY user_id ORDER BY created_at DESC) AS last_transaction_type
	 FROM transactions
	 )
SELECT
	user_id,
	first_transaction_type,
	last_transaction_type,
	COUNT(*) AS total_transactions
FROM users_transactions_types
WHERE first_transaction_type != last_transaction_type
GROUP BY user_id, first_transaction_type, last_transaction_type 
ORDER BY total_transactions DESC

#Note - first_transaction_type, last_transaction_type in the last GROUP BY are there just for the need - I simply wanted to show them - they also return no duplicates, as we obviously have just one distinct first_transaction_Type and last_transaction_type for every user.



---

## Task 3: Rolling 7-Day Order Revenue

**Scenario:**
Calculate a 7-day rolling average of daily order revenue. For each date, compute the average order amount over that date plus the 6 preceding days. Include all dates from the dates table, even if there were no orders.

**Expected Output Columns:**
- `date` (date)
- `daily_revenue` (numeric) — total order amount for that specific date (NULL if no orders)
- `rolling_7day_avg` (numeric) — average daily revenue over 7-day window

**Requirements:**
- Use `dates` table as base (to include all dates)
- LEFT JOIN with aggregated orders data
- Window frame: ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
- Only return dates from year 2025
- Order by date ASC

**Difficulty Rating:** 4/5


WITH joined_dates_coalesce_amts AS (
	SELECT *,
	COALESCE(o.amount, 0) AS order_amount
	FROM dates d
	LEFT JOIN orders o ON d.date = DATE(o.created_at)
	ORDER BY d.date
	)
SELECT *,
AVG(order_amount) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS avg_last_7_days
FROM joined_dates_coalesce_amts
WHERE EXTRACT('Year' FROM "date" ) = 2025

- That was my first solution, but I noticed it may have skewed results, as here we're counting each row as a date, whereas where there are multiple orders on one day, we're not making sure to count an actual average from 7 days, but rather from the random number of days, depending on the number of our orders from that given day and preceding (e.g. if there's 3 orders on one day, and 4 orders on previous day, we would be in fact counting an avg from the last 2 days (last 7 rows))

WITH joined_dates_coalesce_amts AS (
	SELECT *,
	COALESCE(o.amount, 0) AS order_amount
	FROM dates d
	LEFT JOIN orders o ON d.date = DATE(o.created_at)
	ORDER BY d.date
	),
summed_days AS (
	SELECT 
		date,
		SUM(order_amount) OVER (PARTITION BY date) AS sum_orders_per_day
	FROM joined_dates_coalesce_amts
	WHERE EXTRACT('Year' FROM "date" ) = 2025
	)
SELECT *,
AVG(sum_orders_per_day) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS avg_last_7_days
FROM summed_days

This approach should yield proper results, but perhaps I'm wrong - correct me if that's the case.



---

## Task 4: Anti-Join — Users Who Never Bought Travel Products

**Scenario:**
The marketing team wants to launch a travel campaign targeting users who have made purchases but have NEVER bought anything from the "travel" category. Find these users.

**Expected Output Columns:**
- `user_id` (integer)
- `email` (varchar)
- `city` (varchar)
- `total_orders` (bigint) — count of all orders made by this user

**Requirements:**
- Use `users`, `orders`, `orders_products`, `products`, `product_categories` tables
- Use an anti-join pattern (NOT EXISTS or LEFT JOIN ... WHERE NULL)
- Exclude users who have purchased ANY product from category "travel"
- Include only users with at least 1 order
- Order by `total_orders` DESC

**Difficulty Rating:** 4/5


WITH users_with_travel_products_purchases AS (
	SELECT 
		DISTINCT(o.user_id)
	FROM orders o
	JOIN orders_products op ON op.order_id = o.id
	JOIN products p  ON op.product_id = p.id 
	JOIN product_categories pc ON p.category_id = pc.id
	WHERE pc."name"  = 'travel'
	),
users_except_travel_purchasers AS (
	SELECT user_id FROM orders o
	EXCEPT
	SELECT user_id FROM users_with_travel_products_purchases
	)
SELECT 
uetp.user_id,
COUNT(*) AS total_orders
FROM users_except_travel_purchasers uetp
JOIN orders o ON uetp.user_id = o.user_id
GROUP BY uetp.user_id
ORDER BY total_orders DESC

#Pretty self_explanatory - I also like that question/query, as it seems to also make sense in a real life scenario.


---

## Task 5: Ticket Resolution Time Ranking

**Scenario:**
For each priority level, rank support tickets by their resolution time (from creation to the timestamp of the last message). Use DENSE_RANK so that tickets with identical resolution times receive the same rank.

**Expected Output Columns:**
- `ticket_id` (bigint)
- `priority` (text)
- `resolution_time` (interval) — time from ticket creation to last message
- `rank_in_priority` (bigint) — dense rank within priority group (1 = fastest)

**Requirements:**
- Use `chat_tickets` and `chat_messages` tables
- Only include tickets with status = 'resolved' OR status = 'closed'
- Calculate resolution_time as: MAX(chat_messages.created_at) - chat_tickets.created_at
- Apply DENSE_RANK() OVER (PARTITION BY priority ORDER BY resolution_time ASC)
- Order final output by `priority` ASC, then `rank_in_priority` ASC

**Difficulty Rating:** 4/5

WITH tickets_reso_times AS (
	SELECT 
		*,
		FIRST_VALUE(cm.created_at) OVER (PARTITION BY ct.id ORDER BY cm.created_at) AS ticket_creation_time,
		FIRST_VALUE(cm.created_at) OVER (PARTITION BY ct.id ORDER BY cm.created_at DESC) AS ticket_resolution_time
	FROM chat_tickets ct
	JOIN chat_messages cm ON ct.id = cm.ticket_id
	WHERE ct.status = 'resolved' OR ct.status = 'closed'
	),
tickets_calculated_reso_times AS (
	SELECT 
		DISTINCT(ticket_id),
		priority,
		ticket_resolution_time - ticket_creation_time AS resolution_time
	FROM tickets_reso_times
	)
SELECT *, 
DENSE_RANK() OVER (PARTITION BY priority ORDER BY resolution_time ASC) AS rank_in_priority
FROM tickets_calculated_reso_times
ORDER BY priority, rank_in_priority

I liked that query, it makes sense as well.


---

## Submission Instructions

When you're ready, submit your SQL solutions one at a time or all together. I'll provide detailed feedback including:
- Correctness of logic and syntax
- Efficiency and performance considerations
- Best practice suggestions
- Advanced alternatives (if applicable)

## Tips

- Build your queries incrementally — start simple, then add complexity
- Use CTEs to break down multi-step logic for readability
- Always check the schema.md file for exact column names and data types
- Consider NULL handling in your joins and aggregations
- Test window function frames carefully (ROWS vs RANGE)

Good luck!

### Task Archive: 2025-12-04 (Completed with Solutions)


---

# Daily SQL Practice Tasks

**Generated:** 2025-12-05
**Week 1, Day 2 Focus:** Advanced Aggregations, LAG/LEAD, Cross Joins, CTEs

---

## Task 1: Customer Churn Analysis — LAG Function

**Scenario:**
The business analytics team needs to identify users who are at risk of churning. Calculate the gap (in days) between consecutive orders for each user, then find users whose most recent order gap is more than 2x their average gap between orders.

**Expected Output Columns:**
- `user_id` (integer)
- `first_name` (varchar)
- `last_name` (varchar)
- `avg_days_between_orders` (numeric) — average gap between all consecutive orders
- `most_recent_gap_days` (numeric) — days since their last order
- `churn_risk_ratio` (numeric) — most_recent_gap / avg_gap

**Requirements:**
- Use `orders` and `users` tables
- Apply LAG() window function to calculate gaps between consecutive orders
- Only include users with at least 3 orders (to calculate meaningful averages)
- Filter for users where `churn_risk_ratio > 2.0`
- Order by `churn_risk_ratio` DESC

**Difficulty Rating:** 4/5

WITH users_order_gaps AS (
	SELECT
		user_id,
		date(created_at) AS order_date,
		COUNT(*) AS orders_cnt,
		LAG(date(created_at)) OVER (PARTITION BY user_id ORDER BY date(created_at)) AS prev_user_order_day
	FROM orders
	GROUP BY user_id, date(created_at)
	ORDER BY user_id
	),
users_day_diffs AS (
SELECT
	user_id,
	order_date,
	prev_user_order_day,
	order_date - prev_user_order_day AS days_diff
FROM users_order_gaps
	),
users_avg_diffs AS (
	SELECT 
		user_id,
		COUNT(*) AS orders_cnt,
		ROUND(AVG(days_diff),2) AS avg_diff
	FROM users_day_diffs
	GROUP BY user_id
	HAVING COUNT(*) >= 3
	),
users_above_the_threshold AS (
	SELECT
		udd.user_id,
		udd.days_diff,
		uad.avg_diff,
		FIRST_VALUE(udd.order_date) OVER (PARTITION BY uad.user_id ORDER BY udd.order_date DESC) AS last_order_date
	FROM users_avg_diffs uad
	JOIN users_day_diffs udd ON uad.user_id = udd.user_id
	WHERE udd.days_diff > 2 * (uad.avg_diff)
	)
SELECT
uatt.user_id,
u.first_name,
u.last_name,
uatt.avg_diff,
uatt.last_order_date,
DATE(NOW()) -  uatt.last_order_date AS most_recent_gap_days,
ROUND((DATE(NOW()) -  uatt.last_order_date) / uatt.avg_diff, 2) AS churn_risk_ratio
FROM users_above_the_threshold uatt
JOIN users u ON uatt.user_id  = u.id
ORDER BY churn_risk_ratio DESC


This was probably the longest task I've worked on so far - I followed your requirements precisely this time, although IMO it's a bit unnecessary to expect me to show first_name, last_name etc. Of course I  can extract these without any issues, but that's just makes the whole code base longer without adding real learning value to that - this is a trivial task, I obviously know how to do it and I don't think it's necessary. Note that for future reference and certainly improve on that!



---

## Task 2: Product Category Performance Matrix — Cross Join

**Scenario:**
The product team wants a complete matrix showing revenue for each product category in each month of 2025, including months where a category had zero sales. This helps visualize seasonal trends.

**Expected Output Columns:**
- `category_name` (varchar)
- `month` (integer) — 1-12
- `total_revenue` (numeric) — total revenue for that category in that month (0 if no sales)
- `order_count` (bigint) — number of orders containing that category (0 if none)

**Requirements:**
- Use `product_categories`, `products`, `orders_products`, `orders` tables
- Use CROSS JOIN to create all category × month combinations
- LEFT JOIN actual sales data
- Use COALESCE to show 0 for months with no sales
- Only include data from year 2025
- Order by `category_name` ASC, `month` ASC

**Difficulty Rating:** 4/5


WITH months_years AS (
SELECT
	DISTINCT(EXTRACT('Month' FROM date)) AS month_
	FROM dates
	WHERE EXTRACT('YEAR' FROM date) = 2025
	),
orders_months_join AS (
	SELECT
		my.month_,
		COALESCE(o.id, NULL) AS order_id,
		o.created_at AS order_date,
		COALESCE(op.quantity, 0) AS product_quantity,
		COALESCE(p.price, 0) AS product_price,
		COALESCE(pc.name, 'Empty') AS category_name
	FROM months_years my
	LEFT JOIN orders o ON my.month_= EXTRACT('Month' FROM o.created_at) AND EXTRACT('Year' FROM o.created_at) = 2025
	LEFT JOIN orders_products op ON o.id = op.order_id
	LEFT JOIN products p ON p.id = op.product_id
	LEFT JOIN product_categories pc ON pc.id = p.category_id
	ORDER BY month_
	)
SELECT 
	month_,
	category_name,
	SUM(product_quantity * product_price) AS total_revenue,
	COUNT(DISTINCT CASE WHEN order_id IS NOT NULL THEN order_id END) AS cnt_orders
FROM orders_months_join
GROUP BY month_, category_name
ORDER BY category_name, month_



---

## Task 3: Transaction Patterns — LEAD and LAG Together

**Scenario:**
The fraud detection team wants to identify unusual transaction patterns. For each transaction, show the previous and next transaction amounts for the same user, along with the time gaps.

**Expected Output Columns:**
- `user_id` (integer)
- `transaction_id` (integer) — current transaction id
- `amount` (numeric) — current transaction amount
- `prev_amount` (numeric) — previous transaction amount (NULL if first)
- `next_amount` (numeric) — next transaction amount (NULL if last)
- `time_since_prev` (interval) — time gap from previous transaction
- `time_until_next` (interval) — time gap to next transaction

**Requirements:**
- Use `transactions` table
- Apply both LAG() and LEAD() window functions partitioned by user_id
- Order transactions by created_at within each user
- Only include transactions from users with at least 3 transactions
- Order final output by `user_id` ASC, transaction `created_at` ASC

**Difficulty Rating:** 3/5

WITH users_transactions_amts_times AS (
	SELECT
		*,
		LAG(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_transaction_amt,
		LEAD(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS next_transaction_amt,
		LAG(created_at) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_transaction_time,
		LEAD(created_at) OVER (PARTITION BY user_id ORDER BY created_at) AS next_transaction_time
	FROM transactions
	),
users_transaction_cnts AS (
	SELECT 
		user_id,
		COUNT(id) AS cnt_transactions
	FROM transactions
	GROUP BY user_id
	)
SELECT 
	uta.user_id,
	uta.created_at,
	uta.prev_transaction_amt,
	uta.next_transaction_amt,
	uta.created_at - uta.prev_transaction_time AS time_since_prev,
	uta.next_transaction_time - uta.created_at AS time_until_next
	FROM users_transactions_amts_times uta
	JOIN users_transaction_cnts utc ON uta.user_id = utc.user_id
	WHERE utc.cnt_transactions >= 3
	ORDER BY uta.user_id, uta.created_at


---

## Task 4: Support Ticket Escalation Analysis

**Scenario:**
The customer support manager wants to identify tickets that required escalation (multiple status changes). Find tickets that changed status at least 3 times and calculate the average time between status changes.

**Expected Output Columns:**
- `ticket_id` (bigint)
- `user_id` (bigint)
- `priority` (text)
- `status_change_count` (bigint) — number of status changes
- `avg_time_between_changes` (interval) — average time between status changes
- `final_status` (text) — most recent status

**Requirements:**
- Use `chat_tickets` and `chat_messages` tables
- Filter for messages where `message_type = 'statuschange'`
- Count status changes per ticket
- Calculate time gaps between consecutive status changes using LAG()
- Only include tickets with `status_change_count >= 3`
- Order by `status_change_count` DESC, then `ticket_id` ASC

**Difficulty Rating:** 5/5

WITH statuses_changes AS (
	SELECT 
		*,
		LEAD(created_at) OVER (PARTITION BY ticket_id ORDER BY created_at) AS next_status_change_time,
		LAG(created_at) OVER (PARTITION BY ticket_id ORDER BY created_at) AS prev_status_change_time,
		FIRST_VALUE(status) OVER (PARTITION BY ticket_id ORDER BY created_at DESC) AS last_status
	FROM chat_messages
	WHERE status IS NOT NULL
	),
last_messages AS (
	SELECT 
		DISTINCT(ticket_id),
		FIRST_VALUE(message_text) OVER (PARTITION BY ticket_id ORDER BY created_at DESC) AS last_message
	FROM chat_messages cm 
	WHERE cm.message_text != 'status_change'
	),
filtered_tickets AS (
	SELECT 
		ticket_id,
		COUNT(*) AS cnt_status_change
	FROM statuses_changes
	GROUP BY ticket_id
	HAVING COUNT(*) >= 3
),
tickets_status_change_times AS (
	SELECT
		ticket_id,
		CASE 
			WHEN prev_status_change_time IS NULL THEN next_status_change_time - created_at
			WHEN next_status_change_time IS NULL THEN created_at - prev_status_change_time
			ELSE next_status_change_time - created_at
		END AS status_change_time
	 	FROM statuses_changes
	 	),
tickets_avg_changes AS (
	 SELECT 
		 ticket_id,
		 AVG(status_change_time) AS avg_chg_time
	 FROM tickets_status_change_times
	 GROUP BY ticket_id
	 )
SELECT 
	DISTINCT(ft.ticket_id),
	ct.user_id,
	ct.priority,
	ft.cnt_status_change,
	tac.avg_chg_time,
	sc.last_status,
	lm.last_message
FROM filtered_tickets ft 
JOIN chat_tickets ct ON ft.ticket_id = ct.id
JOIN tickets_avg_changes tac ON ft.ticket_id = tac.ticket_id
JOIN statuses_changes sc ON ft.ticket_id = sc.ticket_id
JOIN last_messages lm ON ft.ticket_id  = lm.ticket_id
ORDER BY ft.cnt_status_change DESC, ticket_id

I also added the last message.


---

## Task 5: Monthly Revenue Growth Percentage

**Scenario:**
The finance team needs a monthly revenue report showing month-over-month growth percentages. Calculate total order revenue for each month in 2025 and compare it to the previous month.

**Expected Output Columns:**
- `year` (integer)
- `month` (integer)
- `monthly_revenue` (numeric) — total revenue for the month
- `prev_month_revenue` (numeric) — previous month's revenue (NULL for January)
- `growth_percentage` (numeric) — ((current - prev) / prev) * 100

**Requirements:**
- Use `orders` table
- Extract year and month from created_at
- Aggregate revenue by month
- Use LAG() to get previous month's revenue
- Calculate growth percentage (handle division by zero with NULLIF)
- Only include months from 2025
- Order by `year` ASC, `month` ASC

**Difficulty Rating:** 3/5

WITH orders_monthly_rev AS (
	SELECT
		SUM(amount) AS monthly_revenue,
		EXTRACT('Month' FROM created_at) AS month_show,
		EXTRACT('Year' FROM created_at) AS year_show
	FROM orders
	WHERE EXTRACT('Year' FROM created_at) = 2025
	GROUP BY EXTRACT('Month' FROM created_at), EXTRACT('Year' FROM created_at)
	),
prev_revenues AS (
	SELECT
		month_show,
		year_show,
		monthly_revenue,
		LAG(monthly_revenue) OVER (ORDER BY month_show) AS prev_month_revenue
	FROM orders_monthly_rev
	)
SELECT 
	month_show,
	year_show,
	monthly_revenue,
	(monthly_revenue / NULLIF(prev_month_revenue, 0)) * 100 AS growth_percent
FROM prev_revenues
ORDER BY year_show, month_show

Quite easy, but useful tasks.


---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Logic correctness and syntax
- Efficiency and performance
- Best practices
- Alternative approaches

## Tips

- Remember to use `FIRST_VALUE(...ORDER BY ... DESC)` pattern for getting last values
- Pay attention to required output columns — include ALL specified columns
- Use CTEs to break down complex multi-step logic
- Consider NULL handling in calculations (COALESCE, NULLIF)
- Test LAG/LEAD with appropriate PARTITION BY and ORDER BY clauses

Good luck!

---

