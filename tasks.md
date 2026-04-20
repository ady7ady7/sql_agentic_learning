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


SELECT 
	u.id AS user_id
FROM crappy_data_db.users u
WHERE NOT EXISTS
(SELECT o.user_id FROM crappy_data_db.orders o
WHERE o.user_id = u.id AND o.user_id IS NOT NULL
)




---

## Task 2: Conditional Aggregation — Transaction Type Breakdown per User

**Scenario:**
The finance team wants a per-user summary of transaction activity, broken down by type — but as columns, not rows.

For each user who has at least 1 transaction, show:
- `user_id`
- `total_transactions` — total count of all transactions
- `deposit_count` — number of `deposit` transactions
- `withdrawal_count` — number of `withdrawal` transactions
- `purchase_count` — number of `purchase` transactions

**Tables:** `transactions`

**Requirements:**
- Use conditional aggregation (`CASE WHEN` inside `COUNT`) for the per-type counts

**Difficulty Rating:** 4/5

WITH users_transactions_breakdown AS (
SELECT 
	user_id,
	COUNT(id) AS total_transactions,
	COUNT(id) FILTER (WHERE TYPE = 'deposit') AS deposit_count,
	COUNT(id) FILTER (WHERE TYPE = 'withdrawal') AS withdrawal_count,
	COUNT(id) FILTER (WHERE TYPE = 'purchase') AS purchase_count
FROM crappy_data_db.transactions t
GROUP BY user_id


---

## Submission Instructions

1. Task 1 — Anti-join approach + NULL trap explanation (3/5)
2. Task 2 — Per-user transaction type breakdown (4/5)
