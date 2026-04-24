## Weekly Summary — Week 19 (2026-04-20 to 2026-04-24)

### Scores
| Day | Focus | Score |
|-----|-------|-------|
| Day 1 | Anti-join + conditional aggregation | 2 tasks (light session) |
| Day 2 | dominant_type via RANK + Type A rollup + NULLIF | 25/30 |
| Day 3 | Type A UNION ALL + PERCENT_RANK + monthly streaks | 26/30 |
| Day 4 | LAG between orders + self-join + cohort retention | 29/30 |
| Day 5 | STDDEV + ticket response time + funnel analysis | 30/30 |

**Week trajectory: improving every day, finished with a perfect score.**

---

### Key Wins

- **dominant_type pattern locked in** — GROUP BY user+type → RANK() → filter rank=1. Clean, natural, no unpivoting needed. You also added bonus per-type counts on your own initiative.
- **Gaps-and-islands solid** — SUM(is_new_streak) as running group ID, COUNT(DISTINCT month) to handle duplicates. Pattern is internalized.
- **Data-aware adaptation** — consistently checking actual data before applying specs (NULLIF not needed, minutes not hours, pending = first delivery status). This is professional-level instinct, keep it.
- **EPOCH time calculations** — first use of `EXTRACT(EPOCH FROM ...)` on real chat data. Clean execution.
- **Funnel analysis** — three-step CTE + UNION ALL funnel pattern understood and executed correctly.
- **29 and 30/30 to close the week** — strong finish despite a busy, draining work week.

---

### Focus Areas

- **Type A fixed-hierarchy CTE pattern** — used `WITH RECURSIVE` + self-join on Days 2 and 3 instead of plain UNION ALL. The pattern: three independent aggregation CTEs, then a flat UNION ALL — no recursion, no joining back. This needs to click as muscle memory.
- **Task instruction quality** — a few tasks this week had ambiguous wording (first message vs first response, funnel step labels). Called out correctly each time. Agent needs to tighten task specs.

---

### Agent Notes (self-correction)

- Proposed tasks with non-existent schema columns (Type B recursive CTE with fake `parent_id`) twice — now saved to memory, will not repeat.
- Penalized correct solutions based on misreading data context (LAG direction, NULL amounts, pending status logic) — pushback was valid every time.
- Task wording mismatches between scenario description and output spec — needs stricter internal consistency check before publishing tasks.

---

### Week 20 Plan

**Concepts to rotate in:**
- Type A CTE pattern — one more clean drill until it's automatic
- Type B recursive CTE — needs a real self-referencing table first; worth discussing adding one to the schema
- Query optimization — EXPLAIN ANALYZE reading, seq scan vs index scan (not covered since Week 18 Day 5)
- Window functions: `FIRST_VALUE`, `NTILE`, cumulative SUM with custom frame specs — underused this week
- YoY / period-over-period comparisons — not revisited since Week 18

**Schema expansion discussion:** Student flagged that the current schema limits certain patterns (Type B recursive, more complex join graphs). Consider adding: an `employees` table with `manager_id` self-reference, or a `product_categories` hierarchy with `parent_id`.
