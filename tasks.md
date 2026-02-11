# Daily SQL Practice Tasks

**Generated:** 2026-02-11
**Week 9, Day 4 Focus:** Hierarchy + HackerRank-Style Multi-CTE Puzzles

---

## Task 1: 3-Level Hierarchy — Delivery Statuses from Real Data

**Scenario:**
Build a 3-level hierarchy where Level 2 comes from **real data** (not hardcoded):
- Level 1: 'All Deliveries'
- Level 2: Distinct delivery statuses pulled from the `deliveries` table (e.g., 'pending', 'delivered', etc.)
- Level 3: Specific order IDs for each status, limited to the top 3 orders by amount per status

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — status name or order identifier
- `parent_name` (text)
- `path` (text) — full path from root

**Requirements:**
- Level 2 nodes should be dynamically pulled from `deliveries.status` (no hardcoded VALUES)
- Level 3 should show actual order IDs (as text), joined via `deliveries.order_id` → `orders`
- Limit Level 3 to top 3 orders per status by `orders.amount` DESC
- Include the path column showing the full hierarchy trail

**Difficulty Rating:** 4/5

---

## Task 2: Pareto Analysis — Revenue Concentration (80/20 Rule)

**Scenario:**
The business wants to understand revenue concentration. Apply the Pareto Principle: identify which users contribute to the top 80% of total revenue, and label them as "Key Accounts" vs "Standard Accounts."

**Expected Output Columns:**
- `user_id` (integer)
- `total_revenue` (numeric) — sum of all order amounts for this user
- `revenue_rank` (integer) — rank by total revenue descending
- `revenue_share_pct` (numeric) — this user's share of overall revenue, rounded to 2 decimals
- `cumulative_share_pct` (numeric) — running cumulative % of total revenue, rounded to 2 decimals
- `account_type` (text) — 'Key Account' if within top 80% cumulative, 'Standard Account' otherwise

**Requirements:**
- Use `orders` table
- Calculate each user's total revenue and their percentage share of overall revenue
- Calculate a running cumulative share (ordered by revenue DESC)
- Label users whose cumulative share is still within 80% as 'Key Account'
- Order by revenue_rank ASC

**Difficulty Rating:** 5/5

---

## Task 3: Support Ticket Complexity Scoring

**Scenario:**
The support team wants to identify complex tickets. Score each ticket based on message count and conversation duration, then rank them within each priority level.

**Expected Output Columns:**
- `ticket_id` (bigint)
- `priority` (text)
- `status` (text) — ticket status
- `message_count` (bigint) — total messages in the ticket
- `duration_hours` (numeric) — hours between first and last message, rounded to 1 decimal
- `complexity_score` (numeric) — `message_count * (duration_hours + 1)`, rounded to 1 decimal
- `priority_rank` (integer) — rank within priority level by complexity_score DESC

**Requirements:**
- Use `chat_tickets` and `chat_messages` tables
- Only count tickets that have at least 2 messages
- Duration = time between earliest and latest message in the ticket
- The `+1` in complexity formula prevents zero-duration tickets from scoring 0
- Order by priority, then priority_rank ASC

**Difficulty Rating:** 5/5

---

## Submission Instructions

1. Task 1 — Hierarchy with real-data Level 2 nodes (4/5)
2. Task 2 — Pareto / revenue concentration analysis (5/5)
3. Task 3 — Support ticket complexity scoring (5/5)
