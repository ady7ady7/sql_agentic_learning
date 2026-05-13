# Daily SQL Practice Tasks

**Generated:** 2026-05-13
**Week 22, Day 3 Focus:** STDDEV GROUP BY drill + PERCENT_RANK on job_db + cross-schema JOIN

---

## Task 1: STDDEV — Transaction Volatility per User (GROUP BY)

**Scenario:**
The risk team wants one row per user showing their transaction amount volatility.

For each user show:
- `user_id`
- `tx_count` — number of transactions
- `avg_amount` — average transaction amount, rounded to 2 decimals
- `stddev_amount` — standard deviation of transaction amounts, rounded to 2 decimals
- `volatility` — `'high'` if stddev > 300, `'medium'` if > 150, `'low'` otherwise

**Tables:** `crappy_data_db.transactions`

**Requirements:**
- Use `STDDEV(amount)` as a plain GROUP BY aggregation — no OVER, no window function
- Exclude NULL amounts
- Only include users with at least 2 transactions
- Order by `stddev_amount DESC`

**Difficulty Rating:** 3/5

WITH users_tx_metrics AS (
SELECT 
	user_id,
	COUNT(*) AS tx_count,
	ROUND(AVG(amount)::NUMERIC, 2) AS avg_amount,
	ROUND(STDDEV(amount)::NUMERIC, 2) AS stddev_amount
FROM crappy_data_db.transactions t
WHERE amount IS NOT NULL
GROUP BY user_id
)
SELECT 
	*,
	CASE WHEN stddev_amount > 300 THEN 'high' WHEN stddev_amount > 150 THEN 'medium' ELSE 'low' END AS volatility
FROM users_tx_metrics
WHERE tx_count >= 2
ORDER BY stddev_amount DESC


---

## Task 2: PERCENT_RANK — Platform Ranking by Offer Count

**Scenario:**
The team wants to see how each platform ranks by total offer volume as a percentile.

For each platform show:
- `platform` — nazwa from platforma
- `offer_count`
- `pct_rank` — PERCENT_RANK() of this platform by offer_count, rounded to 2 decimals

**Tables:** `job_db.oferty`, `job_db.platforma`

**Requirements:**
- Exclude NULL platforma_id
- Order by `offer_count DESC`

**Difficulty Rating:** 4/5

WITH platform_offer_counts AS (
SELECT 
	p.nazwa AS platform,
	COUNT(*) AS offer_count
FROM job_db.oferty o
JOIN job_db.platforma p ON o.platforma_id = p.id
GROUP BY p.nazwa
)
SELECT 
	*,
	PERCENT_RANK() OVER (ORDER BY offer_count) AS PCT_RANK
FROM platform_offer_counts
WHERE platform IS NOT null
ORDER BY offer_count DESC

No need to round pct_rank, it's already rounded

---

## Task 3: Cross-Schema JOIN — Users in Cities With Job Listings

**Scenario:**
The team wants to find users from `crappy_data_db` who live in cities that have at least one job listing in `job_db`. This gives a sense of overlap between the user base and job market activity.

For each matching city show:
- `city`
- `user_count` — number of users from `crappy_data_db.users` in that city
- `job_listing_count` — number of offers in `job_db.oferty` for that city

Exclude NULL cities from both sides. Only include cities that appear in both datasets.
Order by `job_listing_count DESC`.

**Tables:** `crappy_data_db.users`, `job_db.oferty`

**Difficulty Rating:** 5/5


SELECT 
	u.city,
	COUNT(DISTINCT(u.id)) AS user_count,
	COUNT(DISTINCT(o.data_wystawienia)) AS job_listing_count
FROM crappy_data_db.users u
JOIN job_db.oferty o ON u.city = o.miasto
WHERE O.miasto IS NOT NULL AND u.city IS NOT NULL
GROUP BY u.city
ORDER BY job_listing_count DESC



---

## Submission Instructions

1. Task 1 — STDDEV(amount) GROUP BY aggregation, no window (3/5)
2. Task 2 — PERCENT_RANK platforms by offer count (4/5)
3. Task 3 — Cross-schema city JOIN: users vs job listings (5/5)
