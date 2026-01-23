# Daily SQL Practice Tasks

**Generated:** 2026-01-21
**Week 7, Day 1 Focus:** Recursive CTEs — Continued Practice + Gentle Introduction to Levels

---

## Monday Session

Lighter load today. We'll continue reinforcing recursive CTEs while introducing one new concept: tracking **depth/level** in the recursion.

---

## Task 1: Generate a Simple Number Pyramid

**Scenario:**
Generate numbers 1 through 5, but also show the cumulative sum at each step.

**Expected Output Columns:**
- `n` (integer) — 1, 2, 3, 4, 5
- `cumulative_sum` (integer) — 1, 3, 6, 10, 15 (running total)

**Requirements:**
- Use WITH RECURSIVE
- Anchor starts at n=1, cumulative_sum=1
- Recursive term: n+1, cumulative_sum + (n+1)
- Terminate at n=5

**Difficulty Rating:** 3/5

**This introduces:** Carrying forward a calculated value (cumulative sum) through iterations.

---

## Task 2: Weekly Order Summary for Q1 2025

**Scenario:**
Generate all weeks in Q1 2025 (January-March), then show order count and revenue per week.

**Expected Output Columns:**
- `week_start` (date) — Monday of each week
- `week_number` (integer) — 1, 2, 3, ... (week counter)
- `order_count` (bigint) — orders in that week (0 if none)
- `weekly_revenue` (numeric) — sum of amounts, rounded to 2 decimals (0 if none)

**Requirements:**
- Use WITH RECURSIVE to generate weeks starting from '2024-12-30' (Monday before Jan 1)
- Terminate when week_start reaches April
- LEFT JOIN to orders
- Match orders where created_at falls within the week (>= week_start AND < week_start + 7 days)

**Difficulty Rating:** 4/5

**This reinforces:** Recursive CTE + LEFT JOIN pattern from last week.

---

## Task 3: Power of 2 Sequence

**Scenario:**
Generate powers of 2 from 2^0 to 2^10: 1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024

**Expected Output Columns:**
- `exponent` (integer) — 0, 1, 2, 3, ... 10
- `power_of_2` (integer) — 1, 2, 4, 8, ... 1024

**Requirements:**
- Use WITH RECURSIVE
- Anchor starts at exponent=0, power_of_2=1
- Recursive term: exponent+1, power_of_2 * 2
- Terminate at exponent=10

**Difficulty Rating:** 2/5

**This reinforces:** Multiplicative progression (similar to addition but with multiplication).

---

## Submission Instructions

Today's tasks:
1. Task 1 — Cumulative calculation within recursion (3/5)
2. Task 2 — Weekly aggregation with recursive dates (4/5)
3. Task 3 — Simple multiplicative sequence (2/5)

Balanced Monday session with one moderate challenge (Task 2) and two reinforcement tasks.

Good luck!
