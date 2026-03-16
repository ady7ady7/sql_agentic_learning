# Daily SQL Practice Tasks

**Generated:** 2026-03-16
**Week 14, Day 1 Focus:** Self-Referencing CTE (Type B) + PIVOT Scaffolded (Step A) + Anti-Join NULL Edge Case

---

## Task 1: Self-Referencing Recursive CTE — User Referral Chain

**Scenario:**
The `users` table has a `referred_by` column... except it doesn't in our schema. So we'll use a self-contained CTE with inline data to make this runnable.

The following CTE provides the data — treat it as your source table called `referrals`:

```sql
WITH referrals (id, name, referred_by) AS (
    VALUES
    (1, 'Alice',   NULL),
    (2, 'Bob',     1),
    (3, 'Carol',   1),
    (4, 'Dave',    2),
    (5, 'Eve',     2),
    (6, 'Frank',   4),
    (7, 'Grace',   3)
)
```

Build a recursive CTE on top of this that traverses the referral chain to unlimited depth.

**Expected Output Columns:**
- `id` (integer)
- `name` (text)
- `referred_by` (integer)
- `path` (text) — e.g. `'Alice'`, `'Alice->Bob'`, `'Alice->Bob->Dave->Frank'`
- `depth` (integer) — 1 for root, 2 for direct referrals, etc.

**Requirements:**
- Anchor: start with the root (referred_by IS NULL)
- Recursive: JOIN referrals back to the CTE on referrals.referred_by = cte.id
- Build path by appending name at each step
- Track depth starting at 1
- No LEVEL + 1 termination needed — stops naturally
- Order by path ASC

**Difficulty Rating:** 3/5

WITH RECURSIVE referrals (id, name, referred_by) AS (
    VALUES
    (1, 'Alice',   NULL),
    (2, 'Bob',     1),
    (3, 'Carol',   1),
    (4, 'Dave',    2),
    (5, 'Eve',     2),
    (6, 'Frank',   4),
    (7, 'Grace',   3)
),
HIERARCHY AS (
SELECT
	1 AS id,
	'Alice' AS name,
	NULL::TEXT AS referred_by,
	'Alice' AS PATH,
	1 AS DEPTH
UNION ALL
SELECT
	r.id,
	r.name,
	h.name,
	h.PATH || ' < ' || r.name,
	h.DEPTH + 1
FROM HIERARCHY h
JOIN referrals r ON h.id = r.referred_by
)
SELECT * FROM hierarchy


Nice, but I'd like to implement similar logic of data in our database next time and actually use it instead. We can create a table and add relevant fields in users or wherever. I just need your guidelines and cooperation.




---

## Task 2: PIVOT — Scaffolded Introduction (Step A + Step B)

PIVOT is a new concept. This task walks you through it in two steps.

### Step A — Understand the unpivoted shape

Write a query that produces the raw unpivoted data we want to pivot:

```
month | type | transaction_count
```

- Use `transactions` table
- Group by `DATE_TRUNC('month', created_at)` and `type`
- Order by `month ASC`, `type ASC`

Run this and look at the output. Notice how each type is a separate row per month. **This is what we want to rotate into columns.**

---

### Step B — Write one pivot column manually

Now extend Step A into a pivot. Write a query that produces:

```
month | deposit_count
```

Just one column for now — `deposit_count` = number of transactions where `type = 'deposit'` per month.

Use this pattern:
```sql
COUNT(*) FILTER (WHERE type = 'deposit') AS deposit_count
```

or equivalently:
```sql
SUM(CASE WHEN type = 'deposit' THEN 1 ELSE 0 END) AS deposit_count
```

**Expected Output Columns:**
- `month` (date)
- `deposit_count` (bigint)

Order by `month ASC`.

**Note:** Step C (all 5 columns at once) comes tomorrow once this pattern is clear.

**Difficulty Rating:** 2/5

WITH transactions_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', t.created_at) AS month_
FROM crappy_data_db.transactions t
)
SELECT
	tm.month_,
	COUNT(*) FILTER (WHERE tm.type = 'deposit') AS deposit_count
FROM transactions_months tm
GROUP BY tm.month_
ORDER BY month_

Yeah, now it makes a lot of sense and I get the basic idea.

I was easily able to do all 5 columns now, once I get the basic pattern. This is great honestly and feels like a very useful concept to use that saves me the need to use GROUP BY type. Wondering, how memory efficient is this, as it looks awesome. Definitely want to practice and learn this pattern in more advanced scenarios etc.

WITH transactions_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', t.created_at) AS month_
FROM crappy_data_db.transactions t
)
SELECT
	tm.month_,
	COUNT(*) FILTER (WHERE tm.type = 'deposit') AS deposit_count,
	COUNT(*) FILTER (WHERE tm.type = 'transfer') AS transfer_count,
	COUNT(*) FILTER (WHERE tm.type = 'withdrawal') AS withdrawal_count,
	COUNT(*) FILTER (WHERE tm.type = 'purchase') AS purchase_count,
	COUNT(*) FILTER (WHERE tm.type = 'payment') AS payment_count
FROM transactions_months tm
GROUP BY tm.month_
ORDER BY month_


---

## Task 3: Anti-Join — The NULL Trap in NOT IN

**Scenario:**
Yesterday you wrote three anti-join approaches. Today we explore when one of them silently breaks.

The `orders` table has a `user_id` column that is `NOT NULL` — so `NOT IN` works correctly there. But what if `user_id` could be NULL?

**Part A:** Write this query and run it:
```sql
SELECT id FROM users
WHERE id NOT IN (SELECT user_id FROM orders WHERE user_id IS NULL OR user_id IS NOT NULL)
```
What do you expect it to return? What does it actually return? Write your observation as a comment.

In this case we'd get data without any issues, so we'd get all users who did not make any orders, as I'm 100% sure that by design there are NO NULL id's in orders.
Also, I'd expect that we'd only have IS NOT NULL condition, I don't see the point of having IS NULL or IS NOT NULL, it's basically useless.


**Part B:** Now fix Part A using `NOT EXISTS` instead, so it correctly returns users not in orders regardless of NULLs.

SELECT id FROM crappy_data_db.users u
WHERE NOT EXISTS 
(SELECT 
	* 
FROM crappy_data_db.orders o
WHERE o.user_id = u.id)

**Part C:** Fix it again using `LEFT JOIN ... WHERE IS NULL`.

**Expected insight:** `NOT IN` returns **zero rows** when the subquery contains even one NULL — because `x NOT IN (..., NULL, ...)` evaluates to UNKNOWN, not TRUE, for every row. `NOT EXISTS` and `LEFT JOIN` are NULL-safe.

**Difficulty Rating:** 3/5

SELECT 
	u.id
FROM crappy_data_db.users u
LEFT JOIN crappy_data_db.orders o ON u.id = o.user_id
WHERE o.user_id IS NULL

This pattern feels quite unnatural to use at this point though.


---

## Submission Instructions

1. Task 1 — Self-referencing referral chain (3/5)
2. Task 2 — PIVOT Step A + Step B (2/5)
3. Task 3 — Anti-join NULL trap (3/5)
