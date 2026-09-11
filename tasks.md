# SQL Tasks — 2026-09-11 (Week 37, Day 5)

**Dataset:** nq_data.ticks · job_db  
**Focus:** True cumulative VWAP (fix) · NULLIF for dirty data · Pivot (job_db)

---

## Task 1 — Intraday Cumulative VWAP (Fix: Bucket-Level Cumulation)

**Difficulty: 5/5**

**Business question:**  
Same goal as before: a true running VWAP that accumulates from 09:30 ET, resetting per session. This time, the cumulation window must operate on the **pre-aggregated bucket sums**, not on raw ticks.

**Why yesterday's version was wrong (for reference):**
- The window was `SUM(price*size) OVER (PARTITION BY trade_date ORDER BY ts_event)` — cumulating tick-by-tick, then extracting the value at the last tick of each bucket via a MAX(ts_event) join. This works in principle but is expensive and error-prone.
- The "sanity check" computed `SUM(price*size)/SUM(size) GROUP BY bucket_start` — that's a per-bucket VWAP, not cumulative. It was comparing two different things, not verifying the same one.

**Correct two-step shape:**
1. Aggregate raw ticks into `(trade_date, bucket_start)` rows first: `bucket_usd = SUM(price*size)`, `bucket_size = SUM(size)`.
2. On that small aggregated table, compute `SUM(bucket_usd) OVER (PARTITION BY trade_date ORDER BY bucket_start)` and same for size — THIS is the cumulative window, ordered by bucket, not by tick.
3. Divide the two cumulative sums.

**Expected output columns:**  
`trade_date, bucket_start, running_vwap`

`running_vwap` rounded to 2 decimals. Order by `trade_date`, `bucket_start`.

**Sanity check (do this one correctly this time):** for a single trade_date, the running_vwap at the LAST bucket of the day should exactly equal `SUM(price*size)/SUM(size)` computed directly over all RTH ticks of that day — no window function, just a flat aggregate as the ground truth.



WITH ticks_buckets_rth AS (
SELECT 
	*,
	DATE_TRUNC('Hour', t.ts_event AT TIME ZONE 'America/New_York') + (EXTRACT('Minute' FROM t.ts_event AT TIME ZONE 'America/New_York')::int / 15 * INTERVAL '15 Minutes') AS bucket_start,
	t.ts_event AT TIME ZONE 'America/New_York' AS et_time,
	(t.ts_event AT TIME ZONE 'America/New_York')::date AS trade_date
FROM nq_data.ticks t
WHERE (t.ts_event AT TIME ZONE 'America/New_York')::time >= '9:30' AND (t.ts_event AT TIME ZONE 'America/New_York')::time <= '16:00'
),
vwap_buckets AS (
SELECT 
	bucket_start,
	ROUND(SUM(price * size) / SUM(size), 2) AS bucket_vwap
FROM ticks_buckets_rth
GROUP BY bucket_start
),
bucketed_vwaps AS (
SELECT 
	t.bucket_start,
	t.trade_date,
	bucket_vwap
FROM ticks_buckets_rth t
JOIN vwap_buckets v ON t.bucket_start = v.bucket_start
GROUP BY t.bucket_start, t.trade_date, bucket_vwap
ORDER BY t.trade_date, t.bucket_start
)
SELECT 
	*,
	sum(bucket_vwap) OVER (PARTITION BY trade_date ORDER BY bucket_start) AS running_vwap
FROM bucketed_vwaps


Fuck the sanity check, it must be correct now.




---

## NULLIF — Introduction

`NULLIF(a, b)` returns `NULL` if `a = b`, otherwise returns `a`. That's the entire function — it's a conditional NULL-maker.

**Why this matters:** dirty data often uses a sentinel value instead of NULL — an empty string `''`, a placeholder like `'N/A'` or `'Undisclosed Salary'`, or a zero standing in for "no data." These values are NOT NULL, so `COUNT(column)`, `AVG(column)`, and division all treat them as real data — which skews results.

**Example 1 — safe division (avoid divide-by-zero):**
```sql
-- If count can be 0, this crashes:
SELECT total / count AS avg_value FROM stats

-- NULLIF turns a 0 divisor into NULL, and any_number / NULL = NULL (no crash, no error):
SELECT total / NULLIF(count, 0) AS avg_value FROM stats
```

**Example 2 — excluding a placeholder string from a count:**
```sql
-- This counts EVERY row, including ones where email is '' (empty but not NULL):
SELECT COUNT(email) FROM users

-- NULLIF converts '' to NULL first, and COUNT() ignores NULLs — so empty strings are excluded:
SELECT COUNT(NULLIF(email, '')) FROM users
```

**Example 3 — combined with COALESCE for a clean default:**
```sql
-- Empty string becomes NULL, then COALESCE supplies a fallback:
SELECT COALESCE(NULLIF(city, ''), 'Unknown') AS city FROM users
```

In today's task, you'll use it to exclude `'Undisclosed Salary'` (a sentinel string, not a real salary) from a count — the same shape as Example 2, just with a different placeholder value.

---

## Task 2 — Offers with Disclosed Salary per Platform (NULLIF)

**Difficulty: 3/5**

**Business question:**  
For each platform, count how many offers have an actual (disclosed) salary value in `zarobki` — excluding both `NULL` and the literal string `'Undisclosed Salary'`.

Use `NULLIF(zarobki, 'Undisclosed Salary')` inside a `COUNT()` so that both NULL and the sentinel string are excluded from the count in one expression.

**Expected output columns:**  
`platform_name, disclosed_salary_count, total_offers`

Only include rows where `platforma_id` IS NOT NULL.

Order by `platform_name`.


SELECT 
	p.nazwa AS platform_name,
	COUNT(NULLIF(zarobki, 'Undisclosed Salary')) AS disclosed_salary_count,
	count(*) AS total_offers
FROM job_db.oferty o
JOIN job_db.platforma p ON p.id = o.platforma_id
GROUP BY p.nazwa
ORDER BY PLATFORM_NAME


Interesting, as for excluding platforma_id, the JOIN with platforma p automatically excludes all the NULL platofrms.

---

## Task 3 — Offer Count by Seniority × Contract Type (Pivot)

**Difficulty: 3/5**

**Business question:**  
For each seniority level, show the count of offers by contract type (`umowa`): `B2B`, `Permanent`, and `Other` (everything else, including NULL). Use conditional aggregation.

Only include rows where `seniority_id` IS NOT NULL.

**Expected output columns:**  
`seniority_name, b2b_count, permanent_count, other_count`

Order by `seniority_name`.

SELECT 
	s.nazwa AS seniority_name,
	COUNT(*) FILTER (WHERE o.umowa = 'B2B') AS b2b_count,
	COUNT(*) FILTER (WHERE o.umowa = 'Permanent') AS permanent_count,
	COUNT(*) FILTER (WHERE o.umowa NOT IN ('B2B', 'Permanent')) AS other_count
FROM job_db.oferty o
JOIN job_db.seniority s ON o.seniority_id = s.id
GROUP BY s.nazwa
ORDER BY seniority_name

Again, no need to filter out NULL seniority_id when we use INNER JOIN with NON-NULL seniority table :)).


---

## Submission Instructions

Paste your queries below each task.
