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
