# SQL Tasks — 2026-09-25 (Week 39, Day 5)

**Dataset:** job_db  
**Focus:** Pivot (conditional aggregation) · Recursive CTE (Type A, fixed 3-level hierarchy)

---

## Task 1 — Offer Count by Platform × Seniority (Pivot)

**Difficulty: 3/5**

**Business question:**  
For each platform, show the count of offers broken down by seniority level — one column per seniority. Use conditional aggregation.

Only include rows where `platforma_id` IS NOT NULL and `seniority_id` IS NOT NULL.

**Expected output columns:**  
`platform_name` plus one count column per seniority level (Senior, Expert, Mid, Lead/Principal, Manager/C-level, Junior, Staż)

Order by `platform_name`.

SELECT 
	p.nazwa AS platform_name,
	COUNT(*) FILTER (WHERE s.nazwa = 'Senior') AS senior_cnt,
	COUNT(*) FILTER (WHERE s.nazwa = 'Expert') AS expert_cnt,
	COUNT(*) FILTER (WHERE s.nazwa = 'Mid') AS mid_cnt,
	COUNT(*) FILTER (WHERE s.nazwa in ('Lead', 'Principal')) AS lead_principal_cnt,
	COUNT(*) FILTER (WHERE s.nazwa in ('Manager', 'C-level')) AS mg_c_cnt,
	COUNT(*) FILTER (WHERE s.nazwa = 'Junior') AS junior_cnt,
	COUNT(*) FILTER (WHERE s.nazwa = 'Staż') AS intern_cnt
FROM job_db.oferty o
JOIN job_db.platforma p ON o.platforma_id = p.id
JOIN job_db.seniority s ON o.seniority_id = s.id
GROUP BY p.nazwa

---

## Task 2 — Three-Level Rollup: Platform → Seniority → Offer Stats (Recursive CTE, Type A)

**Difficulty: 4/5**

**Business question:**  
Build a 3-level hierarchical rollup using a recursive CTE:
- Level 1: overall totals (all platforms, all seniority levels combined)
- Level 2: totals per platform
- Level 3: totals per platform × seniority combination

Each level should show: the grouping level (1, 2, or 3), platform name (NULL at level 1), seniority name (NULL at levels 1 and 2), offer count, and average max salary (using whatever salary-parsing approach you used earlier this week, or skip salary if you'd rather keep this focused on the CTE structure).

Only include rows where `platforma_id` IS NOT NULL and `seniority_id` IS NOT NULL.

**Expected output columns:**  
`level, platform_name, seniority_name, offer_count`

Order by `level`, `platform_name`, `seniority_name`.

---

SELECT 
	1 AS LEVEL,
	'All platforms' AS platform_name,
	'All seniorities' AS seniority_name,
	COUNT(*) AS offer_count
FROM job_db.oferty o
JOIN job_db.platforma p ON o.platforma_id = p.id
JOIN job_db.seniority s ON o.seniority_id = s.id
WHERE o.platforma_id IS NOT NULL AND o.seniority_id IS NOT NULL
UNION ALL
SELECT 
	2 AS LEVEL,
	p.nazwa AS platform_name,
	'All seniorities' AS seniority_name,
	COUNT(*) AS offer_count
FROM job_db.oferty o
JOIN job_db.platforma p ON o.platforma_id = p.id
JOIN job_db.seniority s ON o.seniority_id = s.id
WHERE o.platforma_id IS NOT NULL AND o.seniority_id IS NOT NULL
GROUP BY p.nazwa
UNION ALL
SELECT 
	3 AS LEVEL,
	p.nazwa AS platform_name,
	s.nazwa AS seniority_name,
	COUNT(*) AS offer_count
FROM job_db.oferty o
JOIN job_db.platforma p ON o.platforma_id = p.id
JOIN job_db.seniority s ON o.seniority_id = s.id
GROUP BY p.nazwa, s.nazwa




## Submission Instructions

Paste your queries below each task.
