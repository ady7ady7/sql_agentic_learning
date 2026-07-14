# NQ Project — Week 30 Day 2

**Generated:** 2026-07-14
**Focus:** Signed delta surge asymmetry (RTH-VOL-014b) + EU high as resistance on bearish RTH days (RTH-GLOB-006)

---

## Task 1: Signed Delta Surge → Asymmetry Between Up and Down Surges

**Scenario:**
RTH-VOL-014 used ABS(bucket_delta) for the surge ratio — this treated a +3,000 up surge the same as a -3,000 down surge. You noted this loses sign information. The hypothesis: down surges (aggressive selling) may exhaust differently than up surges (aggressive buying). Market makers may absorb sell-side aggression more readily than buy-side, or vice versa. The signed ratio preserves direction and lets us split the exhaustion signal by actual surge type. **(ID: RTH-VOL-014b)**

**Architecture:**
1. CTE 1 — from `rth_15min_buckets_agg`, compute `local_avg_abs_delta` = AVG(ABS(bucket_delta)) OVER (PARTITION BY trade_date ORDER BY bucket_start ROWS BETWEEN 4 PRECEDING AND 1 PRECEDING) — same baseline as VOL-014
2. CTE 2 — compute `signed_ratio` = bucket_delta / NULLIF(local_avg_abs_delta, 0) (signed, no ABS on numerator). Classify:
   - `surge_type`: 'up_surge' (signed_ratio > 1.5), 'down_surge' (signed_ratio < -1.5), 'no_surge' (between -1.5 and 1.5)
   - Also carry LEAD(bucket_direction) as `next_bucket_direction`
3. Final SELECT — GROUP BY `surge_type`, compute:
   - `bucket_count`
   - `next_bucket_up_pct`
   - `next_bucket_down_pct`

**Finding to answer:** Is down surge exhaustion stronger than up surge exhaustion, or symmetric? Does a large aggressive sell bucket predict a bounce (next bucket up) more reliably than a large aggressive buy bucket predicts a continuation? Compare all three groups (up_surge / down_surge / no_surge) against the RTH-VOL-014 ABS baseline



WITH local_delta AS (
SELECT 
	*,
	bucket_start::date AS trade_date,
	AVG(ABS(bucket_delta)) OVER ( PARTITION BY bucket_start::date ORDER BY bucket_start ROWS BETWEEN 4 PRECEDING AND 1 PRECEDING) AS local_delta
FROM nq_data.rth_15min_buckets_agg
),
surge_agg AS (
SELECT
	*,
	bucket_delta / local_delta AS delta_ratio, 
	LEAD(bucket_direction) OVER (PARTITION BY trade_date) AS next_bucket_direction,
	CASE WHEN ROUND(ABS(bucket_delta) / ABS(local_delta), 2) > 1.5 THEN TRUE ELSE FALSE END AS is_surge
FROM local_delta
WHERE local_delta IS NOT NULL
),
pre_final_surge_agg AS (
SELECT
	*,
	CASE WHEN delta_ratio > 1.5 THEN 'up_surge' WHEN delta_ratio < -1.5 THEN 'down_surge' ELSE 'no_surge' END AS surge_type
FROM surge_agg
)
SELECT 
	surge_type,
	COUNT(*) AS bucket_count,
	ROUND(COUNT(*) FILTER (WHERE next_bucket_direction = 'up') / count(*)::NUMERIC * 100, 2)AS next_bucket_up_pct,
	ROUND(COUNT(*) FILTER (WHERE next_bucket_direction = 'down') / count(*)::NUMERIC * 100, 2)AS next_bucket_down_pct
FROM pre_final_surge_agg 
GROUP BY surge_type


surge_type	bucket_count	next_bucket_up_pct	next_bucket_down_pct
up_surge	472	44.92	46.19
down_surge	541	43.62	44.92
no_surge	2,904	50	47.73

**Tables:** `nq_data.rth_15min_buckets_agg`

**Difficulty Rating:** 3/5

**(ID: RTH-VOL-014b)**

---

## Task 2: EU High as Resistance on Bearish RTH Days

**Scenario:**
RTH-GLOB-001/002 established EU low undercut rates on bullish RTH days. The mirror question: on bearish RTH days, does the EU session high act as resistance during RTH? If price opens below EU high and stays below it through the session, it's meaningful resistance. If RTH regularly exceeds EU high even on bearish days, it's not a reliable short entry level. Weekday breakdown for Tuesday specifically is most actionable. **(ID: RTH-GLOB-006)**

