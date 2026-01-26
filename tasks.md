# Daily SQL Practice Tasks

**Generated:** 2026-01-22
**Week 7, Day 2 Focus:** Recursive CTEs — The "Carry Forward" Pattern

---

## Today's Focus

Yesterday's key learning: **In recursive CTEs, each iteration has direct access to the previous row's values.** You don't need window functions or built-in functions — just reference the column and modify it.

Today we'll drill this pattern with simple, focused tasks.

---

## The Pattern

```sql
WITH RECURSIVE cte AS (
    -- Anchor: starting values
    SELECT 1 AS counter, 100 AS value

    UNION ALL

    -- Recursive: modify previous values directly
    SELECT
        counter + 1,        -- increment counter
        value * 2           -- double the previous value (CARRY FORWARD + MODIFY)
    FROM cte
    WHERE counter < 5
)
SELECT * FROM cte;
```

**Key insight:** `value` in the recursive term IS the previous row's value. No SUM() OVER, no POWER() — just `value * 2`.

---

## Task 1: Factorial Sequence

**Scenario:**
Generate factorials from 1! to 7!: 1, 2, 6, 24, 120, 720, 5040

**Expected Output Columns:**
- `n` (integer) — 1, 2, 3, 4, 5, 6, 7
- `factorial` (bigint) — 1, 2, 6, 24, 120, 720, 5040

**Requirements:**
- Use WITH RECURSIVE
- Anchor: n=1, factorial=1
- Recursive: n+1, factorial * (n+1) ← **carry forward and multiply**
- Terminate at n=7

**Difficulty Rating:** 3/5

**Think about it:** factorial(n) = factorial(n-1) × n

---

## Task 2: Compound Interest Growth

**Scenario:**
An investment of $1000 grows at 10% annual interest. Show the balance after each year for 5 years.

**Expected Output Columns:**
- `year` (integer) — 0, 1, 2, 3, 4, 5
- `balance` (numeric) — 1000, 1100, 1210, 1331, 1464.10, 1610.51 (rounded to 2 decimals)

**Requirements:**
- Use WITH RECURSIVE
- Anchor: year=0, balance=1000
- Recursive: year+1, balance * 1.10 ← **carry forward and multiply by growth factor**
- Terminate at year=5
- Round balance to 2 decimals

**Difficulty Rating:** 3/5

---

## Task 3: Fibonacci Sequence

**Scenario:**
Generate the first 10 Fibonacci numbers: 1, 1, 2, 3, 5, 8, 13, 21, 34, 55

**Expected Output Columns:**
- `position` (integer) — 1 through 10
- `fib_value` (integer) — the Fibonacci number at that position

**Requirements:**
- Use WITH RECURSIVE
- Anchor: position=1, fib_value=1, prev_value=0
- Recursive: position+1, fib_value + prev_value (new fib), fib_value (becomes new prev)
- Terminate at position=10

**Difficulty Rating:** 4/5

**Hint:** You need to track TWO values: current and previous. Each iteration:
- new_fib = current + previous
- new_prev = current (the old current becomes the new previous)

```sql
SELECT
    position + 1,
    fib_value + prev_value AS fib_value,  -- new = current + previous
    fib_value AS prev_value               -- old current becomes new previous
```

---

## Submission Instructions

Today's tasks all use the **carry forward** pattern:
1. Task 1 — Multiply by incrementing number (factorial)
2. Task 2 — Multiply by constant factor (compound interest)
3. Task 3 — Carry TWO values forward (Fibonacci)

No window functions needed. No POWER() or FACTORIAL() built-ins. Just reference the previous value and modify it.

Good luck!
