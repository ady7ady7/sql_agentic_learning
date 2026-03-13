# Daily SQL Practice Tasks

**Generated:** 2026-03-12
**Week 13, Day 4 Focus:** Self-Referencing Recursive CTE + PIVOT Introduction + Anti-Join Patterns

---

## Task 1: Self-Referencing Recursive CTE — Employee Org Chart

**Scenario:**
You have an `employee` table with the following structure:

```
id | first_name | last_name | manager_id
1  | Madeline   | Ray       | NULL
2  | Violet     | Green     | 1
3  | Alton      | Vasquez   | 1
4  | Geoffrey   | Delgado   | 1
5  | Allen      | Garcia    | 2
6  | Marian     | Daniels   | 2
7  | Tricia     | Wong      | 3
8  | Bruce      | Grant     | 3
9  | Darin      | Burke     | 4
10 | Bob        | Freeman   | 5
```

Build a recursive CTE that traverses this hierarchy to unlimited depth. For each employee, show their full reporting path from the root.

**Expected Output Columns:**
- `id` (integer)
- `first_name` (text)
- `last_name` (text)
- `manager_id` (integer)
- `path` (text) — e.g. `'Ray'`, `'Ray->Green'`, `'Ray->Green->Garcia->Freeman'`

**Requirements:**
- Anchor: start with the root employee (manager_id IS NULL)
- Recursive part: JOIN employee back to itself on employee.manager_id = cte.id
- Path: built by appending last_name at each level
- No LEVEL + 1 pattern needed — termination happens naturally when no more children exist
- Order by path ASC

**Note:** This uses a real table called `employee` — it is NOT in schema.md as it's a standalone exercise table provided above. Write the query as if this table exists.

**Difficulty Rating:** 3/5

Dude. The problem is that WE DON'T HAVE SUCH A structure in our database. What are you expecting of me? Rejecting this task, don't count it for today, and next time please prepare.

---

## Task 2: PIVOT — Monthly Transaction Counts by Type

**Scenario:**
The finance team wants a pivoted monthly report showing how many transactions occurred per type, with each type as its own column.

Transform this row-based result:
```
month       | type        | count
2024-08-01  | deposit     | 45
2024-08-01  | withdrawal  | 32
2024-08-01  | payment     | 28
...
```

Into this pivoted shape:
```
month       | deposit | withdrawal | payment | transfer | purchase
2024-08-01  | 45      | 32         | 28      | ...      | ...
```

**Expected Output Columns:**
- `month` (date) — truncated to month
- `deposit` (bigint)
- `withdrawal` (bigint)
- `payment` (bigint)
- `transfer` (bigint)
- `purchase` (bigint)

**Requirements:**
- Use `transactions` table
- Use conditional aggregation: `COUNT(*) FILTER (WHERE type = 'deposit')` or `SUM(CASE WHEN type = 'deposit' THEN 1 ELSE 0 END)`
- One CTE to aggregate, final SELECT to present the pivot
- Order by `month ASC`

**Difficulty Rating:** 4/5

You see, I'm NOT AWARE how to actually create such a pivot yet - YOU have to create a scaffolded approach and teach me how to do it, as it's a new concept for me. Make sure you do it properly next time! Do not take away points from me, treat this as valuable feedback.

---

## Task 3: Anti-Join — Users Who Never Placed an Order

**Scenario:**
The marketing team wants to target users who have registered but never placed a single order. Find all such users.

Solve this three ways in the same file — one query per approach:

**Approach A:** Using `NOT IN`
**Approach B:** Using `NOT EXISTS`
**Approach C:** Using `LEFT JOIN ... WHERE IS NULL`

**Expected Output Columns** (same for all three):
- `user_id` (integer)
- `created_at` (timestamp) — user registration date

**Requirements:**
- Use `users` and `orders` tables
- Order by `created_at ASC`
- After writing all three, add a short comment on which approach you'd prefer and why

**Difficulty Rating:** 3/5
1.
SELECT 
	u.id AS user_id,
	u.created_at
FROM crappy_data_db.users U
WHERE u.id NOT IN (SELECT o.user_id FROM crappy_data_db.orders o)

2.
SELECT 
	u.id AS user_id,
	u.created_at
FROM crappy_data_db.users U
WHERE NOT EXISTS (
SELECT *
FROM crappy_data_db.orders o
WHERE o.user_id = u.id
)

This is definitely weird, it feels unnatural as it requires way more code and it's not that logical. Not sure, why would I pick this option though.

3. 

SELECT 
	u.id AS user_id,
	u.created_at
FROM crappy_data_db.users u
LEFT JOIN crappy_data_db.orders o ON u.id = o.user_id
WHERE o.user_id IS NULL
ORDER BY o.user_id

Not bad.


---

## Submission Instructions

1. Task 1 — Self-referencing org chart (3/5)
2. Task 2 — PIVOT monthly transactions by type (4/5)
3. Task 3 — Anti-join three ways (3/5)
