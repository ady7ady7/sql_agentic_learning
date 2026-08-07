# SQL Tasks — 2026-08-07 (Week 33, Day 5)

**Dataset:** orders / transactions / users / product_categories / products / orders_products  
**Focus:** Recursive CTE (Type A) · Window with conditional reset · Date spine

---

## Task 1 — Category → Product → Order Revenue Hierarchy
**Difficulty: 4/5**

**Business question:**  
Build a 3-level revenue rollup:
- **Level 1:** Total revenue per product category
- **Level 2:** Total revenue per product (within its category)
- **Level 3:** Total revenue per individual order line (quantity × price, within its product)

Show all three levels in a single result set with a `level` column (1, 2, or 3), a `label` column (category name / product name / order_id as text), a `parent_label` (NULL for level 1, category name for level 2, product name for level 3), and `revenue`.

Order by `level`, `revenue DESC`.

**Scaffold — recursive CTE structure to fill in:**

```sql
WITH RECURSIVE hierarchy AS (

    -- Anchor: Level 1 — category totals
    SELECT
        1 AS level,
        pc.name AS label,
        NULL::text AS parent_label,
        SUM(???) AS revenue          -- fill in the revenue formula
    FROM ???                          -- fill in the tables and joins
    GROUP BY pc.name

    UNION ALL

    -- Recursive step: Level 2 — product totals, Level 3 — order line totals
    SELECT
        h.level + 1,
        ???,                          -- label for this level
        h.label AS parent_label,
        ???                           -- revenue for this level
    FROM hierarchy h
    JOIN ???                          -- join to get next level down
    WHERE h.level < 3

)
SELECT * FROM hierarchy
ORDER BY level, revenue DESC;
```

Note: revenue at level 3 = `quantity × price` from `orders_products` joined to `products`.

**Expected output columns:**  
`level, label, parent_label, revenue`

**Difficulty: 4/5**

WITH RECURSIVE product_revenues AS (
SELECT
	p.name::TEXT,
	p.id,
	pc.name::TEXT AS category_name,
	SUM(op.quantity * p.price) AS product_revenue
FROM crappy_data_db.orders_products op 
JOIN crappy_data_db.products p  ON op.product_id = p.id
JOIN crappy_data_db.product_categories pc ON p.category_id = pc.id
GROUP BY p.name, pc.name, p.id
),
orders_revenues AS (
SELECT
	p.name,
	p.id,
	op.order_id,
	SUM(op.quantity * p.price) AS order_revenue
FROM crappy_data_db.orders_products op 
JOIN crappy_data_db.products p  ON op.product_id = p.id
JOIN crappy_data_db.product_categories pc ON p.category_id = pc.id
GROUP BY p.name, p.id, op.order_id
),
HIERARCHY AS (
SELECT 
	1 AS LEVEL,
	pc.name::text AS LABEL,
	NULL::TEXT AS parent_label,
	SUM(p.price * op.quantity) AS revenue
FROM crappy_data_db.orders_products op 
JOIN crappy_data_db.products p  ON op.product_id = p.id
JOIN crappy_data_db.product_categories pc ON p.category_id = pc.id
GROUP BY pc.name
UNION ALL 
SELECT
	h.LEVEL + 1,
	COALESCE(p.name::TEXT, o.order_id::TEXT),
	h.LABEL,
	COALESCE(p.product_revenue, o.order_revenue)
FROM hierarchy h
LEFT JOIN product_revenues p ON h.LABEL = p.category_name AND h.LEVEL = 1
LEFT JOIN orders_revenues o ON h.LABEL = o."name" AND h.LEVEL = 2
GROUP BY p.name, o.order_id, o.order_revenue, h.LEVEL, h.LABEL, p.product_revenue
)
SELECT * FROM hierarchy


Mamy to, trochębyło z tym pierdolenia, szczególnie że dawno tego nie robiłem i zapomniałem, żę joiny oba mają być już na pierwszym UNION ALLU, ale dałem radę

---

## Task 2 — Running Total with Reset
**Difficulty: 5/5**

**Business question:**  
For each user, compute a running total of transaction `amount` ordered by `created_at`. Every time the running total **exceeds 1000**, reset it back to 0 and start accumulating again from the next transaction.

Show each transaction with its `group_id` (which "cycle" it belongs to, starting at 1 per user) and the running total within that cycle.

Hint: the trick is identifying group boundaries using a cumulative sum of a reset flag, then using that as a partition key for the inner running total.

**Expected output columns:**  
`user_id, id, created_at, amount, group_id, running_total_in_group`

Order by `user_id`, `created_at`, `id`.

**Difficulty: 5/5**

WITH first_agg AS (
SELECT 
	*,
	SUM(t.amount) OVER (PARTITION BY user_id ORDER BY created_at) AS running_total
FROM crappy_data_db.transactions t
),
totals_ids AS (
SELECT 
	*,
	FLOOR(running_total / 1000) AS group_id
FROM first_agg
)
SELECT 
	*,
	SUM(amount) OVER (PARTITION BY user_id, group_id ORDER BY created_at) AS running_total_in_group
FROM  totals_ids

Ogarnięte z twoją pomocą - w sumie teraz nie wydaje siętakie trudne, ale nie pomyślałem wcześniej o tym FLOOR(running_total/1000)


---

## Task 3 — Months with No Orders (Date Spine)
**Difficulty: 3/5**

**Business question:**  
Using the `dates` table as a calendar spine, find every month between the first and last order date where **no orders were placed at all**. Show the month and a `order_count` of 0.

Then extend the query: for months that DO have orders, show the month and the actual order count. The final result should cover every month in the range with either the real count or 0.

**Expected output columns:**  
`order_month, order_count`

Order by `order_month`.

**Difficulty: 3/5**

WITH orders_months AS (
SELECT 
	DATE_TRUNC('Month', d."date") AS month_,
	COALESCE(COUNT(o.id), 0) AS order_count
FROM crappy_data_db.dates d
LEFT JOIN crappy_data_db.orders o ON d."date" = DATE_TRUNC('Month', o.created_at)
GROUP BY DATE_TRUNC('Month', d."date")
ORDER BY month_
),
fl_order_dates AS (
SELECT	
	MIN(created_at) AS first_order,
	MAX(created_at) AS last_order
FROM crappy_data_db.orders o
)
SELECT
	om.month_,
	om.order_count
FROM orders_months om
LEFT JOIN fl_order_dates od ON om.month_ > od.first_order AND om.month_ < od.last_order
WHERE om.month_ > od.first_order AND om.month_ < od.last_order

---

## Submission Instructions

Paste your queries below each task.
