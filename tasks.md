# Daily SQL Practice Tasks

**Generated:** 2026-01-31
**Week 8, Day 2 Focus:** Hierarchical CTEs with Real Table Data + Advanced Window Functions

---

## Task 1: Product Categories → Products (2-Level Hierarchy)

**Scenario:**
Using the actual `product_categories` and `products` tables, build a 2-level hierarchy showing categories at level 1 and their products at level 2.

**Expected Output Columns:**
- `level` (integer) — 1 for categories, 2 for products
- `name` (text) — category name or product name
- `parent_name` (text) — NULL for categories, category name for products

**Requirements:**
- Use same pattern as Day 1 (mapping CTE + hierarchy CTE)
- Anchor: SELECT from `product_categories`
- Recursive: JOIN to `products` where `products.category_id` matches
- Order by parent_name NULLS FIRST, then name

**Difficulty Rating:** 3/5

**Hint:** The "mapping" is already in the database — `products.category_id` points to `product_categories.id`.

---

## Task 2: Users → Orders Hierarchy (2-Level)

**Scenario:**
Build a 2-level hierarchy: Users (level 1) → Their orders (level 2).
Only include users who have at least 1 order.

**Expected Output Columns:**
- `level` (integer) — 1 for users, 2 for orders
- `name` (text) — user's email for level 1, order ID cast to text for level 2
- `parent_name` (text) — NULL for users, user's email for orders

**Requirements:**
- Anchor: Users who have orders (use EXISTS or IN subquery)
- Recursive: JOIN to orders where `orders.user_id` matches user
- Limit to first 5 users (by user id) to keep output manageable
- Order by parent_name NULLS FIRST, level, name

**Difficulty Rating:** 3/5

---

## Task 3: Advanced Window Function — Percentile Ranking

**Scenario:**
For each user, calculate their total order amount and rank them by percentile (what % of users have less total spending than them).

**Expected Output Columns:**
- `user_id` (integer)
- `total_spent` (numeric) — sum of all their order amounts
- `percentile_rank` (numeric) — value between 0 and 1, rounded to 2 decimals

**Requirements:**
- Aggregate orders by user_id first
- Use `PERCENT_RANK()` window function
- Order by percentile_rank DESC
- Only include users with total_spent > 0

**Difficulty Rating:** 4/5

**Note:** `PERCENT_RANK()` returns (rank - 1) / (total_rows - 1), giving 0 to the lowest and 1 to the highest.

---

## Week 8 Scaffolding Plan

| Day | Focus | Status |
|-----|-------|--------|
| Day 1 | 2-level hierarchy with mapping CTE | ✓ Done |
| Day 2 | 2-level with real table data (TODAY) | |
| Day 3 | 3-level hierarchy introduction | |
| Day 4 | Path building (e.g., 'Category > Product') | |
| Day 5 | Combine hierarchies with aggregations | |

---

## Submission Instructions

1. Task 1 — Categories → Products hierarchy (3/5)
2. Task 2 — Users → Orders hierarchy (3/5)
3. Task 3 — PERCENT_RANK window function (4/5)

Today applies yesterday's pattern to real data.

