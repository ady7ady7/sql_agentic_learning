# Daily SQL Practice Tasks

**Generated:** 2026-01-08
**Week 4, Day 5 Focus:** Complex Aggregations, Multi-Level Grouping, Advanced Filtering

---

## Task 1: Products with Declining Sales

**Scenario:**
The sales team wants to identify products with declining performance. For each product, calculate revenue for the last 3 months and the 3 months before that, then find products where recent revenue is lower than previous revenue.

**Expected Output Columns:**
- `product_id` (integer)
- `product_name` (varchar)
- `recent_3_month_revenue` (numeric) — total revenue from last 3 complete months
- `previous_3_month_revenue` (numeric) — total revenue from months 4-6 ago
- `revenue_decline` (numeric) — difference (recent - previous), should be negative
- `decline_percentage` (numeric) — percentage decline, rounded to 2 decimals

**Requirements:**
- Use `orders`, `orders_products`, and `products` tables
- Define "last 3 complete months" as the 3 full calendar months before the current month
- Calculate revenue as SUM(quantity * price)
- Only include products where recent_3_month_revenue < previous_3_month_revenue
- Order by `decline_percentage` ASC (most declining first)

**Difficulty Rating:** 5/5

---

## Task 2: User Engagement Tiers

**Scenario:**
The product team wants to classify users based on multiple engagement metrics. Create engagement tiers combining order frequency, total spend, and recency.

**Expected Output Columns:**
- `user_id` (integer)
- `total_orders` (bigint)
- `total_spent` (numeric)
- `days_since_last_order` (integer)
- `engagement_tier` (text) — classification based on combined criteria
- `tier_rank` (integer) — rank within their tier

**Requirements:**
- Use `users` and `orders` tables
- Calculate total orders, total spending, and days since last order per user
- Classify into tiers:
  - "Champion": 5+ orders AND spent > $500 AND last order within 30 days
  - "Loyal": 5+ orders AND spent > $300
  - "Recent": Last order within 30 days but doesn't meet Champion criteria
  - "At Risk": Last order 31-90 days ago
  - "Churned": Last order > 90 days ago
- Rank users within their tier by total_spent DESC
- Order by engagement_tier, then tier_rank

**Difficulty Rating:** 4/5

---

## Task 3: Category Cross-Sell Analysis

**Scenario:**
The marketing team wants to understand which product categories are frequently purchased together in the same order. Find category pairs that appear together in orders.

**Expected Output Columns:**
- `category_1_name` (varchar) — first category
- `category_2_name` (varchar) — second category
- `orders_together` (bigint) — count of orders containing both categories
- `avg_combined_revenue` (numeric) — average total revenue when both categories in same order, rounded to 2 decimals

**Requirements:**
- Use `orders`, `orders_products`, `products`, and `product_categories` tables
- Find orders containing products from at least 2 different categories
- Calculate how often each category pair appears together
- Calculate average revenue for orders containing both categories
- Avoid duplicate pairs (category A + B should not also appear as B + A)
- Exclude self-pairs (same category appearing twice)
- Only include pairs appearing together in at least 5 orders
- Order by `orders_together` DESC

**Difficulty Rating:** 5/5

---

## Submission Instructions

Submit your SQL solutions when ready. I'll provide detailed feedback on:
- Complex date range filtering and calculations
- Multi-criteria classification logic
- Self-joins for finding pairs
- Advanced aggregation patterns

## Tips

- Date ranges: Use EXTRACT to identify complete months
- Multi-criteria CASE: Can nest CASE WHEN conditions
- Self-joins: Remember to deduplicate with comparison operators
- Window functions: Can use RANK() OVER (PARTITION BY tier) for tier rankings

Good luck!
