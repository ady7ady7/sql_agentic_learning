# Daily SQL Practice Tasks

**Generated:** 2026-01-20
**Week 6, Day 5 Focus:** Recursive CTEs — Consolidation & Simple Combinations

---

## Final Day of Week 6

Today wraps up our recursive CTE focus. Tasks are designed to consolidate what you've learned without overwhelming complexity.

---

## Task 1: Generate Year-End Countdown

**Scenario:**
Generate a countdown showing the last 10 days of 2025 (December 22-31), with a "days until new year" counter.

**Expected Output Columns:**
- `date` (date) — December 22 through December 31
- `days_until_new_year` (integer) — 10, 9, 8, ... 1

**Requirements:**
- Use WITH RECURSIVE
- Anchor starts at '2025-12-22' with days_until = 10
- Recursive term adds 1 day and subtracts 1 from counter
- Terminate at December 31

**Difficulty Rating:** 3/5

---

## Task 2: Generate Fiscal Quarters (April Start)

**Scenario:**
Some companies use a fiscal year starting in April. Generate all 4 fiscal quarters for FY2025 (April 2025 - March 2026).

**Expected Output Columns:**
- `fiscal_quarter` (text) — 'FY25-Q1', 'FY25-Q2', 'FY25-Q3', 'FY25-Q4'
- `quarter_start` (date) — first day of each fiscal quarter
- `quarter_end` (date) — last day of each fiscal quarter

**Requirements:**
- Use WITH RECURSIVE
- Q1 starts April 1, 2025; Q4 ends March 31, 2026
- Track quarter number (1-4) for label generation
- Calculate quarter_end as start + 3 months - 1 day

**Difficulty Rating:** 3/5

**Hint for quarter_end:**
```sql
(quarter_start + INTERVAL '3 months' - INTERVAL '1 day')::DATE
```

---

## Task 3: Daily Transaction Summary with Generated Dates

**Scenario:**
Generate all days in December 2025, then show daily transaction counts and totals (including days with zero transactions).

**Expected Output Columns:**
- `day` (date) — each day in December 2025
- `transaction_count` (bigint) — number of transactions (0 if none)
- `daily_total` (numeric) — sum of amounts, rounded to 2 decimals (0 if none)

**Requirements:**
- Use WITH RECURSIVE to generate December 1-31, 2025
- LEFT JOIN to transactions table
- Match on DATE(created_at) = day
- Use COALESCE for zero handling
- Order by day

**Difficulty Rating:** 4/5

**This brings back the JOIN pattern from Day 3's quarterly revenue task.**

---

## Submission Instructions

Today's tasks:
1. Task 1 — Two counters moving opposite directions (date up, countdown down)
2. Task 2 — Fiscal year logic with calculated end dates
3. Task 3 — Recursive dates + LEFT JOIN (reinforcing the combination)

After today's review, we'll do the **Week 6 Recap**.

Good luck on the final day!
