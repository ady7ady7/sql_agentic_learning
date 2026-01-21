# Daily SQL Practice Tasks

**Generated:** 2026-01-19
**Week 6, Day 4 Focus:** Recursive CTEs — Pure Pattern Practice (No JOINs)

---

## Today's Goal

More repetition of the basic recursive pattern. No JOINs, no complex combinations — just the recursive CTE itself with small variations.

---

## Task 1: Countdown from 10 to 1

**Scenario:**
Generate a countdown: 10, 9, 8, 7, 6, 5, 4, 3, 2, 1

**Expected Output Columns:**
- `countdown` (integer) — values 10 down to 1

**Requirements:**
- Use WITH RECURSIVE
- Anchor starts at 10
- Recursive term subtracts 1
- Terminate at 1

**Difficulty Rating:** 2/5

---

## Task 2: Generate Hours of the Day

**Scenario:**
Generate all 24 hours of a day as timestamps (00:00, 01:00, 02:00, ... 23:00).

**Expected Output Columns:**
- `hour_timestamp` (timestamp) — starting from '2025-01-01 00:00:00' through '2025-01-01 23:00:00'
- `hour_label` (text) — '00:00', '01:00', '02:00', etc.

**Requirements:**
- Use WITH RECURSIVE
- Anchor starts at '2025-01-01 00:00:00'::TIMESTAMP
- Recursive term adds INTERVAL '1 hour'
- Terminate before reaching '2025-01-02 00:00:00'
- Use TO_CHAR(timestamp, 'HH24:MI') for the label

**Difficulty Rating:** 3/5

**Hint:**
```sql
TO_CHAR(hour_timestamp, 'HH24:MI')  -- Returns '00:00', '01:00', etc.
```

---

## Task 3: Multiplication Table (5x)

**Scenario:**
Generate the 5 times multiplication table: 5×1=5, 5×2=10, 5×3=15, ... up to 5×10=50

**Expected Output Columns:**
- `multiplier` (integer) — 1 through 10
- `result` (integer) — 5, 10, 15, 20, 25, 30, 35, 40, 45, 50

**Requirements:**
- Use WITH RECURSIVE
- Anchor starts with multiplier = 1
- Recursive term increments multiplier by 1
- Calculate result as 5 * multiplier
- Terminate at multiplier = 10

**Difficulty Rating:** 2/5

---

## Submission Instructions

Today is pure pattern practice:
1. Task 1 — Counting down (subtract instead of add)
2. Task 2 — Hours with timestamps (similar to dates)
3. Task 3 — Derived calculation (multiplier → result)

No JOINs, no aggregations — just getting comfortable with the recursive structure itself.

Good luck!
