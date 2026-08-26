# SQL Tasks — 2026-08-26 (Week 35, Day 2)

**Dataset:** transactions / orders / users / job_db  
**Focus:** YoY revenue · Self-join affinity · job_db trend

---

## Task 1 — Year-over-Year Monthly Revenue Growth

**Difficulty: 4/5**

**Business question:**  
For each month in the dataset, calculate the total order revenue and compare it to the same month in the previous year. Show the month, current revenue, previous year revenue, and YoY growth percentage.

Exclude months where the previous year's revenue is NULL (no data to compare).

Formula: `ROUND(((current - prev) / prev) * 100, 2)`

**Expected output columns:**  
`month, total_revenue, prev_year_revenue, yoy_growth_pct`

Order by `month`.


WITH orders_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
FROM crappy_data_db.orders o
),
monthly_revenues AS (
SELECT 
	month_,
	SUM(amount) AS total_revenue
FROM orders_months
GROUP BY month_
)
SELECT 
	*,
	LAG(total_revenue, 12) OVER (ORDER BY month_) AS prev_year_revenue,
	ROUND((total_revenue - LAG(total_revenue, 12) OVER (ORDER BY month_))::numeric / total_revenue::NUMERIC * 100, 2) AS yoy_growth_pct
FROM monthly_revenues


---

## Task 2 — Transaction Type Affinity (Self-Join)

**Difficulty: 4/5**

**Business question:**  
Find which pairs of transaction types most frequently appear together for the same user (i.e., the user has at least one transaction of each type).

Group by type pair, count distinct users who have both types, return the top 10 pairs by user count.

Avoid duplicate pairs — ensure type_a < type_b alphabetically.

**Expected output columns:**  
`type_a, type_b, user_count`

Order by `user_count DESC`, then `type_a`.


SELECT 
	t1.TYPE AS type_a,
	t2.TYPE AS type_b,
	COUNT(DISTINCT(t1.user_id)) AS user_count
FROM crappy_data_db.transactions t1
JOIN crappy_data_db.transactions t2 ON t1.user_id = t2.user_id
WHERE t1.TYPE > t2.TYPE AND t1.id != t2.id
GROUP BY t1.TYPE, t2.type
ORDER BY user_count DESC, type_a


type_a	type_b	user_count
transfer	deposit	80
payment	deposit	77
transfer	payment	75
withdrawal	deposit	74
purchase	deposit	73
transfer	purchase	73
withdrawal	transfer	73
withdrawal	payment	70
purchase	payment	68
withdrawal	purchase	64

As an interesting observations

---

## Task 3 — Job Offer Volume Trend by Platform (job_db)

**Difficulty: 5/5**

**Business question:**  
For each platform, show the monthly count of job offers (`data_wystawienia`). Then, using a window function, calculate a 3-month rolling average of offer volume per platform.

Only include rows where `data_wystawienia` IS NOT NULL and `platforma_id` IS NOT NULL.

The rolling average should look back: current month + 2 preceding months (ROWS frame).

**Expected output columns:**  
`platform_name, month, offer_count, rolling_3m_avg`

`rolling_3m_avg` rounded to 1 decimal place.

Order by `platform_name`, `month`.


WITH offers_platforms_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', o.data_wystawienia) AS month_
FROM job_db.oferty o
JOIN job_db.platforma p ON p.id = o.platforma_id
WHERE o.data_wystawienia IS NOT NULL AND o.platforma_id IS NOT NULL
),
monthly_offers AS (
SELECT 
	nazwa AS platform_name,
	month_,
	COUNT(*) AS offer_count
FROM offers_platforms_months
GROUP BY nazwa, month_
ORDER BY month_
)
SELECT 
	*,
	ROUND(AVG(offer_count) OVER (PARTITION BY platform_name ORDER BY month_ ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 1) AS rolling_3m_avg
FROM monthly_offers
ORDER BY platform_name, month_


---

## Submission Instructions

Paste your queries below each task.
