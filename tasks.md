# NQ Project — Week 30 Day 1

**Generated:** 2026-07-13
**Focus:** Delta surge analysis — local relative delta as intraday momentum signal (RTH-VOL-014) + session character classification from opening delta pressure (RTH-VOL-015)

---

## Task 1: Local Delta Surge → Next Bucket Direction

**Scenario:**
RTH-VOL-007/008 established that positive prior bucket delta predicts next bucket direction (09:45 window: 66.2% up, +20 pts avg move). But that used a binary positive/negative split — it ignored magnitude. The hypothesis here: an unusually large delta relative to recent buckets (a "surge") carries a stronger directional signal than a mild positive/negative delta. A surge of +2,000 contracts net in one bucket when the last 4 averaged ±300 is a different signal than a routine +200. **(ID: RTH-VOL-014)**

**Architecture:**
1. CTE 1 — from `rth_15min_buckets_agg`, compute `local_avg_abs_delta` = AVG(ABS(bucket_delta)) over the 4 preceding buckets within the same trade_date, using `LAG` or a window with `ROWS BETWEEN 4 PRECEDING AND 1 PRECEDING`
2. CTE 2 — compute `local_delta_ratio` = ABS(bucket_delta) / NULLIF(local_avg_abs_delta, 0). Flag `is_surge` = local_delta_ratio > 1.5 (or experiment with 2.0). Also carry `bucket_direction` and use LAG to get `next_bucket_direction`
3. Final SELECT — GROUP BY `is_surge` + `bucket_direction`, compute:
   - `bucket_count`
   - `next_bucket_up_pct`
   - Compare surge vs non-surge follow-through rates

**Finding to answer:** Does a local delta surge (unusually large pressure relative to the last 4 buckets) produce stronger directional follow-through than a routine delta? Is the 66.2% from RTH-VOL-008 amplified when the delta is also a surge? Does surge + direction beat direction alone?

**Tables:** `nq_data.rth_15min_buckets_agg`

**Difficulty Rating:** 3/5

**(ID: RTH-VOL-014)**

WITH local_delta AS (
SELECT 
	*,
	bucket_start::date AS trade_date,
	AVG(ABS(bucket_delta)) OVER ( PARTITION BY bucket_start::date ORDER BY bucket_start ROWS BETWEEN 4 PRECEDING AND 1 PRECEDING) AS local_delta
FROM nq_data.rth_15min_buckets_agg
),
local_delta_surge_agg AS (
SELECT 
	*,
	ROUND(ABS(bucket_delta) / ABS(local_delta), 2) AS local_delta_ratio,
	LEAD(bucket_direction) OVER (PARTITION BY trade_date) AS next_bucket_direction,
	CASE WHEN ROUND(ABS(bucket_delta) / ABS(local_delta), 2) > 1.5 THEN TRUE ELSE FALSE END AS is_surge
FROM local_delta
)
SELECT 
	bucket_direction,
	is_surge,
	COUNT(*) AS bucket_count,
	ROUND(COUNT(*) FILTER (WHERE next_bucket_direction = 'up') / COUNT(*)::NUMERIC * 100, 2) AS next_bucket_up_pct,
	ROUND(COUNT(*) FILTER (WHERE next_bucket_direction = 'down') / COUNT(*)::NUMERIC * 100, 2) AS next_bucket_down_pct
FROM local_delta_surge_agg
GROUP BY bucket_direction, is_surge



bucket_direction	is_surge	bucket_count	next_bucket_up_pct	next_bucket_down_pct
down	false	1,475	50.58	47.25
up	false	1,578	50.06	47.78
down	true	523	44.36	44.93
flat	false	13	38.46	61.54
up	true	487	43.94	46.2

Perhaps it would make more sense to check only positive delta for up and negative for down instead of ABS? idk.


---

## Task 2: Opening Delta Pressure → Session Character Classification

