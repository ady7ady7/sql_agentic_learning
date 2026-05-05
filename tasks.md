# Daily SQL Practice Tasks

**Generated:** 2026-05-05
**Week 21, Day 2 Focus:** VALUES CROSS JOIN repeat + salary text parsing + platform share per seniority

---

## Task 1: VALUES + CROSS JOIN — Contract Type per Seniority

**Scenario:**
The team wants to know how often each contract type (`'B2B'`, `'Permanent'`, `'Mandate contract'`) appears for each seniority level.

For each seniority + contract type combination show:
- `seniority` — nazwa from seniority
- `contract_type` — the contract type keyword
- `offer_count` — number of offers where `umowa ILIKE '%keyword%'` for that seniority

Use a `VALUES` clause for the contract type list and `CROSS JOIN` to `oferty`. Exclude NULL seniority_id.

**Tables:** `job_db.oferty`, `job_db.seniority`

**Order by:** `seniority ASC`, `offer_count DESC`

**Difficulty Rating:** 3/5

WITH keywords AS (
SELECT keyword
FROM ( 
	VALUES ('B2B'), ('Permanent'), ('Mandate') 
) AS t(keyword)
)
SELECT 
	o.seniority_id,
	k.keyword AS contract_type,
	COUNT(*) FILTER (WHERE o.umowa ILIKE '%' || k.keyword || '%') AS offer_count
FROM keywords k
CROSS JOIN job_db.oferty o
GROUP BY o.seniority_id, k.keyword
ORDER BY seniority_id, offer_count DESC


---

## Task 2: Salary Parsing — Average Salary Floor by Seniority

**Scenario:**
The `zarobki` field is messy text, but rows containing `'PLN/month'` follow a pattern like `'14 400 - 17 600 PLN/month'`. The team wants to extract the lower bound of the salary range and average it per seniority.

For each seniority show:
- `seniority` — nazwa from seniority
- `avg_salary_floor` — average of the lower bound salary, rounded to 0 decimals

**To extract the lower bound:**
1. Filter to rows where `zarobki ILIKE '%PLN/month%'`
2. Use `SPLIT_PART(zarobki, ' - ', 1)` to get the part before ` - `
3. Use `REGEXP_REPLACE(..., '[^0-9]', '', 'g')` to strip non-numeric characters
4. Cast to integer

**Tables:** `job_db.oferty`, `job_db.seniority`

**Requirements:**
- Exclude NULL seniority_id
- Order by `avg_salary_floor DESC`

**Difficulty Rating:** 4/5

SELECT 
	seniority_id,
	round(AVG(REPLACE((REGEXP_MATCH(zarobki, '(\d[\d\s]+)'))[1], ' ', '')::NUMERIC), 2) AS avg_salary_floor
FROM job_db.oferty o
WHERE seniority_id IS NOT null
GROUP BY seniority_id
ORDER BY avg_salary_floor DESC


---

## Task 3: Platform Share — Dominance per Seniority

**Scenario:**
For each seniority level, which platform has the highest share of offers? Show all platform/seniority combinations with their percentage share.

For each combination show:
- `seniority` — nazwa from seniority
- `platform` — nazwa from platforma
- `offer_count`
- `total_for_seniority` — total offers for that seniority across all platforms
- `share_pct` — `offer_count / total_for_seniority * 100`, rounded to 1 decimal

Exclude NULL seniority_id and NULL platforma_id. Order by `seniority ASC`, `share_pct DESC`.

**Tables:** `job_db.oferty`, `job_db.platforma`, `job_db.seniority`

**Requirements:**
- Use `SUM(offer_count) OVER (PARTITION BY seniority)` to get `total_for_seniority`

**Difficulty Rating:** 5/5

WITH seniorities_platforms_counts AS (
SELECT 
	o.seniority_id,
	o.platforma_id,
	COUNT(*) AS offer_count
FROM job_db.oferty o
WHERE o.seniority_id IS NOT NULL
GROUP BY o.seniority_id , o.platforma_id
),
seniorities_counts AS (
SELECT 
	*,
	SUM(offer_count) OVER (PARTITION BY seniority_id) AS total_for_seniority
FROM seniorities_platforms_counts 
)
SELECT 
	p.nazwa AS seniority,
	s.nazwa AS platform,
	offer_count,
	total_for_seniority,
	ROUND((offer_count / total_for_seniority)::NUMERIC * 100, 1) AS share_pct
FROM seniorities_counts sc
JOIN job_db.platforma p ON p.id = sc.platforma_id
JOIN job_db.seniority s ON s.id = sc.seniority_id
ORDER BY seniority_id, share_pct DESC


---

## Submission Instructions

1. Task 1 — VALUES + CROSS JOIN contract types per seniority (3/5)
2. Task 2 — Salary floor extraction + average per seniority (4/5)
3. Task 3 — Platform share % per seniority with window SUM (5/5)
