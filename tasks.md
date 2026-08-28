# SQL Tasks — 2026-08-28 (Week 35, Day 4)

**Dataset:** transactions / users / job_db  
**Focus:** PERCENT_RANK + NTILE · job_db pivot (conditional aggregation)

---

## Task 1 — User Spending Distribution (PERCENT_RANK + NTILE)

**Difficulty: 4/5**

**Business question:**  
For each user, calculate their total transaction amount. Then show where they rank in the overall spending distribution using both `PERCENT_RANK` and `NTILE(5)` (quintiles).

Only include users who have at least 2 transactions.

**Expected output columns:**  
`user_id, total_amount, percent_rank, quintile`

- `percent_rank` rounded to 4 decimal places
- `quintile` is 1 (lowest) to 5 (highest)

Order by `total_amount DESC`.


WITH users_amounts AS (
SELECT 
	user_id,
	SUM(t.amount) AS total_amount
FROM crappy_data_db.transactions t
GROUP BY user_id
)
SELECT 
	*,
	PERCENT_RANK() OVER (ORDER BY TOTAL_AMOUNT) AS percent_rank,
	ntile(5) OVER (ORDER BY total_amount) AS quantile
FROM users_amounts


---

## Task 2 — Platform × Contract Type Pivot (job_db)

**Difficulty: 4/5**

**Business question:**  
For each job platform, show how many offers have each contract type: `B2B`, `Permanent`, and everything else (including NULL) as `Other`.

Use conditional aggregation (`FILTER` or `CASE WHEN`) — one row per platform, three columns for the counts.

Only include platforms where `platforma_id` IS NOT NULL. Join to `job_db.platforma` to get the platform name.

**Expected output columns:**  
`platform_name, b2b_count, permanent_count, other_count`

Order by `platform_name`.


SELECT 
	p.nazwa AS platform_name,
	COUNT(*) FILTER (WHERE o.umowa = 'B2B') AS b2b_count,
	COUNT(*) FILTER (WHERE o.umowa = 'Permanent') AS permanent_count,
	COUNT(*) FILTER (WHERE o.umowa != 'B2B' AND o.umowa != 'Permanent') AS other_count
FROM job_db.oferty o
JOIN job_db.platforma p ON o.platforma_id = p.id
WHERE o.platforma_id IS NOT NULL
GROUP BY p.nazwa
ORDER BY platform_name


---

## Submission Instructions

Paste your queries below each task.
