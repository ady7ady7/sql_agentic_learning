## Weekly Summary — Week 21 (2026-05-04 to 2026-05-08)

### Scores
| Day | Focus | Score |
|-----|-------|-------|
| Day 1 | job_db exploration: platform/seniority + city dominance + VALUES CROSS JOIN | 28/30 |
| Day 2 | VALUES CROSS JOIN repeat + salary parsing + platform share | 25/30 |
| Day 3 | Work type breakdown + VALUES CROSS JOIN 3rd rep + cumulative offers | 29/30 |
| Day 4 | NOT EXISTS + YoY LAG(12) + dominant work type | 24/30 |
| Day 5 | NOT EXISTS drill + NTILE + self-join co-occurrence | 29/30 |

**Week total: 135/150. Strongest week yet in terms of consistency.**

---

### Key Wins

- **VALUES + CROSS JOIN pattern locked in** — three reps across Days 1-3, fully independent by Day 3. `COUNT FILTER (WHERE ILIKE '%' || keyword || '%')` is now natural.
- **Dominant type pattern solid** — GROUP BY → RANK → filter rank=1, applied independently on job_db data.
- **NOT EXISTS finally correct** — Day 5 clean execution after persistent direction errors. Root cause identified: drilling three anti-join variants at once created confusion. One pattern, one rep going forward.
- **Cumulative SUM with UNBOUNDED PRECEDING** — applied correctly on new data independently.
- **YoY LAG(12)** — clean on job_db data, NULL propagation correct.
- **job_db integrated** — comfortable exploring a new schema, adapting to text fields, NULLs, and Polish naming.

---

### Focus Areas

- **NOT EXISTS direction** — failed on Day 4 (inverted + impossible condition), fixed on Day 5. Needs one more rep next week to fully cement.
- **Salary parsing (REGEXP_MATCH)** — needed scaffolding, genuinely complex. Not a priority to drill — data modeling problem more than SQL skill gap.
- **Self-join deduplication** — used `>` in WHERE instead of `<` in JOIN ON. Minor efficiency point worth remembering.

---

### Agent Notes

- Drilling three anti-join variants simultaneously caused confusion — confirmed by student. Going forward: NOT EXISTS only.
- REGEXP tasks should be avoided unless student specifically requests them — too complex for the learning return.

---

### Week 22 Plan

- **NOT EXISTS one more rep** — cement the direction permanently
- **Type B recursive CTE** — still pending a self-referencing table; discuss adding one or move on
- **PERCENT_RANK + STDDEV** on job_db — apply window functions to new schema
- **Cohort LEFT JOIN pattern** — proper date range version, still not done cleanly
- **Consider second dataset** — job_db is ~1300 rows, limited for scale-based analysis
