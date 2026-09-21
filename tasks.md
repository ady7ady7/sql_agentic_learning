# SQL Tasks — 2026-09-21 (Week 39, Day 1)

**Dataset:** crappy_data_db · job_db  
**Focus:** DISTINCT ON — three levels of difficulty, easiest to hardest

---

## Task 1 — Top Spender per City (DISTINCT ON)

**Difficulty: 4/5**

**Business question:**  
For each city, find the single user with the highest total transaction amount (one total per user, same aggregate-first shape as before). Use `DISTINCT ON (city)`.

Only include cities with at least 3 users who have transactions.

**Expected output columns:**  
`city, user_id, total_amount`

Order by `city`.


WITH users_transactions AS (
SELECT 
	user_id,
	SUM(amount) AS total_amount
FROM crappy_data_db.transactions t
GROUP BY user_id
)
SELECT DISTINCT ON (u.city)
	u.city,
	t.user_id,
	t.total_amount
FROM crappy_data_db.users u
JOIN users_transactions t ON u.id = t.user_id
WHERE u.city IS NOT NULL
ORDER BY city, total_amount DESC




---

## Task 2 — Most Recent Job Offer per City (DISTINCT ON)

**Difficulty: 3/5**

**Business question:**  
For each city in `job_db.oferty`, find the most recently listed offer (by `data_wystawienia`).

Only include rows where `data_wystawienia` IS NOT NULL and `miasto` IS NOT NULL.

**Expected output columns:**  
`miasto, pozycja, data_wystawienia`

Order by `miasto`.


SELECT DISTINCT ON (miasto)
	miasto,
	pozycja,
	data_wystawienia
FROM job_db.oferty o
WHERE data_wystawienia IS NOT NULL AND miasto IS NOT NULL
ORDER BY miasto, data_wystawienia

---

## Task 3 — Last Transaction Before Each Order (DISTINCT ON, Cross-Table)

**Difficulty: 5/5**

**Business question:**  
For each order, find that user's most recent transaction that happened strictly BEFORE the order's `created_at`. Use `DISTINCT ON (order_id)`.

**Why this is harder than Task 1/2:** the DISTINCT ON key (`order_id`) comes from one table, but the row you're selecting FROM and the tiebreaker column (`transaction.created_at`) come from a different table, joined with a condition that isn't just equality (`transaction.created_at < order.created_at`). You need the JOIN to happen first, producing (order, transaction) candidate pairs, and only then apply DISTINCT ON to pick the closest-preceding one per order.

**Suggested shape:**
```sql
SELECT DISTINCT ON (o.id)
    o.id AS order_id, o.created_at AS order_time,
    t.id AS transaction_id, t.created_at AS transaction_time, t.amount
FROM crappy_data_db.orders o
JOIN crappy_data_db.transactions t
    ON t.user_id = o.user_id AND t.created_at < o.created_at
ORDER BY o.id, t.created_at DESC
```

Orders with no prior transaction simply won't appear (JOIN excludes them) — that's expected, not a bug.

**Expected output columns:**  
`order_id, order_time, transaction_id, transaction_time, amount`

Order by `order_id`.


SELECT DISTINCT ON (o.id)
	o.id AS order_id,
	t.id AS transaction_id,
	o.created_at AS order_time,
	t.created_at AS transaction_time,
	t.amount AS transaction_amount,
	o.user_id AS user_id
FROM crappy_data_db.orders o
JOIN crappy_data_db.transactions t 
	ON o.user_id = t.user_id
	AND o.created_at > t.created_at
ORDER BY o.id, t.created_at DESC



---

## Submission Instructions

Paste your queries below each task.
