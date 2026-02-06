# Daily SQL Practice Tasks

**Generated:** 2026-02-04
**Week 9, Day 1 Focus:** Hierarchy Practice + Multi-CTE Challenges

---

## Welcome to Week 9!

New format: 1 hierarchy task + 2 complex multi-CTE tasks that require actual thinking.

---

## Task 1: 3-Level Hierarchy — Categories → Products → Price Tiers

**Scenario:**
Build a 3-level hierarchy using hardcoded data:
- Level 1: 'All Products'
- Level 2: 'Budget' (< $50), 'Mid-Range' ($50-$150), 'Premium' (> $150)
- Level 3: Specific price points — 'Under $20', '$20-$50' (under Budget), '$50-$100', '$100-$150' (under Mid-Range), '$150-$300', '$300+' (under Premium)

Include path column.

**Expected Output Columns:**
- `level` (integer)
- `name` (text)
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Mapping CTE + recursive hierarchy with path
- Same pattern as Week 8 — write from memory

**Difficulty Rating:** 3/5

---

## Task 2: Monthly Revenue Dashboard (Multi-CTE)

**Scenario:**
Build a monthly revenue dashboard that shows for each month:
- Total revenue
- Month-over-month change (absolute and percentage)
- 3-month moving average
- Whether the month was above or below the overall average

**Expected Output Columns:**
- `month` (date) — DATE_TRUNC'd
- `monthly_revenue` (numeric)
- `prev_month_revenue` (numeric) — LAG
- `mom_change` (numeric) — current minus previous
- `mom_pct_change` (numeric) — percentage change, rounded to 1 decimal
- `moving_avg_3m` (numeric) — average of current + 2 preceding months, rounded to 2 decimals
- `vs_overall` (text) — 'Above Average' or 'Below Average'

**Requirements:**
- CTE 1: Aggregate orders by month
- CTE 2: Add LAG for previous month, calculate MoM change
- CTE 3: Add moving average with window frame
- Final SELECT: Compare to overall average using a subquery or additional CTE
- Round percentages and averages appropriately

**Difficulty Rating:** 5/5

---

## Task 3: Product Category Performance Comparison (Multi-CTE)

**Scenario:**
For each product category, calculate:
- Total revenue (quantity * price)
- Number of unique customers
- Average order value for that category
- Category's share of total revenue (as percentage)
- Rank by revenue

**Expected Output Columns:**
- `category_name` (text)
- `total_revenue` (numeric)
- `unique_customers` (bigint)
- `avg_order_value` (numeric) — average revenue per order containing this category, rounded to 2 decimals
- `revenue_share_pct` (numeric) — percentage of total revenue, rounded to 1 decimal
- `revenue_rank` (bigint)

**Requirements:**
- CTE 1: Join orders_products → products → product_categories → orders, aggregate by category
- CTE 2: Calculate total revenue across ALL categories (for percentage)
- Final SELECT: Combine with window function for rank
- Order by revenue_rank

**Difficulty Rating:** 5/5

---

## Week 9 Plan

| Day | Focus |
|-----|-------|
| Day 1 | Hierarchy + multi-CTE dashboards (TODAY) |
| Day 2 | Hierarchy + complex aggregation challenges |
| Day 3 | Hierarchy + subquery & CTE combinations |
| Day 4 | Hierarchy + HackerRank-style puzzles |
| Day 5 | Week consolidation |

---

## Submission Instructions

1. Task 1 — 3-level hierarchy from memory (3/5)
2. Task 2 — Monthly revenue dashboard, 4 CTEs (5/5)
3. Task 3 — Category performance comparison (5/5)

These should make you think. Take your time.

