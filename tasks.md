# Daily SQL Practice Tasks

**Generated:** 2026-05-22
**Week 23, Day 5 Focus:** Light Friday — dominant_type RANK fix + NULLIF denominator + weekly recap

---

## Task 1: dominant_type — RANK Tie Fix

**Scenario:**
Same as yesterday: for each user, find their dominant transaction type. If there's a tie, return all tied types.

This time use `RANK()` (not ROW_NUMBER) so ties are preserved. Also put the NULL filters in the first CTE, before aggregation.

Show:
- `user_id`
- `dominant_type`
- `type_count`

Exclude NULL user_id and NULL type. Order by `user_id ASC`, `type_count DESC`.

**Tables:** `crappy_data_db.transactions`

**Difficulty Rating:** 3/5

---

## Task 2: NULLIF — Safe Avg with Zero-Value Orders

**Scenario:**
For each user, calculate average order value. Exclude NULL amounts (they are invalid). Keep zero-amount orders (zeros are valid). Guard against division by zero with NULLIF on the denominator only.

Show:
- `user_id`
- `valid_order_count` — COUNT of non-NULL amounts
- `total_revenue` — SUM of non-NULL amounts (NULL if none)
- `avg_order_value` — `total_revenue / NULLIF(valid_order_count, 0)`, rounded to 2

Order by `user_id ASC`.

**Tables:** `crappy_data_db.orders`

**Difficulty Rating:** 3/5

---

## Submission Instructions

1. Task 1 — dominant_type with RANK (3/5)
2. Task 2 — NULLIF on denominator only (3/5)
