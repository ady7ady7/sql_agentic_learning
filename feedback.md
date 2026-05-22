## Week 23, Day 5 — Agent Feedback on Student

**Score: 18/20**

**Task 1 (dominant_type RANK fix):** Clean. RANK() correct, ties preserved, pattern solid. 10/10.

**Task 2 (NULLIF denominator):** SUM(NULLIF(amount, 0)) still wrapping data values — zeros incorrectly excluded. Plain SUM(amount) is correct; SUM ignores NULLs naturally. NULLIF only belongs on the denominator. 8/10.

---

## Weekly Summary — Week 23 (2026-05-18 to 2026-05-22)

### Scores
| Day | Focus | Score |
|-----|-------|-------|
| Day 1 | GROUP BY + HAVING job_db + MoM LAG | 20/20 |
| Day 2 | Type A CTE + cohort LEFT JOIN + FIRST_VALUE | 26/30 |
| Day 3 | Cohort retention retry + cumulative SUM + cross-schema | 27/30 |
| Day 4 | dominant_type CTE + PERCENT_RANK + NULLIF | 26/30 |
| Day 5 | dominant_type RANK fix + NULLIF denominator | 18/20 |

**Week total: 117/130. Solid week with real learning on new patterns.**

---

### Key Wins

- **Cohort LEFT JOIN cracked** — boundary logic (>= month+1, < month+4), LEFT JOIN preserved, retention formula all correct on Day 3 retry. Pattern is now locked.
- **dominant_type pattern introduced and absorbed** — two-CTE aggregate-then-rank approach understood. RANK() vs ROW_NUMBER distinction clear by Day 5.
- **PERCENT_RANK on job_db** — clean execution, correct partition and order, no issues.
- **Cumulative SUM with explicit frame** — ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW correct first attempt.
- **Cross-schema JOIN** — city = miasto join between crappy_data_db and job_db working cleanly.
- **FIRST_VALUE** — correct usage with DESC sort for latest value, DISTINCT deduplication clean.

---

### Focus Areas

- **NULLIF on data vs denominator** — recurring mistake: wrapping SUM/COUNT values in NULLIF(col, 0) instead of using it only on denominators. Rule: `SUM(amount)` and `COUNT(amount)` ignore NULLs naturally. NULLIF belongs only in `/ NULLIF(count, 0)` to prevent division by zero. Missed on both Day 4 and Day 5.
- **ROW_NUMBER vs RANK for ties** — fixed on Day 5 after Day 4 slip. When the spec says "return all ties", always RANK().
- **COUNT(*) vs COUNT(DISTINCT col) on tables without PK** — on job_db.oferty (no PK), COUNT(DISTINCT pozycja) counts distinct titles not offers. Use COUNT(*) as offer count proxy.
- **Redundant JOINs** — Day 2 Task 1 included an unnecessary JOIN to orders for line revenue calculation.

---

### Week 24 Plan

- **NULLIF** — one more clean drill with a real dirty-data scenario, denominator-only usage
- **gaps-and-islands time proximity** — group events within 30-minute windows (session detection pattern), introduced as new concept #3
- **YoY comparison** — LAG(revenue, 12) OVER (ORDER BY month), hasn't appeared since Week 21
- **Type B recursive CTE** — still blocked on schema (no self-referencing table); defer until PK/parent_id added or use a different table
- **dominant_type** — one more rep in a different context (e.g. per platform or per city instead of per user)

---

## Student Feedback on Questions

Week complete.
