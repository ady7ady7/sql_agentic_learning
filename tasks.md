# Daily SQL Practice Tasks

**Generated:** 2026-02-01
**Week 8, Day 3 Focus:** 3-Level Hierarchies + Advanced Window Functions

---

## Task 1: 3-Level Hierarchy — Company → Department → Team

**Scenario:**
Extend to 3 levels using hardcoded data:
- Level 1: 'TechCorp' (the company)
- Level 2: 'Engineering', 'Sales' (departments)
- Level 3: 'Backend', 'Frontend' (under Engineering), 'Inside Sales', 'Enterprise' (under Sales)

**Expected Output:**
```
level | name         | parent_name
------+--------------+------------
1     | TechCorp     | NULL
2     | Engineering  | TechCorp
2     | Sales        | TechCorp
3     | Backend      | Engineering
3     | Frontend     | Engineering
3     | Inside Sales | Sales
3     | Enterprise   | Sales
```

**Requirements:**
- Use mapping CTE to define parent-child relationships for ALL levels
- Recursive CTE that can go 3 levels deep
- Change `WHERE h.level = 1` to `WHERE h.level < 3` to allow 3 iterations

**Difficulty Rating:** 3/5

**Hint:** Your mapping CTE needs entries like:
- ('Engineering', 'TechCorp')
- ('Backend', 'Engineering')
- etc.

---

## Task 2: 3-Level with Real Data — Categories → Products → Orders

**Scenario:**
Build a 3-level hierarchy from actual tables:
- Level 1: Product categories
- Level 2: Products (linked via category_id)
- Level 3: Order IDs where that product was purchased (via orders_products)

Limit to 1 category ('travel') to keep output manageable.

**Expected Output Columns:**
- `level` (integer) — 1, 2, or 3
- `name` (text) — category name, product name, or order ID as text
- `parent_name` (text) — NULL for level 1, parent's name for levels 2-3

**Requirements:**
- Anchor: Single category 'travel'
- Level 2: JOIN products on category_id
- Level 3: JOIN orders_products on product_id, then show order_id
- Carry necessary IDs through (category_id for level 2, product_id for level 3)

**Difficulty Rating:** 4/5

---

## Task 3: Advanced — Running Difference with LAG

**Scenario:**
For each user, show their transactions in chronological order with:
- The transaction amount
- The previous transaction amount (using LAG)
- The difference from previous transaction

**Expected Output Columns:**
- `user_id` (integer)
- `transaction_date` (date)
- `amount` (numeric)
- `prev_amount` (numeric) — previous transaction amount, NULL for first
- `amount_diff` (numeric) — current minus previous, NULL for first

**Requirements:**
- Use LAG() with PARTITION BY user_id ORDER BY created_at
- Calculate difference as amount - prev_amount
- Only include users 1-5 to limit output
- Order by user_id, transaction_date

**Difficulty Rating:** 4/5

---

## Week 8 Scaffolding Plan

| Day | Focus | Status |
|-----|-------|--------|
| Day 1 | 2-level hierarchy with mapping CTE | ✓ Done |
| Day 2 | 2-level with real table data | ✓ Done |
| Day 3 | 3-level hierarchy introduction (TODAY) | |
| Day 4 | Path building (e.g., 'Category > Product') | |
| Day 5 | Combine hierarchies with aggregations | |

---

## Submission Instructions

1. Task 1 — 3-level hardcoded hierarchy (3/5)
2. Task 2 — 3-level with real tables (4/5)
3. Task 3 — LAG with running difference (4/5)

Today adds depth — from 2 levels to 3.

