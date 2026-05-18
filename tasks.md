# Daily SQL Practice Tasks

**Generated:** 2026-05-18
**Week 23, Day 1 Focus:** GROUP BY + HAVING on job_db + MoM LAG per user

---

## Task 1: GROUP BY + HAVING — High-Volume Platforms by Contract Type

**Scenario:**
The team wants to see contract type distribution, but only for platforms that have more than 100 offers total.

For each qualifying platform + contract type combination show:
- `platform` — nazwa from platforma
- `umowa` — contract type
- `offer_count`

Exclude NULL platforma_id and NULL umowa. Only include platforms where total offer count > 100. Order by `platform ASC`, `offer_count DESC`.

**Tables:** `job_db.oferty`, `job_db.platforma`

**Difficulty Rating:** 3/5

SELECT 
	p.nazwa AS platform,
	o.umowa,
	COUNT(*) AS offer_count
FROM job_db.oferty o
JOIN job_db.seniority s ON o.seniority_id = s.id
JOIN job_db.platforma p ON p.id = o.platforma_id
WHERE P.NAZWA IS NOT NULL AND umowa IS NOT null
GROUP BY p.nazwa, o.umowa
HAVING COUNT(*) >= 100
ORDER BY platform, offer_count DESC

---

## Task 2: LAG — Month-over-Month Order Revenue per User

**Scenario:**
For each user, show their monthly order revenue alongside the previous month's revenue — to spot individual growth or decline.

For each user + month show:
- `user_id`
- `month` — DATE_TRUNC to month
- `monthly_revenue` — total order amount for that user that month
- `prev_month_revenue` — previous month's revenue for that user via LAG(1)
- `mom_diff` — `monthly_revenue - prev_month_revenue` (NULL if no prior month)

**Tables:** `crappy_data_db.orders`

**Requirements:**
- Exclude NULL amounts
- Order by `user_id ASC`, `month ASC`

**Difficulty Rating:** 3/5

WITH ORDERS_MONTHS AS (
SELECT
	*,
	DATE_TRUNC('Month', created_at) AS month
FROM crappy_data_db.orders o
),
users_monthly_revs AS (
SELECT 
	USER_ID,
	MONTH,
	SUM(amount) AS monthly_revenue
FROM orders_months
GROUP BY user_id, MONTH
),
users_pmonth_revs AS (
SELECT 
	*,
	LAG(monthly_revenue) OVER (PARTITION BY user_id ORDER BY MONTH) AS prev_month_revenue
FROM users_monthly_revs
)
SELECT 
	*,
	monthly_revenue - prev_month_revenue AS mom_diff
FROM users_pmonth_revs

---

## Submission Instructions

1. Task 1 — High-volume platforms by contract type (3/5)
2. Task 2 — MoM order revenue per user with LAG(1) (3/5)
