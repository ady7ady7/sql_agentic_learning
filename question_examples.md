Below you can find examples of good questions that allow to explore data and practice writing SQL queries.
Each question also has a POTENTIAL ANSWER below, where the answer is not an actual direct answer, but a query used to find an answer when ran in the database.



1. How many users had the amount of their first order lower than the total amount of all their subsequent orders?

POTENTIAL ANSWER:

WITH users_orders_amounts AS (
	SELECT 
	user_id,
	id,
	amount,
	FIRST_VALUE(amount) OVER (PARTITION BY user_id ORDER BY created_at) as first_order_amount
	FROM orders
	),
users_orders_comparison AS (
SELECT 
user_id,
amount,
first_order_amount,
CASE 
	WHEN first_order_amount <= amount THEN 1 ELSE 0
	END AS higher_than_first_order,
COUNT(amount) OVER (PARTITION BY user_id) as total_orders
FROM users_orders_amounts
),
counted_orders AS (
	SELECT user_id,
	SUM(higher_than_first_order) as orders_higher_than_first,
	total_orders
	FROM users_orders_comparison
	WHERE total_orders > 1
	GROUP BY user_id, total_orders
	ORDER BY user_id
	)

SELECT user_id
FROM counted_orders
WHERE orders_higher_than_first = total_orders


2. What is the average time it takes for an order status to change in the deliveries table?

POTENTIAL ANSWER:

WITH status_change_times AS (
	SELECT 
	order_id,
	status,
	created_at,
	LAG(created_at) OVER (PARTITION BY order_id ORDER BY created_at) AS previous_date
	FROM deliveries
	ORDER BY order_id, created_at
	),
calculated_change_times AS (
	SELECT 
	order_id,
	status,
	created_at,
	created_at - previous_date AS status_change_time
	FROM status_change_times
	WHERE previous_date IS NOT NULL
	)
SELECT AVG(status_change_time)
FROM calculated_change_times


3. What is the average waiting time for the first response in a ticket?

POTENTIAL ANSWER:


WITH first_users_messages AS (
	SELECT 
	DISTINCT(ticket_id),
	FIRST_VALUE(created_at) OVER (PARTITION BY ticket_id) as first_users_msg
	from chat_messages
	ORDER BY ticket_id
	),
employees_responses AS (
	SELECT 
	DISTINCT(ticket_id),
	FIRST_VALUE(created_at) OVER (PARTITION BY ticket_id) as first_employees_msg
	FROM chat_messages
	WHERE message_type = 'text'
	AND author_id IS NOT NULL
	ORDER BY ticket_id
	),
response_times AS (
	SELECT 
	fum.ticket_id, 
	fum.first_users_msg,
	er.first_employees_msg,
	er.first_employees_msg - fum.first_users_msg AS response_time
	FROM first_users_messages fum
	JOIN employees_responses er
	ON fum.ticket_id = er.ticket_id
	)
SELECT AVG(response_time) from response_times

4. How many products were sold on the day when the order revenue was the highest?

POTENTIAL ANSWER:

SELECT
    DATE(created_at) as day_date,
    SUM(quantity) as total_products_sold,
    SUM(quantity * price) as total_revenue
FROM orders_products op
JOIN orders o ON op.order_id = o.id
JOIN products p ON op.product_id = p.id
JOIN product_categories pc ON p.category_id = pc.id
WHERE EXTRACT(YEAR from o.created_at) = 2025
GROUP BY DATE(created_at)
ORDER BY total_revenue DESC

5. Are there any tickets that were handled by more than one employee?

POTENTIAL ANSWER:

SELECT
    ticket_id,
	COUNT(author_id) as cnt_employees
FROM chat_messages
GROUP BY ticket_id

6. What is the average number of messages per ticket?

POTENTIAL ANSWER:

WITH tickets_messages AS (
	SELECT ticket_id,
	COUNT(*) as cnt_messages
	FROM chat_messages
	WHERE message_text IS NOT NULL
	GROUP BY ticket_id
	)

SELECT AVG(cnt_messages) FROM tickets_messages

7. Who had, on average, more products (quantity) per order — women or men?

POTENTIAL ANSWER:

WITH quantity_per_order AS (
SELECT op.order_id,
    u.gender,
    SUM(op.quantity) AS sum_quantity
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN orders_products op ON o.id = op.order_id
GROUP BY op.order_id, u.gender
)
SELECT
    gender,
    AVG(sum_quantity) AS avg_quantity
FROM quantity_per_order
GROUP BY gender
ORDER BY avg_quantity DESC

8. Who bought more products in total — women or men?

POTENTIAL ANSWER:

SELECT 
    u.gender,
    SUM(op.quantity) AS sum_of_products_bought
FROM orders_products op
JOIN orders o ON op.order_id = o.id
JOIN users u ON o.user_id = u.id
GROUP BY u.gender


9. Are there months where the total amount spent on orders in that month was greater than the sum of the amounts from the two previous months?

POTENTIAL ANSWER:

WITH orders_months_yrs AS (
	SELECT 
        EXTRACT('Month' FROM created_at) as month,
        EXTRACT('Year' FROM created_at) as year,
        sum(amount) as total_amount
	FROM orders
	GROUP BY EXTRACT('Year' FROM created_at), EXTRACT('Month' FROM created_at)
	ORDER BY year
	),
comparison_window AS (
	SELECT 
        month, 
        year, 
        total_amount,
        SUM(total_amount) OVER (ROWS BETWEEN 2 PRECEDING AND 1 PRECEDING) as sum_of_2_previous_months
	FROM orders_months_yrs
	)

SELECT 
    month, 
    year, 
    total_amount, 
    sum_of_2_previous_months,
    total_amount > sum_of_2_previous_months as higher_than_2_previous_months
FROM comparison_window 

10. How many users who are in the top 30 users with the highest total transaction amounts are also in the top 30 users with the highest total order amounts?

POTENTIAL ANSWER:

WITH top_transactionists AS (
	SELECT user_id, SUM(amount) as total_amount,
	RANK() OVER (ORDER BY SUM(amount) DESC)
	FROM transactions
	GROUP BY user_id
	ORDER BY total_amount DESC
	),
top_orderers AS (
	SELECT user_id, SUM(amount) as total_amount,
	RANK() OVER (ORDER BY SUM(amount) DESC)
	FROM orders
	GROUP BY user_id
	ORDER BY total_amount DESC
	)

SELECT user_id FROM top_transactionists
WHERE rank <= 30
INTERSECT
SELECT user_id FROM top_orderers
WHERE rank <= 30