# Daily SQL Practice Tasks

**Generated:** 2026-05-07
**Week 21, Day 4 Focus:** NOT EXISTS on job_db + YoY offer count LAG(12) + dominant work type per platform via RANK

---

## Task 1: Anti-Join — Seniority Levels With No Salary Data

**Scenario:**
The team wants to know which seniority levels have zero offers with any salary information — i.e. every offer for that seniority either has NULL in `zarobki` or doesn't contain `'PLN'` at all.

Show:
- `seniority_id`
- `seniority` — nazwa from seniority

Use `NOT EXISTS` starting from `job_db.seniority`, checking against `job_db.oferty`.

**Tables:** `job_db.seniority`, `job_db.oferty`

**Order by:** `seniority_id ASC`

**Difficulty Rating:** 3/5

SELECT
distinct
	S.id AS seniority_id,
	s.nazwa AS seniority
FROM job_db.oferty o
JOIN job_db.seniority s ON o.seniority_id = s.id
WHERE NOT EXISTS (SELECT s2.id  FROM job_db.seniority s2
					WHERE s2.id = o.seniority_id 
					AND o.zarobki IS NULL AND o.zarobki LIKE '%PLN%')
ORDER BY seniority_id

---

## Task 2: YoY — Monthly Offer Count 2024 vs 2025

**Scenario:**
The team wants to compare how many offers were listed each month in 2025 versus the same month in 2024.

For each month show:
- `month` — DATE_TRUNC to month
- `offer_count` — number of offers listed that month
- `prev_year_count` — offer count for the same month one year prior via `LAG(offer_count, 12)`
- `yoy_diff` — `offer_count - prev_year_count` (NULL if no prior year data)

**Tables:** `job_db.oferty`

**Requirements:**
- Exclude NULL `data_wystawienia`
- Order by `month ASC`

**Difficulty Rating:** 4/5

WITH offers_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', data_wystawienia) AS month_
FROM job_db.oferty o
),
monthly_counts AS (
SELECT 
	month_,
	COUNT(*) AS offer_count
FROM offers_months
WHERE month_ IS NOT null
GROUP BY month_
),
monthly_counts_prev_year AS (
SELECT 
	*,
	lag(offer_count, 12) OVER (ORDER BY month_) AS prev_year_count
FROM monthly_counts
)
SELECT 
	*,
	offer_count - prev_year_count  AS yoy_diff
FROM monthly_counts_prev_year


---

## Task 3: Dominant Work Type per Platform

**Scenario:**
For each platform, find the work type (`typ`) that appears most often — the dominant work type.

Show:
- `platform` — nazwa from platforma
- `dominant_typ`
- `offer_count` — count of offers with that work type for that platform

**Tables:** `job_db.oferty`, `job_db.platforma`

**Requirements:**
- CTE 1: GROUP BY platforma_id + typ to get counts
- CTE 2: RANK() OVER (PARTITION BY platforma_id ORDER BY offer_count DESC) to rank types per platform
- Final SELECT: filter to rank = 1, JOIN to platforma for the name
- Exclude NULL platforma_id and NULL typ
- Order by `platform ASC`

**Difficulty Rating:** 5/5


WITH platform_types_cnts AS (
SELECT 
	p.nazwa AS platform,
	o.typ,
	COUNT(*) AS offer_count
FROM job_db.oferty o
JOIN job_db.platforma p ON o.platforma_id = p.id
WHERE P.nazwa IS NOT NULL AND O.typ IS NOT NULL
GROUP BY p.nazwa, o.typ
),
platform_type_ranks AS (
SELECT 
	*,
	rank() OVER (PARTITION BY platform ORDER BY offer_count DESC) AS rank
FROM platform_types_cnts
)
SELECT 
	platform,
	typ AS dominant_type,
	offer_count
FROM platform_type_ranks
WHERE RANK = 1
ORDER BY platform


---

## Submission Instructions

1. Task 1 — NOT EXISTS: seniority levels with no salary data (3/5)
2. Task 2 — YoY monthly offer count with LAG(12) (4/5)
3. Task 3 — Dominant work type per platform via RANK (5/5)
