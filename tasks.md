# Daily SQL Practice Tasks

**Generated:** 2026-01-17
**Week 6, Day 2 Focus:** Recursive CTEs — Foundations & Building Blocks

---

## Understanding Recursive CTEs

### Why Do We Need Them?

Regular CTEs are just named subqueries — they run once. **Recursive CTEs** run repeatedly, building results row by row until a termination condition is met.

**Real-world use cases:**
1. **Generating sequences** — dates, numbers, IDs (when no calendar/sequence table exists)
2. **Hierarchical data** — org charts (employee → manager → CEO), category trees, folder structures
3. **Graph traversal** — finding paths, connections, dependencies
4. **Iterative calculations** — compound interest, running totals with complex rules

**Why not just use a dates table or existing data?**
- Not every database has helper tables
- HackerRank/LeetCode problems often require generating data from nothing
- Some hierarchies are arbitrary depth — you can't hardcode the levels

### The Anatomy of a Recursive CTE

```sql
WITH RECURSIVE cte_name AS (
    -- ANCHOR: The starting point (runs once)
    SELECT initial_value AS column_name

    UNION ALL

    -- RECURSIVE TERM: References itself, runs until no new rows
    SELECT next_value
    FROM cte_name
    WHERE termination_condition
)
SELECT * FROM cte_name;
```

**How it works:**
1. **Anchor** executes first → produces initial row(s)
2. **Recursive term** runs, using previous iteration's results
3. Repeat step 2 until recursive term returns zero rows
4. All iterations' results are combined (UNION ALL)

### Simple Example — Counting 1 to 5

```sql
WITH RECURSIVE counter AS (
    SELECT 1 AS n           -- Anchor: start at 1
    UNION ALL
    SELECT n + 1            -- Recursive: add 1 each time
    FROM counter
    WHERE n < 5             -- Termination: stop when n reaches 5
)
SELECT * FROM counter;

-- Result: 1, 2, 3, 4, 5
```

---

## Task 1: Generate a Number Sequence (Warm-up)

**Scenario:**
Generate numbers from 1 to 10 using a recursive CTE.

**Expected Output Columns:**
- `number` (integer) — values 1 through 10

**Requirements:**
- Use WITH RECURSIVE
- Anchor starts at 1
- Recursive term adds 1
- Terminate when reaching 10

**Difficulty Rating:** 2/5

**This is intentionally simple** — master the pattern before adding complexity.

---

## Task 2: Generate a Date Series (Apply the Pattern)

**Scenario:**
Generate all dates in January 2025 (2025-01-01 through 2025-01-31) using a recursive CTE.

**Expected Output Columns:**
- `date_value` (date) — each day in January 2025

**Requirements:**
- Use WITH RECURSIVE
- Anchor starts at '2025-01-01'
- Recursive term adds 1 day using INTERVAL
- Terminate at '2025-01-31'
- Cast properly to DATE type

**Difficulty Rating:** 3/5

**Hint:**
```sql
SELECT (date_value + INTERVAL '1 day')::DATE
```

---

## Task 3: Monthly Revenue Report with Generated Dates

**Scenario:**
Now combine what you learned: generate all 12 months of 2025 using a recursive CTE, then LEFT JOIN to orders to show revenue per month (including months with zero revenue).

**Expected Output Columns:**
- `month_start` (date) — first day of each month (2025-01-01 through 2025-12-01)
- `order_count` (bigint) — number of orders (0 if none)
- `total_revenue` (numeric) — sum of amounts, rounded to 2 decimals (0 if none)

**Requirements:**
- Use WITH RECURSIVE to generate 12 months
- LEFT JOIN to orders table
- Use COALESCE for zero handling
- Order by month_start ASC

**Difficulty Rating:** 4/5

**This is yesterday's Task 1 done correctly** — now with the recursive pattern you understand.

---

## Submission Instructions

Today is about building the recursive CTE muscle memory:
1. Task 1 — Pure pattern practice (numbers)
2. Task 2 — Apply to dates
3. Task 3 — Combine with real data

Tomorrow we'll move to hierarchical queries (org charts, category trees) which is where recursive CTEs really shine.

Good luck!
