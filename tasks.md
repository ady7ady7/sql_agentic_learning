# Daily SQL Practice Tasks

**Generated:** 2026-05-04
**Week 21, Day 1 Focus:** Exploring job_db — offer distribution, platform/seniority breakdown, tech stack filtering

---

## Task 1: Offer Distribution — By Platform and Seniority

**Scenario:**
Get a high-level feel for the data. For each combination of platform and seniority level, count how many offers exist.

Show:
- `platform` — nazwa from platforma
- `seniority` — nazwa from seniority
- `offer_count`

Only include combinations that actually have offers. Order by `offer_count DESC`.

**Tables:** `job_db.oferty`, `job_db.platforma`, `job_db.seniority`

**Difficulty Rating:** 3/5

SELECT 
	o.platforma_id,
	p.nazwa,
	o.seniority_id,
	s.nazwa,
	COUNT(*) AS offer_count
FROM job_db.oferty o
JOIN job_db.platforma p ON p.id = o.platforma_id 
JOIN job_db.seniority s ON s.id = o.seniority_id
GROUP by o.platforma_id, p.nazwa, o.seniority_id, s.nazwa
ORDER BY offer_count DESC


FYI, There are around 1300-1500 rows in the whole db.
It doesn't seem to be a very huge schema, so I reckon we might also want to expand it by generating some more data and/or simulating more realistic conditions, and/or add relevant views/tables as we explore it.

We might also thing about different db/schemas if we want to work with a higher scale db.


---

## Task 2: City Dominance — Top 5 Cities per Seniority Level

**Scenario:**
The team wants to know which cities dominate job postings for each seniority level — specifically the top 5 cities by offer count per seniority.

Show:
- `seniority` — nazwa from seniority
- `miasto`
- `offer_count`
- `city_rank` — rank within seniority by offer count (use RANK())

Only return rows where `city_rank <= 5`. Exclude NULL cities and NULL seniority_id.

**Tables:** `job_db.oferty`, `job_db.seniority`

**Difficulty Rating:** 4/5


WITH cities_seniority_counts AS (
SELECT 
	s.nazwa AS seniority,
	o.miasto,
	COUNT(*) AS offer_count
FROM job_db.oferty o
JOIN job_db.seniority s ON s.id = o.seniority_id
GROUP BY s.nazwa, o.miasto
),
cities_seniorities_ranks AS (
SELECT 
	*,
	ROW_NUMBER() OVER (PARTITION BY seniority ORDER BY offer_count DESC, miasto) AS city_rank
FROM cities_seniority_counts
WHERE miasto IS NOT NULL AND seniority IS NOT NULL
)
SELECT * FROM cities_seniorities_ranks
WHERE city_rank <= 5



---

## Task 3: Tech Stack Exploration — Most Common Technologies

**Scenario:**
The `technologie` field is a free-text blob, but you can still get signal from it. Find the top 10 most frequently mentioned individual keywords from a predefined list: `'Python'`, `'Java'`, `'SQL'`, `'AWS'`, `'Azure'`, `'JavaScript'`, `'Docker'`, `'Kubernetes'`, `'Git'`, `'Linux'`.

For each keyword show:
- `technology`
- `mention_count` — number of offers where `technologie ILIKE '%keyword%'`

Use a `VALUES` clause to define the keyword list, then join/filter against `oferty`. Order by `mention_count DESC`.

**Tables:** `job_db.oferty`

**Difficulty Rating:** 5/5

WITH keywords AS (
SELECT keyword
FROM (
    VALUES ('Python'), ('Java'), ('SQL'), ('AWS'), ('Azure'),
           ('JavaScript'), ('Docker'), ('Kubernetes'), ('Git'), ('Linux')
) AS t(keyword)
)
SELECT 
	k.keyword,
	COUNT(*) FILTER (WHERE o.technologie LIKE '%' || k.keyword || '%') AS tech_count
FROM job_db.oferty o
CROSS JOIN keywords k
GROUP BY k.keyword
ORDER BY tech_count DESC


It was a very difficult task for me and I needed your help with that. The pattern IS DEFINITELY NOT LOCKED IN and we need to practice that

In Polish:

Ciekawy pattern – najpierw tworzymy CTE z valuesami – to w ogóle nie jest obowiązkowe btw., ale tak zrobiłem, a potem samo filtrowanie dodaje element ‘%’ || variable || ‘%’, bo faktycznie chcemy filtrować po danej zmiennej.

No i tutaj też cos bardzo nieoczywistego, czyli cross-join, w tym wypadku ma sens, bo sprawdzamy każdą ofertę pod kontem każdego słowa. Ale cross joinó z reguły nie robię.


---

## Submission Instructions

1. Task 1 — Platform × seniority offer count (3/5)
2. Task 2 — Top 5 cities per seniority with RANK() (4/5)
3. Task 3 — Tech keyword frequency via VALUES + ILIKE (5/5)
