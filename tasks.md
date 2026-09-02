# SQL Tasks — 2026-09-03 (Week 36, Day 3)

**Dataset:** job_db  
**Focus:** Seniority share trend · Technology parsing · Monthly offer growth rate

---

## Task 1 — Monthly Seniority Share Trend

**Difficulty: 4/5**

**Business question:**  
For each month and seniority level, calculate:
- `offer_count` — number of offers that month for that seniority
- `monthly_total` — total offers across all seniority levels that month
- `share_pct` — offer_count / monthly_total * 100, rounded to 2 decimal places
- `prev_share_pct` — share_pct from the previous month (LAG)
- `share_change` — difference: share_pct - prev_share_pct, rounded to 2 decimal places

Only include rows where `data_wystawienia` IS NOT NULL and `seniority_id` IS NOT NULL.

**Expected output columns:**  
`month, seniority_name, offer_count, monthly_total, share_pct, prev_share_pct, share_change`

Order by `month`, `seniority_name`.



WITH offers_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', data_wystawienia) AS month_
FROM job_db.oferty o
),
monthly_offers_seniorities AS (
SELECT
	seniority_id,
	month_,
	COUNT(*) AS offer_count
FROM offers_months
WHERE data_wystawienia IS NOT NULL AND seniority_id IS NOT NULL
GROUP BY seniority_id, month_
),
seniorities_percentages AS (
SELECT 
	*,
	SUM(offer_count) OVER (PARTITION BY month_) AS monthly_total,
	ROUND(offer_count / SUM(offer_count) OVER (PARTITION BY month_) ::NUMERIC * 100, 2) AS share_pct
FROM monthly_offers_seniorities
),
prev_percentages AS (
SELECT 
	*,
	LAG(share_pct) OVER (PARTITION BY seniority_id ORDER BY month_) AS prev_share_pct
FROM seniorities_percentages
)
SELECT 
	s.nazwa AS seniority_name,
	p.month_,
	p.offer_count,
	p.monthly_total,
	p.share_pct,
	p.prev_share_pct,
	p.share_pct - p.prev_share_pct AS share_change
FROM prev_percentages p
JOIN job_db.seniority s ON p.seniority_id = s.id
ORDER BY month_, seniority_name

---

## Task 2 — Top 5 Technologies per Platform

**Difficulty: 5/5**

**Business question:**  
The `technologie` column contains comma-separated (or space-separated) tech stacks, e.g. `"Python, SQL, AWS"` or `"Confluence Jira"`. Use `regexp_split_to_table(technologie, '[,\s]+')` to explode each offer into individual technologies.

For each platform, find the top 5 most frequently mentioned technologies. Exclude empty strings and NULL values after splitting.

Use `RANK()` to rank technologies within each platform by count — include ties at position 5.

**Expected output columns:**  
`platform_name, technology, mention_count, rank`

Only include rows where `platforma_id` IS NOT NULL and `technologie` IS NOT NULL.

Order by `platform_name`, `rank`, `technology`.

WITH offers_split_techs AS (
SELECT 
	*,
	regexp_split_to_table(technologie, '[,\s]+') AS present_tech
FROM job_db.oferty o
WHERE o.platforma_id IS NOT NULL AND o.technologie IS NOT NULL
),
techs_count AS (
SELECT 
	p.nazwa AS platform_name,
	present_tech,
	COUNT(*) AS mention_count
FROM offers_split_techs o
JOIN job_db.platforma p ON p.id = o.platforma_id
GROUP BY p.nazwa, present_tech
),
platforms_rank AS (
SELECT 
	*,
	RANK() OVER (PARTITION BY platform_name ORDER BY mention_count DESC) AS rank
FROM techs_count
)
SELECT * FROM platforms_rank
WHERE RANK <= 5
ORDER BY platform_name, RANK, present_tech

---

## Task 3 — Monthly Offer Growth Rate

**Difficulty: 4/5**

**Business question:**  
For each month, calculate the total number of job offers and the month-over-month growth rate compared to the previous month.

Formula: `ROUND((current - prev) / prev::numeric * 100, 2)`

Exclude the first month (no previous month to compare). Also exclude months where `data_wystawienia` IS NOT NULL only.

**Expected output columns:**  
`month, offer_count, prev_offer_count, growth_rate_pct`

Order by `month`.

WITH offers_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', o.data_wystawienia) AS month_
FROM job_db.oferty o
WHERE o.data_wystawienia IS NOT NULL
),
monthly_offer_cnts AS (
SELECT 
	month_,
	COUNT(*) AS offer_count
FROM offers_months
GROUP BY month_
)
SELECT 
	*,
	lag(offer_count) OVER (ORDER BY month_) AS prev_offer_count,
	ROUND(offer_count / lag(offer_count) OVER (ORDER BY month_)::NUMERIC * 100, 2) AS growth_rate_pct
FROM monthly_offer_cnts




---

## Submission Instructions

Paste your queries below each task.
