# Daily SQL Practice Tasks

**Generated:** 2026-02-06
**Week 9, Day 3 Focus:** Hierarchy + Subquery & CTE Combinations

---

## Task 1: 3-Level Hierarchy — Users → Order Count Tiers → Specific Users

**Scenario:**
Build a 3-level hierarchy with hardcoded tiers and real user data:
- Level 1: 'All Users'
- Level 2: 'Power Users' (10+ orders), 'Regular Users' (3-9 orders), 'New Users' (1-2 orders)
- Level 3: Actual user emails from the `orders` table, matched to their tier

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — tier name or user email
- `parent_name` (text)

**Requirements:**
- CTE 1: Aggregate orders per user, classify into tier based on COUNT
- Recursive hierarchy: Anchor = 'All Users', Level 2 = tier names, Level 3 = user emails
- The tricky part: Level 2→3 maps real users to hardcoded tier names
- Limit level 3 to first 3 users per tier (use ROW_NUMBER in the classification CTE)

**Difficulty Rating:** 4/5

---

## Task 2: Revenue Anomaly Detection (Multi-CTE)

**Scenario:**
Find days where revenue was "anomalous" — significantly above or below the norm. Define anomalous as more than 1.5x the 7-day moving average, or less than 0.5x.

**Expected Output Columns:**
- `order_date` (date)
- `daily_revenue` (numeric)
- `moving_avg_7d` (numeric) — 7-day trailing average, rounded to 2 decimals
- `ratio_to_avg` (numeric) — daily_revenue / moving_avg_7d, rounded to 2 decimals
- `anomaly_type` (text) — 'Spike' (> 1.5x), 'Drop' (< 0.5x), or 'Normal'

**Requirements:**
- CTE 1: Aggregate orders by date
- CTE 2: Add 7-day moving average (ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
- CTE 3: Calculate ratio and classify anomaly
- Final SELECT: Show ALL days but order by ratio_to_avg DESC to see spikes first
- Handle NULL moving averages for first 6 days (use NULLIF or CASE)

**Difficulty Rating:** 5/5

---

## Task 3: Cross-Category Buyers Analysis (Multi-CTE)

**Scenario:**
Find users who buy from multiple product categories. For each such user, show how many categories they buy from, their total spending per category, and their "favorite" category (highest spending).

**Expected Output Columns:**
- `user_id` (integer)
- `categories_count` (bigint) — number of distinct categories purchased
- `favorite_category` (text) — category with highest spending
- `favorite_category_revenue` (numeric) — spending in favorite category
- `total_spent` (numeric) — total across all categories
- `favorite_pct` (numeric) — what % of total spending goes to favorite category, rounded to 1 decimal

**Requirements:**
- CTE 1: Join orders → orders_products → products → product_categories, aggregate by user_id + category
- CTE 2: For each user, count categories and get total spending
- CTE 3: Rank categories per user by spending to find favorite (ROW_NUMBER or RANK)
- Final SELECT: Combine and calculate percentage
- Only include users who buy from 2+ categories
- Order by categories_count DESC, total_spent DESC

**Difficulty Rating:** 5/5

---

## Submission Instructions

1. Task 1 — Hierarchy mixing hardcoded + real data (4/5)
2. Task 2 — Revenue anomaly detection (5/5)
3. Task 3 — Cross-category buyer analysis (5/5)

