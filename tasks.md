# SQL Tasks — 2026-08-03 (Week 33, Day 1)

**Dataset:** orders / transactions / users  
**Focus:** LAG with custom offset · Dominant type per user · Query optimization

---

## Task 1 — Transaction Momentum: Compare to N Steps Back
**Difficulty: 3/5**

**Business question:**  
For each user, look at their transactions in chronological order. For every transaction, show:
- The current `amount`
- The amount from **3 transactions ago** (same user, by `created_at`)
- The **difference** between current and 3-ago (positive = grew, negative = shrank)
- Only include rows where the 3-ago value actually exists (no NULLs)

Order results by `user_id`, `created_at`.

**Expected output columns:**  
`user_id, created_at, amount, amount_3_ago, diff`

**Difficulty: 3/5**


WITH first_agg AS (
SELECT
	user_id,
	created_at,
	amount,
	LAG(amount, 3) OVER (PARTITION BY user_id ORDER BY t.created_at) AS amount_3_ago,
	amount - LAG(amount, 3) OVER (PARTITION BY user_id ORDER BY t.created_at) AS diff
FROM crappy_data_db.transactions t 
)
SELECT * FROM first_agg
WHERE amount_3_ago IS NOT NULL
ORDER BY user_id, created_at

---

## Task 2 — Dominant Transaction Type Per User
**Difficulty: 4/5**

**Business question:**  
For each user, find their **dominant transaction type** — the type they use most often. If there's a tie, return all tied types.

Then, at the end, show a summary: how many users have each type as their dominant (or co-dominant) type?

Two-part output:

**Part A:** Per user  
`user_id, type, tx_count, type_rank`  
(only rows where `type_rank = 1`)

**Part B:** Summary  
`type, user_count`  
(count of users for whom this type is dominant)

Order Part B by `user_count DESC`.

**Difficulty: 4/5**

WITH users_transaction_cnts AS (
SELECT 
	user_id,
	TYPE,
	COUNT(*) AS tx_count
FROM crappy_data_db.transactions t
GROUP BY user_id, TYPE
),
tx_ranks AS (
SELECT 
	*,
	row_number() OVER (PARTITION BY user_id ORDER BY tx_count DESC) AS type_rank
FROM users_transaction_cnts 
),
users_top_tx_types AS (
SELECT
	user_id,
	TYPE,
	tx_count,
	type_rank
FROM tx_ranks
WHERE type_rank = 1
)
SELECT 
	TYPE,
	COUNT(*) AS user_count
FROM users_top_tx_types
GROUP BY TYPE
ORDER BY user_count DESC


EASY, you have both part A and B, for part A just consider there's no last CTE, just SELECT * FROM users_top_tx_types there instead :)).



---

## Task 3 — Query Optimization: Anti-Join + NULL Trap
**Difficulty: 5/5**

**Business question:**  
Find all users who have **never placed an order**. Use `NOT EXISTS`.

Then, as a comment above your query, explain in 2–3 zdaniach:
- Dlaczego `NOT IN` jest niebezpieczny gdy podzbior może zawierać NULLe?
- Co konkretnie się dzieje (jakie wiersze zwraca, jaki wynik)?

Null w przypadku not ina spierdala wynik i pokazuje de facto pusty zbiór w takiej sytuacji.

WITH uids_with_no_orders AS (
SELECT 
	u.id AS user_id
FROM crappy_data_db.users u
WHERE NOT EXISTS (
	SELECT o.user_id FROM crappy_data_db.orders o
	WHERE o.user_id = u.id
)
)
SELECT * FROM uids_with_no_orders


Następnie: rozszerz query — znajdź użytkowników którzy nigdy nie złożyli zamówienia **i** mają co najmniej jedną transakcję typu `deposit`. Pokaż `user_id` i łączną sumę ich depozytów (`total_deposits`), posortowane malejąco po `total_deposits`.

**Expected output columns:**  
`user_id, total_deposits`

**Difficulty: 5/5**


WITH uids_with_no_orders AS (
SELECT 
	u.id AS user_id
FROM crappy_data_db.users u
WHERE NOT EXISTS (
	SELECT o.user_id FROM crappy_data_db.orders o
	WHERE o.user_id = u.id
)
)
SELECT 
	u.user_id,
	SUM(t.amount) AS total_deposits
FROM uids_with_no_orders u
JOIN crappy_data_db.transactions t ON u.user_id = t.user_id AND t."type" = 'deposit'
GROUP BY u.user_id
ORDER BY total_deposits DESC

Żaden problem.


---

## Submission Instructions

Paste your queries below each task. For Task 3, include the comments — they're part of the answer.
