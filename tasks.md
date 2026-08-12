# SQL Tasks — 2026-08-12 (Week 34, Day 3)

**Dataset:** job_db (platforma, seniority, oferty)  
**Focus:** Conditional aggregation pivot · Dirty text field analysis · RANK per group

---

## Task 1 — Oferty per Platforma × Seniority (Pivot)
**Difficulty: 3/5**

**Business question:**  
For each platform (`platforma.nazwa`), show how many job offers exist at each seniority level as separate columns. Use the 6 main platforms and the following seniority levels: Junior, Mid, Senior, Expert, Lead/Principal.

Use conditional aggregation with `FILTER` or `CASE WHEN` — one row per platform, one column per seniority level.

Also include a `total` column with the total offer count per platform. Order by `total DESC`.

**Expected output columns:**  
`platform, junior, mid, senior, expert, lead_principal, total`

**Difficulty: 3/5**

SELECT 
	p.nazwa AS platform,
	COUNT(*) FILTER (WHERE s.nazwa = 'Junior') AS junior,
	COUNT(*) FILTER (WHERE s.nazwa = 'Mid') AS mid,
	COUNT(*) FILTER (WHERE s.nazwa = 'Senior') AS senior,
	COUNT(*) FILTER (WHERE s.nazwa = 'Expert') AS expert,
	COUNT(*) FILTER (WHERE s.nazwa = 'Lead / Principal') AS lead_principal,
	COUNT(*) AS total
FROM job_db.oferty o
JOIN job_db.platforma p ON o.platforma_id = p.id
JOIN job_db.seniority s ON s.id = o.seniority_id 
GROUP BY p.nazwa
ORDER BY total DESC


---

## Task 2 — Dominująca Technologia per Platforma
**Difficulty: 4/5**

**Business question:**  
For each platform, find which of the following technologies appears most often in job listings: `Python`, `Java`, `SQL`, `JavaScript`, `AWS`.

A listing "contains" a technology if the `technologie` field contains that string (case-insensitive). One listing can count for multiple technologies.

For each platform, show all 5 technology counts and identify the dominant one (highest count). If there's a tie, show all tied technologies.

**Expected output columns (Part A — counts per platform):**  
`platform, python_cnt, java_cnt, sql_cnt, javascript_cnt, aws_cnt`

**Part B — dominant technology per platform:**  
`platform, dominant_tech, count`

Order Part B by `platform`.

**Difficulty: 4/5**

Part 1:

SELECT 
	p.nazwa AS platform,
	COUNT(*) FILTER (WHERE o.technologie LIKE ('%Python%')) AS python_cnt,
	COUNT(*) FILTER (WHERE o.technologie LIKE ('%Java%')) AS java_cnt,
	COUNT(*) FILTER (WHERE o.technologie LIKE ('%SQL%')) AS sql_cnt,
	COUNT(*) FILTER (WHERE o.technologie LIKE ('%JavaScript%')) AS javascript_cnt,
	COUNT(*) FILTER (WHERE o.technologie LIKE ('%AWS%')) AS aws_cnt
FROM job_db.oferty o
JOIN job_db.platforma p ON o.platforma_id = p.id
JOIN job_db.seniority s ON s.id = o.seniority_id 
GROUP BY p.nazwa

B:


WITH platforms_techs_cnts AS (
SELECT 
	p.nazwa AS platform,
	COUNT(*) FILTER (WHERE o.technologie LIKE ('%Python%')) AS python_cnt,
	COUNT(*) FILTER (WHERE o.technologie LIKE ('%Java%')) AS java_cnt,
	COUNT(*) FILTER (WHERE o.technologie LIKE ('%SQL%')) AS sql_cnt,
	COUNT(*) FILTER (WHERE o.technologie LIKE ('%JavaScript%')) AS javascript_cnt,
	COUNT(*) FILTER (WHERE o.technologie LIKE ('%AWS%')) AS aws_cnt
FROM job_db.oferty o
JOIN job_db.platforma p ON o.platforma_id = p.id
JOIN job_db.seniority s ON s.id = o.seniority_id 
GROUP BY p.nazwa
),
techs_long AS (
SELECT
	platform, 
	'Python' AS tech,
	python_cnt AS cnt
FROM platforms_techs_cnts
UNION ALL 
SELECT
	platform, 
	'Java' AS tech,
	java_cnt AS cnt
FROM platforms_techs_cnts
UNION ALL 
SELECT
	platform, 
	'SQL' AS tech,
	sql_cnt AS cnt
FROM platforms_techs_cnts
UNION ALL 
SELECT
	platform, 
	'JavaScript' AS tech,
	javascript_cnt AS cnt
FROM platforms_techs_cnts
UNION ALL 
SELECT
	platform, 
	'AWS' AS tech,
	aws_cnt AS cnt
FROM platforms_techs_cnts
),
platform_ranks AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY platform ORDER BY cnt desc) AS platform_rank
FROM techs_long
)
SELECT 
	platform,
	tech AS dominant_tech,
	cnt AS count
FROM platform_ranks
WHERE platform_rank = 1
ORDER BY platform

---

## Task 3 — Która Platforma Dominuje w Każdym Mieście?
**Difficulty: 4/5**

**Business question:**  
For each city with at least 5 total job offers, rank platforms by number of offers in that city. Show the top-ranked platform per city (rank = 1). If two platforms tie for first, show both.

Only include cities where the top platform has at least 3 offers. Exclude NULL cities.

**Expected output columns:**  
`miasto, platform, offer_count, rank`

Order by `offer_count DESC`, `miasto`.

**Difficulty: 4/5**

WITH cities_offers AS (
SELECT 
	miasto,
	platforma_id,
	COUNT(*) AS offer_count
FROM job_db.oferty o
WHERE miasto IS NOT NULL
GROUP BY miasto, o.platforma_id
),
cities_ranks AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY miasto ORDER BY offer_count DESC) AS rank
FROM cities_offers
)
SELECT 
	p.nazwa,
	cr.miasto,
	cr.offer_count,
	cr.rank
FROM cities_ranks cr
JOIN job_db.platforma p ON cr.platforma_id = p.id
WHERE RANK = 1 AND offer_count >= 3
ORDER BY cr.offer_count DESC, miasto



---

## Submission Instructions

Paste your queries below each task.