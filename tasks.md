# Daily SQL Practice Tasks

**Generated:** 2026-02-02
**Week 8, Day 4 Focus:** 3-Level Hierarchy Practice + Path Building

---

## Revised Plan

Day 3 showed the 3-level pattern needs more practice. Today: same pattern, different data, plus path building introduction.

---

## Task 1: 3-Level Hierarchy — Continent → Country → City

**Scenario:**
Build a 3-level hierarchy with geography:
- Level 1: 'World'
- Level 2: 'Europe', 'Asia'
- Level 3: 'Germany', 'France' (under Europe), 'Japan', 'China' (under Asia)

**Expected Output:**
```
level | name    | parent_name
------+---------+------------
1     | World   | NULL
2     | Europe  | World
2     | Asia    | World
3     | Germany | Europe
3     | France  | Europe
3     | Japan   | Asia
3     | China   | Asia
```

**Requirements:**
- Mapping CTE with all parent-child pairs
- Single anchor row ('World')
- `WHERE h.level < 3`

**Difficulty Rating:** 3/5

This is the SAME pattern as Day 3 Task 1. Write it from scratch without looking back.

---

## Task 2: 3-Level with Path Building — Add Full Path

**Scenario:**
Take Task 1 and add a `path` column showing the full hierarchy path.

**Expected Output:**
```
level | name    | parent_name | path
------+---------+-------------+--------------------
1     | World   | NULL        | World
2     | Europe  | World       | World > Europe
2     | Asia    | World       | World > Asia
3     | Germany | Europe      | World > Europe > Germany
3     | France  | Europe      | World > Europe > France
3     | Japan   | Asia        | World > Asia > Japan
3     | China   | Asia        | World > Asia > China
```

**Requirements:**
- Same structure as Task 1
- Add `path` column using string concatenation (`||`)
- Anchor path = just the name
- Recursive path = parent's path || ' > ' || child's name

**Difficulty Rating:** 3/5

**Hint:** In anchor: `'World' AS path`. In recursive: `h.path || ' > ' || m.child AS path`

---

## Task 3: Advanced — NTILE for Quartile Bucketing

**Scenario:**
Divide users into 4 equal groups (quartiles) based on their total order amounts.

**Expected Output Columns:**
- `user_id` (integer)
- `total_spent` (numeric) — sum of order amounts
- `quartile` (integer) — 1 (lowest spenders) to 4 (highest spenders)

**Requirements:**
- Aggregate orders by user_id
- Use `NTILE(4)` window function
- Order by total_spent to assign quartiles
- Final output ordered by quartile, then total_spent DESC

**Difficulty Rating:** 4/5

**Note:** `NTILE(n)` divides rows into n roughly equal buckets numbered 1 to n.

---

## Week 8 Scaffolding Plan (Revised)

| Day | Focus | Status |
|-----|-------|--------|
| Day 1 | 2-level hierarchy with mapping CTE | ✓ Done |
| Day 2 | 2-level with real table data | ✓ Done |
| Day 3 | 3-level hierarchy introduction | ✓ Done (struggled) |
| Day 4 | 3-level practice + path building (TODAY) | |
| Day 5 | 3-level with simple real data | |

---

## Submission Instructions

1. Task 1 — 3-level hardcoded, same pattern (3/5)
2. Task 2 — Add path column (3/5)
3. Task 3 — NTILE quartile bucketing (4/5)

Goal: Lock in the 3-level pattern through repetition.

