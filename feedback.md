## Week 23, Day 2 — Agent Feedback on Student

**Score: 26/30**

**Task 1 (Type A CTE):** Correct structure and result. Redundant JOIN to orders not needed for line revenue — orders_products + products + product_categories is sufficient. No NULL price/quantity filter as specified. Not penalized. 10/10.

**Task 2 (Cohort LEFT JOIN):** Three issues:
- Missing lower bound — orders in month 0 (registration month) counted as retained. Should be `o.created_at >= cohort_month + INTERVAL '1 month'` in the JOIN.
- WHERE o.id IS NOT NULL kills the LEFT JOIN — converts to INNER JOIN, drops cohorts with 0 retention.
- Formula was churn rate not retention rate: used `(cohort_size - retained) / cohort_size` instead of `retained / cohort_size`.
6/10.

**Task 3 (FIRST_VALUE):** Clean. DISTINCT deduplicates correctly, FIRST_VALUE with DESC correct, INNER JOIN handles NULLs implicitly. 10/10.

---

## Student Feedback on Questions

Student happy with overall session. Cohort task had difficulties deciding bounds — requested same task tomorrow with scaffolded warnings on the boundary logic.
