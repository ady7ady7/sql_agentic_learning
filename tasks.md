# Daily SQL Practice Tasks

**Generated:** 2026-02-05
**Week 9, Day 2 Focus:** Hierarchy Practice + Complex Aggregation Challenges

---

## Task 1: 3-Level Hierarchy with Real Data — Categories → Products → Delivery Status

**Scenario:**
Build a 3-level hierarchy from real tables:
- Level 1: Product categories (from `product_categories`)
- Level 2: Products (from `products` via category_id)
- Level 3: Delivery statuses for orders containing that product (from `orders_products` → `deliveries`)

Limit to category 'travel' to keep output manageable.

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — category name, product name, or delivery status
- `parent_name` (text)
- `path` (text)

**Requirements:**
- Anchor: Category 'travel' from product_categories
- Level 2: JOIN products on category_id
- Level 3: JOIN orders_products → deliveries to get statuses
- Carry necessary IDs through recursion
- Include path column

**Difficulty Rating:** 4/5

**Hint:** You'll need `category_id` at level 1→2 and `product_id` at level 2→3. Use LEFT JOINs with level conditions like Day 3 of Week 8.

---

## Task 2: Customer Lifetime Value Segmentation (Multi-CTE)

**Scenario:**
Build a customer segmentation report that classifies users into value tiers based on their total spending, order frequency, and average order value.

**Expected Output Columns:**
- `user_id` (integer)
- `total_spent` (numeric) — sum of all order amounts
- `order_count` (bigint) — number of orders
- `avg_order_value` (numeric) — average order amount, rounded to 2 decimals
- `days_as_customer` (integer) — days between first and last order
- `orders_per_month` (numeric) — order_count / (days_as_customer / 30.0), rounded to 2 decimals
- `value_tier` (text) — 'Platinum' (top 10%), 'Gold' (top 25%), 'Silver' (top 50%), 'Bronze' (bottom 50%) based on total_spent

**Requirements:**
- CTE 1: Aggregate per user — total_spent, order_count, avg_order, first/last order dates
- CTE 2: Calculate days_as_customer and orders_per_month
- CTE 3: Add PERCENT_RANK to determine value tier
- Final SELECT: Apply CASE WHEN on percentile for tier labels
- Filter out users with only 1 order (days_as_customer = 0 causes division issues)
- Order by total_spent DESC

**Difficulty Rating:** 5/5

---

## Task 3: Order Gap Analysis by User (Multi-CTE)

**Scenario:**
For each user, find the gaps between consecutive orders and identify patterns:
- Average gap between orders
- Longest gap
- Whether they're "accelerating" (recent gaps shorter than average) or "slowing down"

**Expected Output Columns:**
- `user_id` (integer)
- `order_count` (bigint)
- `avg_gap_days` (numeric) — average days between consecutive orders, rounded to 1 decimal
- `max_gap_days` (integer) — longest gap in days
- `last_gap_days` (integer) — most recent gap
- `trend` (text) — 'Accelerating' if last_gap < avg_gap, 'Slowing Down' if last_gap > avg_gap, 'Steady' if equal

**Requirements:**
- CTE 1: Get all orders per user with LAG to find previous order date
- CTE 2: Calculate gap in days (current_date - prev_date) using DATE_PART or EXTRACT
- CTE 3: Aggregate per user — AVG gap, MAX gap, and get the last gap (most recent)
- Final SELECT: Determine trend
- Only include users with 3+ orders
- Order by avg_gap_days

**Difficulty Rating:** 5/5

**Hint for last gap:** Use FIRST_VALUE with ORDER BY created_at DESC within PARTITION BY user_id to get the most recent gap.

---

## Submission Instructions

1. Task 1 — 3-level real data hierarchy with path (4/5)
2. Task 2 — Customer lifetime value segmentation (5/5)
3. Task 3 — Order gap analysis with trend (5/5)

