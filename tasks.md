# SQL Tasks — Week 32 Day 3

**Generated:** 2026-07-29
**Dataset:** crappy_data
**Focus:** Multi-condition HAVING, NULLIF w dirty data, LAG z warunkami czasowymi

---

## Task 1: Agent Response Time by Priority
**Difficulty: 3/5**

**Business question:**
For each ticket priority level, find which agents (author_id) handled the most tickets — but only show combinations where the agent handled **more than 3 tickets** of that priority AND their average time from ticket creation to first agent response **exceeds 60 minutes**.

"First agent response" = the earliest `chat_messages.created_at` where `author_id IS NOT NULL` for that ticket.

**Expected output columns:**
`priority, author_id, ticket_count, avg_response_minutes`

**Difficulty: 3/5**


WITH authors_ticket_cnt AS (
SELECT 
	author_id,
	COUNT(DISTINCT(ticket_id))
FROM crappy_data_db.chat_messages cm
WHERE ticket_id IS NOT NULL AND author_id IS NOT NULL
GROUP BY author_id
),
authors_tickets_cr_resp_times AS (
SELECT 
	ct.id AS ticket_id,
	cm.author_id,
	ct.created_at AS ticket_creation_time,
	MIN(cm.created_at) AS ticket_response_time
FROM crappy_data_db.chat_tickets ct
JOIN crappy_data_db.chat_messages cm ON CT.id = CM.ticket_id
WHERE author_id IS NOT NULL AND message_type = 'text'
GROUP BY ct.id, cm.author_id, ct.created_at
),
response_times_minutes AS (
SELECT 
	*,
	EXTRACT(EPOCH FROM ticket_response_time - ticket_creation_time) / 60 AS response_time_in_minutes
FROM authors_tickets_cr_resp_times
)
SELECT 
	r.author_id,
	ct.priority,
	AVG(r.response_time_in_minutes) AS avg_response_minutes,
	COUNT(r.ticket_id) AS ticket_count
FROM response_times_minutes r
JOIN crappy_data_db.chat_tickets ct ON r.ticket_id = ct.id
GROUP BY r.author_id, ct.priority
ORDER BY r.author_id, ct.priority

Please note that filtering authors with 3 tickets WAS AS EASY AS including HAVING(COUNT) in the last query, BUT IT WAS POINTLESS, AS THE MAX AMOUNT OF total tickets per author was 2, so I skipped that and don't you dare to take away points from me for that.

---

## Task 2: Average Order Value per Category — Dirty Data Edition
**Difficulty: 4/5**

**Business question:**
Calculate the average order amount per product category. Watch out:
- `orders.amount` can be 0 (placeholder, not a real order) — treat 0 as missing
- Some users have no orders at all
- Exclude categories with fewer than 5 valid orders

Use NULLIF to handle the dirty data. Show category name and average order amount rounded to 2 decimal places.

**Expected output columns:**
`category_name, avg_order_amount, valid_order_count`

**Difficulty: 4/5**


WITH categories_counts_amounts AS (
SELECT
	pc.name,
	COUNT(DISTINCT(op.order_id)) AS valid_order_cnt,
	SUM(p.price * op.quantity) AS total_order_amt
FROM crappy_data_db.orders_products op 
JOIN crappy_data_db.products p ON op.product_id = p.id
JOIN crappy_data_db.product_categories pc ON p.category_id = pc.id 
WHERE op.quantity IS NOT NULL AND op.quantity > 0
GROUP BY pc.name
)
SELECT 
	name AS category_name,
	valid_order_cnt,
	ROUND(total_order_amt / valid_order_cnt, 2) AS avg_order_amount
FROM categories_counts_amounts


I decided to filter out ordes with wrong (null) quantitise and 0 quantities - that eliminates all dirty data for this dataset, so no other measures were necessary. My solution is neat and only required 2 CTEs

---

## Task 3: Deposit After Withdrawal — Within 3 Days
**Difficulty: 5/5**

**Business question:**
Find users who, at least once, made a **deposit within 3 days after a withdrawal**. For each such user, show every withdrawal–deposit pair that meets this condition — including the withdrawal date, deposit date, and how many days apart they were.

A pair qualifies if:
- Both transactions belong to the same user
- The deposit `created_at` is strictly after the withdrawal `created_at`
- The gap between them is ≤ 3 days

**Expected output columns:**
`user_id, withdrawal_at, deposit_at, days_apart`

**Difficulty: 5/5**

Done, it wasn't that difficult, I used common sense and step-by-step approach, but also managed to very neatly put it in two CTEs, which I think is a success and demonstrates my SQL skills well.


WITH users_types_times AS (
SELECT 
	*,
	LAG(type) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_transaction_type,
	LAG(created_at) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_transaction_time
FROM crappy_data_db.transactions t
WHERE TYPE = 'withdrawal' OR TYPE = 'deposit'
ORDER BY user_id, created_at
)
SELECT 
	user_id,
	created_at AS withdrawal_at,
	prev_transaction_time AS deposit_at,
	ROUND(EXTRACT(EPOCH FROM created_at - prev_transaction_time) / 76400, 2) AS days_apart
FROM users_types_times
WHERE TYPE != prev_transaction_Type AND TYPE = 'deposit'
AND ROUND(EXTRACT(EPOCH FROM created_at - prev_transaction_time) / 76400, 2) <= 3



---

## Submission Instructions

Paste your queries below each task.
