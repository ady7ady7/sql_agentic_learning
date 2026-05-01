## Weekly Summary — Week 20 (2026-04-27 to 2026-05-01)

### Scores
| Day | Focus | Score |
|-----|-------|-------|
| Day 1 | Type A CTE drill + NTILE + EXPLAIN ANALYZE | 30/30 |
| Days 2+3 | FIRST_VALUE + rolling SUM + anti-join + LAG offset + YoY + NULLIF | 51/60 |
| Day 4 | NOT EXISTS anti-join + YoY NULLIF + window frame comparison | 29/30 |
| Day 5 | PERCENT_RANK + cohort retention | 18/20 |

**Week total: ~128/140. Strong week with one double session to catch up.**

---

### Key Wins

- **Type A CTE finally natural** — Day 1 was clean, no recursion, no self-join. Pattern is locked.
- **NULLIF in division** — properly applied `NULLIF(lag(revenue,12), 0)` in YoY denominator on Day 4 after missing it on Days 2+3. Learned from the correction.
- **Window frame specs** — `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW`, `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` all in one query. Solid understanding.
- **NTILE** — new function, applied cleanly first time.
- **NOT EXISTS direction** — correct FROM/WHERE direction on Day 4 after the inversion on Days 2+3.
- **Creative solutions** — second-order ROW_NUMBER approach for cohort retention, boolean expressions instead of CASE WHEN. Good instincts.

---

### Focus Areas

- **Anti-join direction** — inverted on Days 2+3 (queried from orders_products instead of products). Corrected on Day 4. Worth one more drill early next week.
- **NULLIF in YoY** — missed on Day 5 task, caught on Day 4 retry. Now solid.
- **Interval precision** — cohort task used `INTERVAL '1 MONTH'` instead of `'4 months'` for a 3-month window. Read the spec carefully on time ranges.

---

### Agent Notes

- Proposed Type B recursive CTE tasks twice with fake schema — saved to memory, will not repeat until real self-referencing table exists.
- Student requested settling on NOT EXISTS as the default anti-join pattern going forward.
- Schema expansion planned for next week — student has premium course datasets to choose from, including a large smartphone sales dataset.

---

### Week 21 Plan

- **Schema expansion** — add new dataset (student to choose and share CREATE TABLE + INSERT)
- **Anti-join one more drill** — correct direction, NOT EXISTS only
- **Cohort LEFT JOIN pattern** — the spec version with explicit date range, not yet done cleanly
- **STDDEV + window functions in new schema** — apply learned patterns to fresh data
- **Query optimization** — EXPLAIN ANALYZE on more complex queries, index scan vs seq scan discussion
