# Daily SQL Practice Tasks

**Generated:** 2026-02-03
**Week 8, Day 5 Focus:** Week Consolidation — Hierarchy Review + Window Functions

---

## Final Day of Week 8

Consolidating what we've learned. No new complexity — just reinforcement.

---

## Task 1: 3-Level Hierarchy — Organization Structure

**Scenario:**
Build a 3-level org hierarchy:
- Level 1: 'Acme Inc'
- Level 2: 'HR', 'Finance', 'IT'
- Level 3: 'Recruiting', 'Payroll' (under HR), 'Accounting', 'Tax' (under Finance), 'DevOps', 'Support' (under IT)

**Expected Output:**
```
level | name       | parent_name | path
------+------------+-------------+------------------------
1     | Acme Inc   | NULL        | Acme Inc
2     | HR         | Acme Inc    | Acme Inc > HR
2     | Finance    | Acme Inc    | Acme Inc > Finance
2     | IT         | Acme Inc    | Acme Inc > IT
3     | Recruiting | HR          | Acme Inc > HR > Recruiting
3     | Payroll    | HR          | Acme Inc > HR > Payroll
3     | Accounting | Finance     | Acme Inc > Finance > Accounting
3     | Tax        | Finance     | Acme Inc > Finance > Tax
3     | DevOps     | IT          | Acme Inc > IT > DevOps
3     | Support    | IT          | Acme Inc > IT > Support
```

**Requirements:**
- Mapping CTE with all relationships
- Include path column from the start
- Write it WITHOUT looking at previous solutions

**Difficulty Rating:** 3/5

---

## Task 2: 2-Level Real Data with Path — Categories → Products

**Scenario:**
Build a 2-level hierarchy from `product_categories` and `products` with path column.

**Expected Output Columns:**
- `level` (integer)
- `name` (text)
- `parent_name` (text)
- `path` (text) — e.g., 'travel > Luggage'

**Requirements:**
- Anchor: All categories from `product_categories`
- Recursive: JOIN to `products` on category_id
- Include path (category name for level 1, category > product for level 2)
- Carry `category_id` through for the JOIN

**Difficulty Rating:** 3/5

This is simpler than Day 3's 3-level real data — just 2 levels.

---

## Task 3: Advanced — DENSE_RANK with Ties

**Scenario:**
Rank products by their total quantity sold across all orders. Use DENSE_RANK to handle ties (products with same quantity get same rank, no gaps).

**Expected Output Columns:**
- `product_id` (integer)
- `product_name` (text)
- `total_quantity` (bigint) — sum of quantity from orders_products
- `sales_rank` (bigint) — DENSE_RANK by total_quantity DESC

**Requirements:**
- JOIN products and orders_products
- Aggregate by product
- Use DENSE_RANK() OVER (ORDER BY total_quantity DESC)
- Order by sales_rank, then product_name

**Difficulty Rating:** 4/5

---

## Week 8 Summary

| Day | Focus | Score |
|-----|-------|-------|
| Day 1 | 2-level hierarchy introduction | 28/30 |
| Day 2 | 2-level with real data | 29/30 |
| Day 3 | 3-level introduction (struggled) | 24/30 |
| Day 4 | 3-level practice + path building | 29/30 |
| Day 5 | Consolidation (TODAY) | |

---

## Submission Instructions

1. Task 1 — 3-level org with path, from memory (3/5)
2. Task 2 — 2-level real data with path (3/5)
3. Task 3 — DENSE_RANK product ranking (4/5)

After today: Week 8 Recap, then Week 9 planning.

