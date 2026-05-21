## Week 23, Day 4 — Agent Feedback on Student

**Score: 26/30**

**Task 1 (dominant_type):** Two-CTE pattern correct. Used ROW_NUMBER() instead of RANK() — breaks tie-inclusion since ROW_NUMBER arbitrarily picks one winner. NULL filters in second CTE instead of first (minor — should filter before aggregation). 8/10.

**Task 2 (PERCENT_RANK):** Clean. Correct partition/order, ROUND with NUMERIC cast, NULL filters in WHERE. 10/10.

**Task 3 (NULLIF):** NULLIF(amount, 0) applied to data values — treats zeros as invalid, which is wrong. Zeros are valid order amounts. NULLIF belongs only on the denominator: total_revenue / NULLIF(valid_order_count, 0). COUNT and SUM naturally ignore NULLs. 8/10.

---

## Student Feedback on Questions

Session complete.
