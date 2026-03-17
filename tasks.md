# Daily SQL Practice Tasks

**Generated:** 2026-03-17
**Week 14, Day 2 Focus:** PIVOT Step C + Self-Referencing CTE on Real Data + Anti-Join in Complex Context

---

## Task 1: PIVOT Step C — Full Revenue Pivot by Transaction Type

**Scenario:**
You've mastered the single-column PIVOT. Now write the full pivot — but this time using **revenue** (SUM of amount) instead of counts, and include a `total_revenue` column as well.

**Expected Output Columns:**
- `month` (date) — truncated to month
- `deposit_revenue` (numeric) — rounded to 2 decimals
- `withdrawal_revenue` (numeric) — rounded to 2 decimals
- `payment_revenue` (numeric) — rounded to 2 decimals
- `transfer_revenue` (numeric) — rounded to 2 decimals
- `purchase_revenue` (numeric) — rounded to 2 decimals
- `total_revenue` (numeric) — sum of all types, rounded to 2 decimals

**Requirements:**
- Use `transactions` table
- Use `SUM(amount) FILTER (WHERE type = '...')` pattern
- Exclude NULL amounts
- Order by `month ASC`

**Difficulty Rating:** 3/5

I was wondering if you wanted total revenue for a given month or whole revenue for all time, but I assumed monthly revenues are expected

WITH transactions_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', t.created_at) AS month_
FROM crappy_data_db.transactions t
)
SELECT 
	month_,
	SUM(amount) FILTER (WHERE type = 'deposit') AS deposit_revenue,
	SUM(amount) FILTER (WHERE type = 'withdrawal') AS withdrawal_revenue,
	SUM(amount) FILTER (WHERE type = 'payment') AS payment_revenue,
	SUM(amount) FILTER (WHERE type = 'transfer') AS transfer_revenue,
	SUM(amount) FILTER (WHERE TYPE = 'purchase') AS purchase_revenue,
	SUM(amount) AS total_revenue
FROM transactions_months
GROUP BY month_
ORDER BY month_

---

## Task 2: Self-Referencing CTE — Product Category Tree

**Scenario:**
The `product_categories` table has a `parent_id` column — except it doesn't currently in our schema. Use this inline VALUES table as your data source (it's self-contained and runnable):

```sql
WITH categories (id, name, parent_id) AS (
    VALUES
    (1, 'All Products',    NULL),
    (2, 'Electronics',     1),
    (3, 'Clothing',        1),
    (4, 'Phones',          2),
    (5, 'Laptops',         2),
    (6, 'Men',             3),
    (7, 'Women',           3),
    (8, 'iPhone',          4),
    (9, 'Samsung',         4),
    (10, 'T-Shirts',       6)
)
```

Traverse this tree recursively to unlimited depth. Show each category's full path from root.

**Expected Output Columns:**
- `id` (integer)
- `name` (text)
- `parent_id` (integer)
- `path` (text) — e.g. `'All Products -> Electronics -> Phones -> iPhone'`
- `depth` (integer) — 1 for root

**Requirements:**
- Anchor: root node (`parent_id IS NULL`)
- Recursive: JOIN categories back to CTE on `categories.parent_id = cte.id`
- Path separator: ` -> ` (with spaces)
- Natural termination — no LEVEL limit needed
- Order by `path ASC`

**Difficulty Rating:** 3/5


WITH RECURSIVE categories (id, name, parent_id) AS (
    VALUES
    (1, 'All Products',    NULL),
    (2, 'Electronics',     1),
    (3, 'Clothing',        1),
    (4, 'Phones',          2),
    (5, 'Laptops',         2),
    (6, 'Men',             3),
    (7, 'Women',           3),
    (8, 'iPhone',          4),
    (9, 'Samsung',         4),
    (10, 'T-Shirts',       6)
),
hierarchy AS (
SELECT 
	1 AS id,
	'All Products' AS name,
	NULL::TEXT AS parent_id,
	'All Products' AS PATH,
	1 AS DEPTH
UNION ALL
SELECT
	c.id,
	c.name,
	h.name,
	h.PATH || ' < ' || c.name,
	h.DEPTH + 1
FROM hierarchy h
JOIN categories c ON h.id = c.parent_id
)
SELECT * FROM hierarchy


---

## Task 3: Anti-Join — Products Never Ordered

**Scenario:**
The inventory team wants to identify products that have never appeared in any order. These are candidates for removal from the catalogue.

Solve this using **all three approaches** (NOT IN, NOT EXISTS, LEFT JOIN IS NULL), but this time add a twist: the `orders_products` table links orders to products — so the subquery/join is one step removed from `products`.

Then answer: **which approach do you prefer here and why?**

**Expected Output Columns:**
- `product_id` (integer)
- `product_name` (text)
- `price` (numeric)

**Requirements:**
- Use `products` and `orders_products` tables
- Order by `product_id ASC`

**Difficulty Rating:** 3/5

1. SELECT id AS product_id FROM crappy_data_db.products p
WHERE id NOT IN (SELECT product_id FROM crappy_data_db.orders_products op WHERE op.product_id IS NOT NULL)

Very clean, although such products do not exist - thsi is probably the simplest option, and I consider it the most effective

2. SELECT id AS product_id 
FROM crappy_data_db.products p
WHERE NOT EXISTS
(SELECT * 
FROM crappy_data_db.orders_products op
WHERE op.product_id = p.id
)

Same, it also feels quite good.


3. SELECT p.id AS product_id 
FROM crappy_data_db.products p
LEFT JOIN crappy_data_db.orders_products op ON p.id = op.product_id 
WHERE op.product_id IS NULL


Quite sad that none of these actually give results this time, as there are no such products.
I'd pick options 1-2. 2 is good because it is NULL-proof, but I think option 1 is also NULL-proof, as long as we use IS NOT NULL condition in WHERE of our subquery.


---

## Submission Instructions

1. Task 1 — Full revenue PIVOT with total column (3/5)
2. Task 2 — Self-referencing category tree CTE (3/5)
3. Task 3 — Anti-join on products never ordered, three ways (3/5)
