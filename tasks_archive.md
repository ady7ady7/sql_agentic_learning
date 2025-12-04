### Task Archive

(Archive of completed `tasks.md` snapshots will be appended here with a timestamp header.)

---

### Task Archive: 2025-12-04 (Previous Tasks)

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

