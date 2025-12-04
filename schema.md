# schema.md

## Overview
Our database (local Postgres) is a synthetic dataset designed for practicing SQL queries (JOINs, aggregations, windows, subqueries, customer/orders/tickets analytics).
The data DOES NOT COME from a real production system; it contains random people, orders, transactions, products, user sessions, tickets, and chat messages.

You will not have direct access to my database itself, but below you can see the schema + some examples, so you are aware of what we actually have in the database.
---

## Table: users

System users (demographic data + activity). [file:21]  

Columns:
- id (serial, PK, not null) – user identifier, e.g., 1  
- created_at (timestamp, nullable, default now) – registration date, e.g., `2024-10-23 06:23:16`  
- first_name (varchar(100), nullable) – first name, e.g., `Magdalena`  
- last_name (varchar(100), nullable) – last name, e.g., `Pirek`  
- age (integer, nullable) – age, sometimes NULL, e.g., 57, NULL  
- gender (varchar(10), nullable) – `Male` / `Female` (text values only, no constraint), e.g., `Female`  
- email (varchar(150), nullable) – email address, e.g., `magdalena.pirek63@gmail.com`  
- city (varchar(100), nullable) – city, e.g., `Warsaw`, `Berlin`, or NULL  
- country (varchar(100), nullable) – country, e.g., `Poland`, `Germany`, or NULL  
- is_active (boolean, nullable) – if user is active, e.g., `true`, `false`, or NULL  

Data includes NULLs primarily in age, city, country, and sporadically in other columns. [file:21]  

---

## Table: user_sessions_daily

Daily aggregated user sessions. [file:21]  

Columns:
- id (serial, PK, not null) – record ID, e.g., 1  
- user_id (integer, not null, FK → users.id) – user ID, e.g., 9  
- date (date, not null) – day, e.g., `2025-07-02`  
- count_sessions (integer, not null) – number of sessions per day, e.g., 0–10+ (including 0)  

No NULLs in fields. [file:21]  

---

## Table: transactions

Financial transactions linked to users (various transaction types). [file:21]  

Columns:
- id (serial, PK, not null) – transaction ID, e.g., 1  
- created_at (timestamp, nullable, default now) – transaction datetime, e.g., `2024-08-21 07:30:26`  
- user_id (integer, nullable, FK → users.id) – user; mostly filled but can be NULL  
- amount (numeric(10,2), nullable) – amount, e.g., 150.82  
- type (text, nullable) – transaction type, including:
  - `withdrawal`  
  - `payment`  
  - `transfer`  
  - `deposit`  
  - `purchase`  

Nullable columns include created_at, user_id, amount, type, though data mostly is present. [file:21]  

---

## Table: orders

User orders (order total amount and user link). [file:21]  

Columns:
- id (serial, PK, not null) – order ID, e.g., 1  
- created_at (timestamp, not null, default current_timestamp) – order creation timestamp, e.g., `2025-05-30 12:02:07`  
- user_id (integer, not null, FK → users.id) – user ID, e.g., 99  
- amount (double precision, nullable) – order amount (total), e.g., 588.83  

amount may be NULL, though mostly set in data. [file:21]  

---

## Table: product_categories

Product categories. [file:21]  

Columns:
- id (serial, PK, not null) – category ID, e.g., 1  
- name (varchar(50), not null, unique) – category name, e.g., `travel`, `hobbies`, `office supplies`  
- description (text, nullable) – category description or NULL  

Description may be NULL, names are unique. [file:21]  

---

## Table: products

Products in the system. [file:21]  

Columns:
- id (integer, PK, not null) – product ID, e.g., 1  
- name (varchar(100), nullable) – product name, e.g., `Luggage`, `Backpack`, `Packing Cubes` etc.  
- price (numeric(10,2), nullable) – unit price, e.g., 89.99  
- category_id (integer, nullable, FK → product_categories.id) – product category, mostly present  

Name, price, and category_id can be NULL but mostly set. [file:21]  

---

## Table: orders_products

Order items (many-to-many relation orders-products). [file:21]  

Columns:
- id (serial, PK, not null) – line item ID, e.g., 1  
- order_id (integer, not null, FK → orders.id) – order reference  
- product_id (integer, not null, FK → products.id) – product reference  
- quantity (integer, not null, CHECK quantity > 0) – quantity purchased, e.g., 1, 2, 5  

No NULLs; quantity always positive due to constraint. [file:21]  

---

## Table: deliveries

Order deliveries (delivery status). [file:21]  

Columns:
- id (serial, PK, not null) – delivery ID  
- created_at (timestamp, not null, default now) – delivery creation datetime  
- order_id (integer, not null, FK → orders.id ON DELETE CASCADE) – linked order; deleting order deletes delivery  
- status (text, not null) – delivery status, e.g., `pending`, `delivered` (no constraint on values)  

No NULLs in key columns. [file:21]  

---

## Table: dates

Date dimension table (calendar). [file:21]  

Columns:
- date (date, PK, not null) – continuous calendar date  

Used for calendar joins even when no data exists for other tables on certain dates. No NULLs. [file:21]  

---

## Table: chat_tickets

Chat support tickets. [file:21]  

Columns:
- id (bigint, identity, PK, not null) – ticket ID, e.g., 1  
- user_id (bigint, not null) – user who opened ticket (not FK linked but logically linked)  
- status (text, not null) – ticket status, e.g., `pending`, `assigned`, `resolved`, `closed`, etc.  
- priority (text, not null) – priority level, e.g., `low`, `medium`, `high`, `urgent`  
- created_at (timestamp with time zone, not null, default now) – creation date/time  
- updated_at (timestamp with time zone, not null, default now) – last update date/time  

No NULL values present in these fields. [file:21]  

---

## Table: chat_messages

Messages within tickets (conversation log). [file:21]  

Columns:
- id (bigint, identity, PK, not null) – message ID  
- ticket_id (bigint, not null, FK → chat_tickets.id ON DELETE CASCADE) – linked ticket  
- user_id (bigint, nullable) – sender user ID (NULL possible for system/agent messages)  
- author_id (bigint, nullable) – author ID (agent or system), NULL if from client  
- message_type (text, not null) – message type, e.g., `text`, `statuschange`  
- message_text (text, nullable) – message content text, NULL for status changes  
- status (text, nullable) – new status (for `statuschange`), e.g., `open`, `closed`, NULL otherwise  
- created_at (timestamp with time zone, not null, default now) – timestamp of message  

NULLs possible in user_id, author_id, message_text, and status, depending on message type. [file:21]  

---

## View: active_users

View filtering only active users. [file:21]  

- Selects all columns from users (id, created_at, first_name, last_name, age, gender, email, city, country, is_active)  
- WHERE is_active = TRUE, so excludes NULLs and false users  

No computation, simply filtering active flag. [file:21]  

---

## Table: active_users_table

Materialized copy of active_users structure (no PK, no constraints). [file:21]  

Columns permit NULLs in all fields, allowing experimental data loading. [file:21]  

---

## Indexes and constraints overview

- Primary keys (PK): on id columns of all tables (except active_users_table has no PK), dates on "date"  
- Foreign keys (FK): user_sessions_daily.user_id → users.id, transactions.user_id → users.id, orders.user_id → users.id, products.category_id → product_categories.id, orders_products links orders and products, deliveries.order_id → orders.id (cascade delete), chat_messages.ticket_id → chat_tickets.id (cascade delete)  
- Unique: product_categories.name  
- Check constraint: orders_products.quantity > 0  
- Additional index: idx_users_city on users(city)  

[file:21]  