**Architecture:**
1. CTE 1 — compute EU high per trade_date from raw ticks: filter `ts_event AT TIME ZONE 'America/New_York'` between '02:00' and '09:30', group by `(ts_event AT TIME ZONE 'America/New_York')::date` as trade_date, take MAX(price) as `eu_high`
2. CTE 2 — JOIN to `daily_ohlcv_rth` on trade_date. Filter bearish RTH days: `close < open`. Compute:
   - `exceeded_eu_high`: rth high > eu_high (price broke above EU high during RTH)
   - `rth_open_vs_eu_high`: rth open - eu_high (positive = RTH opens above EU high, negative = opens below)
3. Final SELECT — overall + GROUP BY weekday:
   - `days`
   - `pct_exceeded_eu_high` — how often does RTH price break above EU high on bearish days
   - `avg_rth_open_vs_eu_high` — where does RTH open relative to EU high on average
   - `avg_eu_high_to_rth_high` — on days that DO exceed EU high, how far above does it go

**Finding to answer:** On bearish RTH days, does EU high hold as resistance (RTH stays below it)? Or does price regularly spike above EU high before selling off? Is Tuesday specifically a day where EU high reliably caps the move?

**Tables:** `nq_data.nq_data` (raw ticks), `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 3/5

**(ID: RTH-GLOB-006)**


I've slightly changed th elogic, but it makes perfect snse:

WITH eu_first_ticks_agg AS (
SELECT 
	*,
	ts_event::DATE AS trade_date
FROM nq_data.ticks t 
WHERE (ts_event AT TIME ZONE 'America/New_York')::time >= '2:00' AND (ts_event AT TIME ZONE 'America/New_York')::time < '9:30'
),
eu_levels_aggregation AS (
SELECT 
	trade_date,
	MAX(price) AS eu_high,
	MIN(price) AS eu_low,
	MAX(price) - MIN(price) AS eu_range,
	ROUND(MAX(price) - ((MAX(price) - MIN(price)) / 2), 2) AS eu_midpoint
FROM eu_first_ticks_agg
GROUP BY trade_date
),
eu_us_joint_agg AS (
SELECT DISTINCT ON (e.trade_date, e.eu_high)
	e.trade_date,
	d.weekday,
	e.eu_high,
	e.eu_low,
	e.eu_midpoint,
	e.eu_range,
	d.OPEN AS rth_open,
	d."close" AS rth_close,
	d.low AS rth_low,
	d.high AS rth_high,
	LAG(d."close") OVER (ORDER BY d.trade_date) AS prev_close,
	LAG(d.open) OVER (ORDER BY d.trade_date) AS prev_open,
	ROUND((d.OPEN - e.eu_low) / e.eu_range::NUMERIC * 100, 2) AS rth_open_location,
	ROUND(d.OPEN - e.eu_high, 2) AS rth_open_vs_eu_high,
	CASE WHEN d.low < e.eu_midpoint THEN 1 ELSE 0 END AS reached_eu_midpoint,
	CASE WHEN d.low < e.eu_low THEN 1 ELSE 0 END AS reached_eu_low,
	CASE WHEN d.high > e.eu_high THEN 1 ELSE 0 END AS reached_eu_high
FROM eu_levels_aggregation e
JOIN nq_data.daily_ohlcv_rth d ON e.trade_date = d.trade_date
),
prev_day_direction_agg AS (
SELECT 
	*,
	CASE WHEN rth_close > rth_open THEN 'bullish' ELSE 'bearish' END AS day_direction,
	CASE WHEN prev_close > prev_open THEN 'bullish' ELSE 'bearish' END AS prev_day_direction
FROM eu_us_joint_agg
WHERE prev_close IS NOT NULL AND prev_open IS NOT NULL
)
SELECT
	prev_day_direction,
	COUNT(*) AS days,
	ROUND(AVG(eu_range), 2) AS avg_eu_range,
	ROUND(AVG(rth_open_vs_eu_high), 2) AS avg_rth_open_vs_eu_high,
	ROUND(SUM(reached_eu_midpoint)/ COUNT(*)::NUMERIC * 100, 2) AS avg_eu_range,
	ROUND(AVG(rth_open_location), 2) AS avg_rth_open_location,
	ROUND(SUM(reached_eu_low)/ COUNT(*)::NUMERIC * 100, 2) AS pct_undercut_eu_low,
	ROUND(SUM(reached_eu_high)/ COUNT(*)::NUMERIC * 100, 2) AS pct_went_above_eu_high
FROM prev_day_direction_agg
WHERE day_direction = 'bearish'
GROUP BY prev_day_direction


prev_day_direction	days	avg_eu_range	avg_rth_open_vs_eu_high	avg_eu_range	avg_rth_open_location	pct_undercut_eu_low	pct_went_above_eu_high
bearish	37	195.07	-87.11	97.3	56.27	89.19	43.24
bullish	36	185.7	-94.94	97.22	48.18	86.11	38.89
---

## Submission Instructions

Paste your queries and results. Log query IDs: RTH-VOL-014b, RTH-GLOB-006.
