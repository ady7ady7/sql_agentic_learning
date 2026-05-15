## Weekly Summary — Week 22 (2026-05-11 to 2026-05-15)

### Scores
| Day | Focus | Score |
|-----|-------|-------|
| Day 1 | Country order stats + RANK within country + NOT EXISTS | 29/30 |
| Day 2 | EPOCH ticket resolution + cohort LEFT JOIN + STDDEV cross-table | 23/30 |
| Day 3 | STDDEV GROUP BY + PERCENT_RANK job_db + cross-schema JOIN | 29/30 |
| Day 4 | MoM LAG job_db + NTILE spend tiers + gaps-and-islands posting gaps | 29/30 |
| Day 5 | Light Friday — GROUP BY + LAG + NOT EXISTS refresh | 30/30 |

**Week total: 140/150. Consistent and strong, finished with a perfect score.**

---

### Key Wins

- **NOT EXISTS cemented** — correct direction all week, no errors. Pattern is fully locked.
- **STDDEV GROUP BY** — clean after the window function confusion from Day 2. One clear correction, one clean rep, done.
- **Gaps-and-islands on new schema** — independently solved on job_db with month granularity. EPOCH for month calculation correct approach.
- **Cross-schema JOIN** — first query joining crappy_data_db and job_db. Clean execution, good data awareness (no PK on oferty).
- **NTILE cross-table** — skipped unnecessary CTE, joined directly. Efficient thinking.
- **30/30 on Friday** — strong closer despite light tasks.

---

### Focus Areas

- **STDDEV window vs GROUP BY confusion** — mixing OVER with GROUP BY breaks. Rule: if you want one row per user, use `STDDEV(amount)` in GROUP BY. If you want per-row with all rows preserved, use `STDDEV(amount) OVER (PARTITION BY user_id)` without GROUP BY.
- **Cohort LEFT JOIN interval** — `<= 3 months` instead of `< 4 months` missed boundary. Small but worth keeping sharp.
- **Reading task specs carefully** — Day 2 Task 3 used orders instead of transactions. Data familiarity is good but specs still need a read.

---

### Agent Notes

- Student correctly challenges point deductions when data context justifies the approach — this is right and should continue.
- Cross-schema queries now viable — can mix job_db and crappy_data_db in same tasks going forward.

---

### Week 23 Plan

- **Cross-schema tasks** — more joins between job_db and crappy_data_db, interesting analytical angles
- **Type A CTE** — one more clean drill, hasn't appeared in a few weeks
- **Cohort analysis** — proper LEFT JOIN date range version, clean execution
- **Window functions depth** — FIRST_VALUE, cumulative SUM with different frames on job_db
- **Consider adding PK to job_db.oferty** — `ALTER TABLE job_db.oferty ADD COLUMN id SERIAL PRIMARY KEY` would enable cleaner counting
