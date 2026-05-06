# Daily SQL Practice Tasks

**Generated:** 2026-05-06
**Week 21, Day 3 Focus:** GROUP BY multi-dimension + VALUES CROSS JOIN 3rd rep + cumulative SUM on job data

---

## Task 1: Offer Count by Work Type per Platform

**Scenario:**
The team wants a breakdown of how each platform distributes offers across work types (Hybrid, Remote, Stationary etc.).

For each platform + work type combination show:
- `platform` — nazwa from platforma
- `typ` — work type
- `offer_count`

Exclude NULL platforma_id and NULL typ. Order by `platform ASC`, `offer_count DESC`.

**Tables:** `job_db.oferty`, `job_db.platforma`

**Difficulty Rating:** 3/5


SELECT 
	p.nazwa AS platform,
	o.typ,
	COUNT(*) AS offer_count
FROM job_db.oferty o
JOIN job_db.platforma p ON o.platforma_id = p.id
WHERE o.platforma_id IS NOT NULL AND o.typ IS NOT NULL
GROUP BY p.nazwa, o.typ
ORDER BY platform, offer_count DESC

---

## Task 2: VALUES + CROSS JOIN — Work Type Prevalence by Seniority

**Scenario:**
The team wants to know how often each work type (`'Hybrid'`, `'Remote'`, `'Stationary'`) appears for each seniority level.

For each seniority + work type combination show:
- `seniority` — nazwa from seniority
- `work_type` — the work type keyword
- `offer_count` — number of offers where `typ ILIKE '%keyword%'` for that seniority

Use a `VALUES` clause for the work type list, `CROSS JOIN` to `oferty`, and JOIN to `seniority` for the name. Exclude NULL seniority_id. Order by `seniority ASC`, `offer_count DESC`.

**Tables:** `job_db.oferty`, `job_db.seniority`

**Difficulty Rating:** 4/5


WITH keywords AS (
SELECT keyword FROM (
	VALUES ('Hybrid'), ('Remote'), ('Statoniary')
) AS t(keyword)
)
SELECT 
	k.keyword,
	s.nazwa AS seniority,
	COUNT(*) FILTER (WHERE o.typ ILIKE '%' || k.keyword || '%') AS offer_count
FROM keywords k
CROSS JOIN job_db.oferty o
JOIN job_db.seniority s ON s.id = o.seniority_id
GROUP BY k.keyword, s.nazwa
ORDER BY seniority, offer_count DESC



---

## Task 3: Cumulative Offers Over Time per Platform

**Scenario:**
The team wants to see how offers accumulated over time for each platform — a running total of offers by listing date.

For each platform + date combination show:
- `platform` — nazwa from platforma
- `data_wystawienia`
- `daily_offers` — number of offers listed on that date for that platform
- `cumulative_offers` — running total of offers from the earliest date up to and including this date, per platform

Exclude NULL `data_wystawienia` and NULL `platforma_id`. Order by `platform ASC`, `data_wystawienia ASC`.

**Tables:** `job_db.oferty`, `job_db.platforma`

**Requirements:**
- Use `SUM(daily_offers) OVER (PARTITION BY platform ORDER BY data_wystawienia ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)`

**Difficulty Rating:** 5/5

WITH platform_counts AS (
SELECT 
	p.nazwa AS platform,
	o.data_wystawienia,
	COUNT(*) AS daily_offers
FROM job_db.oferty o
JOIN job_db.platforma p ON o.platforma_id = p.id
WHERE o.data_wystawienia IS NOT NULL AND o.platforma_id IS NOT NULL
GROUP BY p.nazwa, o.data_wystawienia
ORDER BY data_wystawienia
)
SELECT 
	*,
	SUM(daily_offers) OVER (PARTITION BY platform ORDER BY data_wystawienia ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cumulative_offers
FROM platform_counts pc
ORDER BY platform, data_wystawienia

---

## Submission Instructions

1. Task 1 — Work type breakdown per platform (3/5)
2. Task 2 — VALUES + CROSS JOIN work types per seniority (4/5)
3. Task 3 — Cumulative offers over time per platform (5/5)
