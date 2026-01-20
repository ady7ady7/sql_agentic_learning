# Daily SQL Practice Tasks

**Generated:** 2026-01-18
**Week 6, Day 3 Focus:** Recursive CTEs — Hierarchical Data Traversal

---

## Hierarchical Data with Recursive CTEs

### The Problem

Many real-world datasets have **parent-child relationships**:
- Employees → Managers → Directors → CEO
- Categories → Subcategories → Sub-subcategories
- Folders → Subfolders → Files
- Comments → Replies → Nested replies

You can't solve these with regular JOINs because **you don't know how deep the hierarchy goes**.

### The Pattern

```sql
WITH RECURSIVE hierarchy AS (
    -- ANCHOR: Start at the root(s) — rows with no parent
    SELECT id, name, parent_id, 1 AS level
    FROM table
    WHERE parent_id IS NULL

    UNION ALL

    -- RECURSIVE: Join children to their parents
    SELECT child.id, child.name, child.parent_id, parent.level + 1
    FROM table child
    JOIN hierarchy parent ON child.parent_id = parent.id
)
SELECT * FROM hierarchy;
```

**Key differences from sequence generation:**
- Anchor selects from a real table (root nodes)
- Recursive term JOINs the table to itself via the CTE
- Often tracks `level` (depth) for ordering/display

### Example: Simple Org Chart

Given this data:
```
id | name    | manager_id
1  | Alice   | NULL        (CEO)
2  | Bob     | 1           (reports to Alice)
3  | Carol   | 1           (reports to Alice)
4  | Dave    | 2           (reports to Bob)
```

The recursive CTE walks the tree:
1. Anchor finds Alice (manager_id IS NULL) → level 1
2. Iteration 1 finds Bob, Carol (manager_id = 1) → level 2
3. Iteration 2 finds Dave (manager_id = 2) → level 3
4. No more children → done

---

## Task 1: Build a Category Tree

**Scenario:**
Our `product_categories` table doesn't have a parent_id column, but let's simulate one. Create a recursive CTE that generates a 3-level category hierarchy from scratch, then display it with indentation.

**Expected Output Columns:**
- `level` (integer) — depth in hierarchy (1, 2, or 3)
- `category_path` (text) — indented category name showing hierarchy

**Requirements:**
- Use WITH RECURSIVE to generate a simulated 3-level hierarchy:
  - Level 1: 'Electronics', 'Clothing', 'Home'
  - Level 2: Under Electronics: 'Phones', 'Laptops'; Under Clothing: 'Men', 'Women'; Under Home: 'Kitchen', 'Bedroom'
  - Level 3: Under Phones: 'iPhone', 'Android'; Under Laptops: 'Gaming', 'Business'
- Use string concatenation to show indentation (e.g., '  ' per level)
- Output should show the tree structure visually

**Difficulty Rating:** 3/5

**Hint — Building the hierarchy:**
```sql
WITH RECURSIVE categories AS (
    -- Anchor: Level 1 categories
    SELECT 1 AS level, 'Electronics' AS name, NULL::TEXT AS parent
    UNION ALL SELECT 1, 'Clothing', NULL
    UNION ALL SELECT 1, 'Home', NULL

    UNION ALL

    -- Recursive: Add children based on parent
    SELECT
        c.level + 1,
        CASE
            WHEN c.name = 'Electronics' AND c.level = 1 THEN 'Phones'
            -- ... more mappings
        END,
        c.name
    FROM categories c
    WHERE c.level < 3
)
```

This task is about understanding how the recursive term can build arbitrary structures.

---

## Task 2: Ticket Response Chain

**Scenario:**
In our `chat_messages` table, messages within a ticket form a conversation chain. Find the longest conversation chains (most messages) per ticket, and show the time elapsed from first to last message.

**Expected Output Columns:**
- `ticket_id` (bigint)
- `message_count` (bigint) — total messages in the ticket
- `first_message_time` (timestamp)
- `last_message_time` (timestamp)
- `conversation_duration_hours` (numeric) — hours between first and last message, rounded to 1 decimal

**Requirements:**
- Use `chat_messages` table
- Group by ticket_id
- Calculate duration using EXTRACT(EPOCH FROM ...) / 3600
- Order by message_count DESC
- Limit to top 10 longest conversations

**Difficulty Rating:** 3/5

**Note:** This task uses aggregation, not recursive CTE — it's a breather task to practice other skills while staying in the messaging domain. Tomorrow we'll do recursive message threading.

---

## Task 3: Cumulative User Growth by Month

**Scenario:**
Using recursive CTE for date generation, show cumulative user registrations over time — how many total users existed at the end of each month.

**Expected Output Columns:**
- `month` (date) — first day of month
- `new_users` (bigint) — users who registered that month
- `cumulative_users` (bigint) — running total of all users up to that month

**Requirements:**
- Use WITH RECURSIVE to generate months (from earliest user registration to latest)
- LEFT JOIN to users table, counting registrations per month
- Use window function SUM() OVER (ORDER BY month) for cumulative total
- Handle months with zero registrations

**Difficulty Rating:** 4/5

**This combines:**
- Recursive CTE (date generation)
- Aggregation (counting per month)
- Window function (running total)

---

## Submission Instructions

Today's progression:
1. Task 1 — Hierarchical structure building (conceptual)
2. Task 2 — Breather task with aggregation
3. Task 3 — Combining recursive CTE + window functions

Tomorrow: Recursive CTEs with actual self-referential table data (if we can find/create a hierarchy in the schema).

Good luck!
