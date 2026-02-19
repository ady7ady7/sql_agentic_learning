# Daily SQL Practice Tasks

**Generated:** 2026-02-19
**Week 10, Day 4 Focus:** Advanced Window Functions + Gaps-and-Islands Variant + Hierarchy Consolidation

---

## Task 1: 3-Level Hierarchy — Users by Country and City

**Scenario:**
Build a 3-level hierarchy over user location data:
- Level 1: `'All Users'`
- Level 2: Distinct countries from the `users` table (exclude NULLs)
- Level 3: Distinct cities within each country (exclude NULLs), show city name

**Expected Output Columns:**
- `level` (integer)
- `name` (text) — country at Level 2, city at Level 3
- `parent_name` (text)
- `path` (text) — full path from root

**Requirements:**
- Pre-aggregate distinct countries and distinct country+city pairs before the recursive CTE
- Exclude NULL countries and NULL cities
- Termination condition required
- No hardcoded values

**Difficulty Rating:** 3/5

WITH RECURSIVE distinct_countries AS (
SELECT 
	DISTINCT country
FROM users u 
WHERE country IS NOT NULL
),
distinct_cities AS (
SELECT DISTINCT 
COUNTRY, CITY FROM users
WHERE city IS NOT NULL
),
HIERARCHY AS (
SELECT
	1 AS LEVEL,
	'All Users' AS name,
	NULL::TEXT AS parent_name,
	'All Users' AS PATH
UNION ALL
SELECT 
	h.LEVEL + 1,
	COALESCE(dc.country::TEXT, dcit.city::TEXT),
	h.name,
	h.PATH || ' > ' || COALESCE(dc.country::TEXT, dcit.city::TEXT)
FROM HIERARCHY h
LEFT JOIN distinct_countries dc ON h.LEVEL = 1
LEFT JOIN distinct_cities dcit ON h.LEVEL = 2 AND dcit.country = h.name
WHERE H.LEVEL < 3
)
SELECT * FROM HIERARCHY

---

## Task 2: Gaps-and-Islands — Transaction Dry Spells

**Scenario:**
The finance team wants to identify periods of inactivity — gaps between transactions for each user. Specifically, find users who had a gap of **at least 30 days** between two consecutive transactions.

For each such gap, show:

**Expected Output Columns:**
- `user_id` (integer)
- `last_transaction_date` (date) — date of the transaction before the gap
- `next_transaction_date` (date) — date of the transaction after the gap
- `gap_days` (integer) — number of days between the two transactions
- `longest_gap` (boolean) — true if this is the longest gap for that user, false otherwise

**Requirements:**
- Use `transactions` table
- Use `DATE(created_at)` to work at day granularity
- Only include gaps >= 30 days
- Order by `gap_days DESC`, `user_id ASC`

**Note:** This is a gaps problem, not islands — you're finding the spaces *between* data points, not grouping consecutive ones. LAG is the right tool here, not the RN-subtraction pattern.

**Difficulty Rating:** 4/5

WITH users_transactions AS (
SELECT 
 id AS transaction_id,
 user_id,
 DATE_TRUNC('Day', created_at) AS current_transaction_date,
 LAG(DATE_TRUNC('Day', created_at)) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_transaction_date
FROM transactions
),
users_gaps AS (
SELECT 
	*,
	EXTRACT('Day' FROM current_transaction_date - prev_transaction_date) AS gap_days,
	MAX(EXTRACT('Day' FROM current_transaction_date - prev_transaction_date)) OVER (PARTITION BY user_id) AS longest_gap
FROM users_transactions ns
WHERE prev_transaction_date IS NOT NULL
)
SELECT 
	user_id,
	transaction_id,
	current_transaction_date,
	prev_transaction_date,
	gap_days
FROM users_gaps
WHERE longest_gap = gap_days
AND gap_days >= 1
ORDER BY gap_days DESC, user_id ASC


Look, after this step It's clear to me THAT THERE ARE NO USERS WITH GAPS ABOVE 1 day - 1 day gap IS LITERALLY THE MAXIMUM in this dataset, so the best thing I could do is filter out users with 0 day gaps (there were a lot of them as well).

IMO I handled it well and adapted to available data.


---

## Task 3: Percentile Bands + Cumulative Share

**Scenario:**
The analytics team wants a transaction amount distribution report. Bucket transactions into percentile bands and show what share of total volume each band represents.

Classify each transaction into one of 4 quartile bands using `NTILE(4)`:
- Band 1: Bottom 25%
- Band 2: 25–50%
- Band 3: 50–75%
- Band 4: Top 25%

Then aggregate by band and show:

**Expected Output Columns:**
- `quartile_band` (integer) — 1 to 4
- `transaction_count` (bigint)
- `band_revenue` (numeric) — total amount in this band, rounded to 2 decimals
- `pct_of_total_revenue` (numeric) — this band's revenue as % of all revenue, rounded to 1 decimal
- `cumulative_revenue_pct` (numeric) — running cumulative % from band 1 to 4, rounded to 1 decimal

**Requirements:**
- Use `transactions` table, exclude NULL amounts
- Compute NTILE in a CTE, then aggregate
- Cumulative % must use a window SUM over the aggregated results
- Order by `quartile_band ASC`

**Difficulty Rating:** 4/5


WITH transactions_amount_quartiles AS (
SELECT 
	*,
	ntile(4) OVER (ORDER BY amount DESC) AS quartile_band
FROM transactions
),
quartile_bands_revenues AS (
SELECT
	quartile_band,
	COUNT(*) AS transaction_count,
	SUM(amount) AS band_revenue,
	(SELECT sum(amount) FROM transactions) AS total_revenue
FROM transactions_amount_quartiles
GROUP BY quartile_band
ORDER BY band_revenue DESC
)
SELECT 
	*,
	ROUND((band_revenue / total_revenue) * 100, 1) || ' %' AS pct_of_total_revenue,
	ROUND(SUM((band_revenue / total_revenue) * 100) OVER (ORDER BY quartile_band), 1) || '%' AS cumulative_revenue_pct
FROM quartile_bands_revenues


Done, your requirements satisfied with 100% effect :)).

---

## Submission Instructions

1. Task 1 — Users by country/city hierarchy (3/5)
2. Task 2 — Transaction dry spells / gaps (4/5)
3. Task 3 — Percentile bands + cumulative share (4/5)