**Scenario:**
RTH-VOL-014 looks at intraday surge signals. This task zooms out: can you classify the character of the entire session based on the delta pressure in the first 3-4 buckets (09:30–10:30)? Days where the opening buckets show persistently high absolute delta ("hot open") may behave differently — wider range, earlier HOD/LOD formation, stronger directional close — compared to days with muted opening delta ("cold open"). **(ID: RTH-VOL-015)**

**Architecture:**
1. CTE 1 — from `rth_15min_buckets_agg`, filter to first 4 buckets per trade_date (09:30, 09:45, 10:00, 10:15). Compute `avg_abs_opening_delta` = AVG(ABS(bucket_delta)) per trade_date
2. CTE 2 — NTILE(3) on `avg_abs_opening_delta` across all trade_dates → low/medium/high opening pressure buckets
3. CTE 3 — JOIN to `daily_ohlcv_rth` for full-day range, close direction, close location. JOIN to `rth_firsthour_rest_ohlc_ranges` for FH range
4. Final SELECT — GROUP BY opening_pressure bucket:
   - `days`
   - `avg_full_day_range`
   - `avg_fh_range`
   - `pct_bullish_close`
   - `avg_close_location`

**Finding to answer:** Do high opening-delta days produce wider full-day ranges? Do they have stronger directional closes (higher pct_bullish or more extreme close location)? Can you use the first 4 buckets of delta to classify the session type before 10:30?

**Tables:** `nq_data.rth_15min_buckets_agg`, `nq_data.daily_ohlcv_rth`, `nq_data.rth_firsthour_rest_ohlc_ranges`

**Difficulty Rating:** 3/5

**(ID: RTH-VOL-015)**

WITH buckets_windows_delta AS (
SELECT 
	*,
	TO_cHAR(bucket_start, 'HH24:MI')::time AS time_window,
	bucket_start::date AS trade_date,
	AVG(ABS(bucket_delta)) OVER ( PARTITION BY bucket_start::date ORDER BY bucket_start ROWS BETWEEN 3 PRECEDING AND CURRENT ROW) AS local_delta
FROM nq_data.rth_15min_buckets_agg
),
opening_delta_agg AS (
SELECT 
	trade_date,
	ROUND(AVG(local_delta), 2) AS avg_abs_opening_delta
FROM buckets_windows_delta
WHERE time_window <= '10:15'
GROUP BY trade_date
),
pre_final_agg AS (
SELECT
	o.trade_date,
	o.avg_abs_opening_delta,
	NTILE(3) OVER (ORDER BY o.avg_abs_opening_delta) AS opening_delta_bucket,
	r.fh_open,
	r.fh_close,
	r.r_open,
	r.r_close,
	ABS(r.fh_high - r.fh_low) AS fh_range,
	ABS(d.high - d.low) AS full_day_range,
	CASE WHEN r.r_close > r.r_open THEN 1 ELSE 0 END AS bullish_rest,
	ROUND((d.CLOSE - d.low) / (d.high - d.low)::NUMERIC * 100, 2) AS close_location
FROM opening_delta_agg o
JOIN nq_data.rth_firsthour_rest_ohlc_ranges r ON o.trade_date = r.trade_date
JOIN nq_data.daily_ohlcv_rth d ON o.trade_date = d.trade_date
)
SELECT
	opening_delta_bucket,
	COUNT(*) AS days,
	ROUND(AVG(full_day_range), 2) AS avg_full_day_range,
	ROUND(AVG(fh_range), 2) AS avg_fh_range,
	ROUND(SUM(bullish_rest) / COUNT(*)::NUMERIC * 100, 2) AS pct_bullish_rest,
	ROUND(AVG(close_location), 2) AS avg_close_location
FROM pre_final_agg
GROUP BY opening_delta_bucket


opening_delta_bucket	days	avg_full_day_range	avg_fh_range	pct_bullish_rest	avg_close_location
3	62	355.77	227.26	51.61	53.77
2	62	335.84	194.81	74.19	65.72
1	62	317.28	176.85	46.77	51.86

---

## Submission Instructions

Paste your queries and results. Log query IDs: RTH-VOL-014, RTH-VOL-015.
