## Agent Feedback on Student — Week 18 Day 5 (2026-04-17)

**Task 1 (Query Optimization Rewrite):** All three correlated subqueries eliminated in one GROUP BY query. EXPLAIN ANALYZE comparison included — 8.6ms → 0.3ms, buffers 127 → 29. Went beyond the spec and measured it. 10/10.

**Task 2 (Inactivity Gap Detection):** LAG used instead of LEAD — gap is backward-looking (ends at current row) instead of forward-looking (starts from current row). Output columns semantically wrong as a result. count_sessions > 0 filter missing. HAVING >= 14 vs spec's > 14 (strict). 8/10.

**Task 3 (YoY LAG(12)):** LAG(total_revenue, 12) correct. yoy_diff NULL propagation correct. GROUP BY YEAR, MONTH redundant — MONTH already contains year information, GROUP BY MONTH alone is sufficient. 9/10.

**Day Score: 27/30**
