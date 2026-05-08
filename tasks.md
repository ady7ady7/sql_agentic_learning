# Daily SQL Practice Tasks

**Generated:** 2026-05-08
**Week 21, Day 5 Focus:** NOT EXISTS clean drill + NTILE on offer counts + self-join platform co-occurrence

---

## Task 1: NOT EXISTS — Platforms With No Hybrid Offers

**Scenario:**
The team wants to know which platforms have no Hybrid offers at all.

Use `NOT EXISTS`. Start from `job_db.platforma`, check against `job_db.oferty`.

**Expected Output Columns:** `platform_id`, `platform`

**Tables:** `job_db.platforma`, `job_db.oferty`

**Order by:** `platform_id ASC`

**Difficulty Rating:** 3/5


SELECT * 
FROM job_db.platforma p
WHERE NOT EXISTS
(SELECT
o.platforma_id FROM job_db.oferty o
WHERE o.platforma_id = p.id AND o.typ ILIKE '%Hybrid%' 
)

---

## Task 2: NTILE — Seniority Tiers by Offer Volume

**Scenario:**
The team wants to rank seniority levels into 4 tiers based on how many offers exist for each level — from least to most in-demand.

For each seniority level show:
- `seniority` — nazwa from seniority
- `offer_count`
- `tier` — 1 (lowest volume) to 4 (highest volume) using `NTILE(4)`
- `tier_label` — `'low'` for tier 1, `'mid-low'` for tier 2, `'mid-high'` for tier 3, `'high'` for tier 4

Exclude NULL seniority_id. Order by `offer_count DESC`.

**Tables:** `job_db.oferty`, `job_db.seniority`

**Difficulty Rating:** 4/5

WITH seniorities_counts AS (
SELECT 
	s.nazwa,
	COUNT(*) AS offer_count
FROM job_db.oferty o
JOIN job_db.seniority s ON o.seniority_id = s.id
WHERE s.id IS NOT NULL
GROUP BY s.nazwa
),
seniorities_tiers AS (
SELECT 
	*,
	NTILE(4) OVER (ORDER BY offer_count) AS TIER
FROM seniorities_counts
)
SELECT 
	*,
	CASE 
		WHEN tier = 1 THEN 'low'
		WHEN tier = 2 THEN 'mid-low'
		WHEN tier = 3 THEN 'mid-high' ELSE 'high'
	END AS tier_label
FROM seniorities_tiers
ORDER BY offer_count DESC


---

## Task 3: Self-Join — Same Position on Multiple Platforms

**Scenario:**
The team wants to find job positions that appear on more than one platform — to identify roles with broad cross-platform demand.

For each pair of offers with the same `pozycja` but different `platforma_id`, show:
- `pozycja`
- `platform_1` — nazwa of the first platform
- `platform_2` — nazwa of the second platform

Use a self-join on `oferty`. To avoid duplicates, enforce `o1.platforma_id < o2.platforma_id`. Exclude NULL pozycja and NULL platforma_id.

Order by `pozycja ASC`, `platform_1 ASC`.

**Tables:** `job_db.oferty`, `job_db.platforma`

**Difficulty Rating:** 5/5

SELECT
	o1.pozycja,
	p1.nazwa AS platform_1,
	p2.nazwa AS platform_2
FROM job_db.oferty o1
JOIN job_db.oferty o2 ON o1.pozycja = o2.pozycja AND o1.platforma_id != o2.platforma_id
JOIN job_db.platforma p1 ON o1.platforma_id = p1.id
JOIN job_db.platforma p2 ON o2.platforma_id = p2.id
WHERE o1.platforma_id > o2.platforma_id
AND O1.platforma_id IS NOT NULL AND O2.platforma_id IS NOT NULL AND o1.pozycja IS NOT NULL
ORDER BY pozycja, platform_1

Not that difficult honestly, but a useful thing to remember


---

## Submission Instructions

1. Task 1 — NOT EXISTS: platforms with no Hybrid offers (3/5)
2. Task 2 — NTILE seniority tiers by offer volume (4/5)
3. Task 3 — Self-join same position on multiple platforms (5/5)
