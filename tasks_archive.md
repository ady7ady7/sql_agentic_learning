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

