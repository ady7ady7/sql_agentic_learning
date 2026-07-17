# NQ Project — Week 30 Day 5

**Generated:** 2026-07-17
**Focus:** Close location × gap direction × weekday (RTH-CLOSE-003) + First hour range × gap size on Friday (RTH-FH-007)

---

## Task 1: Close Location × Gap Direction × Weekday

**Scenario:**
RTH-CLOSE-002 established close location by weekday (Monday 62%, Thursday 52.5%) but without gap direction. RTH-GAP-003 showed large gap-downs produce bullish close 74% of the time — but where in the day's range does price actually close? A gap-down bullish day that closes at 80% of range is a very different trade than one that closes at 55%. This task adds gap direction to the close location picture and gives concrete exit targets per weekday × gap scenario. **(ID: RTH-CLOSE-003)**

**Architecture:**
1. CTE 1 — from `daily_ohlcv_rth`, compute per trade_date:
   - `gap_direction` = CASE WHEN open > LAG(close) OVER (ORDER BY trade_date) THEN 'gap_up' ELSE 'gap_down' END
   - `gap_size` = ABS(open - LAG(close) OVER (ORDER BY trade_date))
   - `close_location` = ROUND((close - low) / (high - low)::NUMERIC * 100, 2)
   - `day_direction` = CASE WHEN close > open THEN 'bullish' ELSE 'bearish' END
2. Final SELECT — GROUP BY `weekday` × `gap_direction` × `day_direction`:
   - `days`
   - `avg_close_location`
   - `pct_closed_upper_half` (close_location > 50)
   - `avg_gap_size`
   - `avg_full_day_range` = AVG(high - low)

**Finding to answer:** On gap-down bullish days, does price close high in range (>65%) — confirming a full reversal? Does this vary by weekday? On gap-down bearish days, does price close near the low (<35%)? Friday gap-down specifically: given 73.3% bullish (N=15), where does it close in range? These numbers become exit targets.

**Tables:** `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 3/5

**(ID: RTH-CLOSE-003)**

WITH daily_agg_gaps AS (
SELECT 
	*,
	high - low AS daily_range,
	CASE WHEN OPEN > LAG(CLOSE) OVER (ORDER BY trade_date) THEN 'gap_up' ELSE 'gap_down' END AS gap_direction,
	abs(OPEN - (LAG(CLOSE) OVER (ORDER BY trade_date))) AS gap_size,
	ROUND(abs((CLOSE - low) / (high - low))::NUMERIC * 100, 2) AS close_location,
	CASE WHEN CLOSE > OPEN THEN 'bullish' ELSE 'bearish' END AS day_direction
FROM nq_data.daily_ohlcv_rth
)
SELECT 
	weekday,
	gap_direction,
	day_direction,
	COUNT(*),
	ROUND(AVG(gap_size), 2) AS avg_gap_size,
	ROUND(AVG(close_location), 2) AS avg_close_location,
	ROUND(COUNT(*) FILTER (WHERE close_location > 50) / COUNT (*)::NUMERIC * 100, 2) AS pct_closed_upper_half,
	ROUND(AVG(daily_range), 2) AS avg_full_day_range
FROM daily_agg_gaps
WHERE gap_size IS NOT NULL
GROUP BY weekday, gap_direction, day_direction

weekday	gap_direction	day_direction	count	avg_gap_size	avg_close_location	pct_closed_upper_half	avg_full_day_range
Friday	gap_up	bearish	10	110.98	25.97	0	346.95
Wednesday	gap_down	bearish	5	66.8	43.89	40	423.8
Wednesday	gap_down	bullish	7	74.82	72.48	85.71	282.18
Thursday	gap_up	bearish	12	200.38	41.89	41.67	384.02
Tuesday	gap_up	bearish	10	76.2	26.36	10	308.43
Tuesday	gap_down	bearish	7	126.96	25.94	0	296.93
Tuesday	gap_up	bullish	8	110.25	77.92	87.5	300.41
Wednesday	gap_up	bullish	12	91.63	76.94	91.67	333.52
Monday	gap_up	bearish	11	212.75	33.94	18.18	265.16
Monday	gap_down	bullish	14	198.96	78.66	92.86	353.2
Friday	gap_down	bullish	11	193.39	75.78	100	428.77
Tuesday	gap_down	bullish	14	173.39	83.73	100	328.38
Wednesday	gap_up	bearish	10	194.63	34.89	20	329.05
Friday	gap_up	bullish	10	110.93	74.8	90	319.05
Thursday	gap_up	bullish	5	87.75	70.43	60	260.4
Thursday	gap_down	bullish	10	186.38	78.55	100	397.55
Friday	gap_down	bearish	4	201.63	19.4	0	398.25
Monday	gap_down	bearish	4	68	46.39	50	320
Monday	gap_up	bullish	9	235.42	73.43	88.89	261.17
Thursday	gap_down	bearish	12	99.35	33.9	25	375.92
---

## Task 2: First Hour Range × Gap Size on Friday

**Scenario:**
RTH-VOL-013 showed Friday has the biggest 09:30→10:30 range expansion jump of any weekday (+26.5pp, from 47.7% to 74.2%). RTH-FH-003 shows Friday avg FH range is 215 pts (second highest). The question: is Friday's wide FH driven by large gap days specifically? A big gap-down Friday may produce a wider FH (price explores both sides of the gap) while a flat-open Friday may be tighter. If confirmed, gap size becomes a pre-market predictor of how wide the first hour will be — useful for sizing. **(ID: RTH-FH-007)**

**Architecture:**
1. CTE 1 — from `daily_ohlcv_rth`, compute per trade_date:
   - `gap_size` = ABS(open - LAG(close) OVER (ORDER BY trade_date))
   - `gap_direction` = CASE WHEN open > LAG(close) OVER (ORDER BY trade_date) THEN 'gap_up' ELSE 'gap_down' END
   - `gap_bucket` = NTILE(3) OVER (ORDER BY ABS(open - LAG(close)...)) — small/medium/large gap
2. CTE 2 — JOIN to `rth_firsthour_rest_ohlc_ranges` on trade_date:
   - `fh_range` = fh_high - fh_low
   - `fh_pct_of_day` = fh_range / (high - low) from daily_ohlcv_rth
   - `rest_bullish` = r_close > r_open
3. Final SELECT — filter Friday only, GROUP BY `gap_bucket` × `gap_direction`:
   - `days`
   - `avg_gap_size`
   - `avg_fh_range`
   - `avg_full_day_range`
   - `fh_pct_of_day`
   - `rest_bullish_pct`

**Finding to answer:** Do large gap Fridays produce wider first hours? Is the relationship linear (small gap → tight FH, large gap → wide FH)? Does a large gap-down Friday produce a wider FH than a large gap-up Friday? And does FH direction on a large gap-down Friday predict the afternoon (rest_bullish_pct)?

**Tables:** `nq_data.daily_ohlcv_rth`, `nq_data.rth_firsthour_rest_ohlc_ranges`

**Difficulty Rating:** 3/5

**(ID: RTH-FH-007)**



WITH daily_agg_gaps AS (
SELECT 
	d.trade_date,
	d.weekday ,
	d.high - d.low AS daily_range,
	r.fh_high - r.fh_low AS fh_range,
	(r.fh_high - r.fh_low) / (d.high - d.low) AS fh_pct_of_day,
	CASE WHEN r.r_close > r.r_open THEN 1 ELSE 0 END AS rest_bullish,
	CASE WHEN d.OPEN > LAG(d.CLOSE) OVER (ORDER BY d.trade_date) THEN 'gap_up' ELSE 'gap_down' END AS gap_direction,
	abs(d.OPEN - (LAG(d.CLOSE) OVER (ORDER BY d.trade_date))) AS gap_size,
	ROUND(abs((d.CLOSE - d.low) / (d.high - d.low))::NUMERIC * 100, 2) AS close_location,
	CASE WHEN d.CLOSE > d.OPEN THEN 'bullish' ELSE 'bearish' END AS day_direction
FROM nq_data.daily_ohlcv_rth d
JOIN nq_data.rth_firsthour_rest_ohlc_ranges r ON d.trade_date = r.trade_date
),
ntile_agg AS (
SELECT 
	*,
	ntile(3) OVER (ORDER BY gap_size) AS gap_bucket
FROM daily_agg_gaps
)
SELECT 
	gap_bucket,
	gap_direction,
	COUNT(*),
	ROUND(AVG(gap_size), 2) AS avg_gap_size,
	ROUND(AVG(fh_range), 2) AS avg_fh_range,
	ROUND(AVG(daily_range), 2) AS avg_full_day_range,
	ROUND(AVG(fh_pct_of_day), 2) AS avg_fh_pct_of_day,
	ROUND(COUNT(*) FILTER (WHERE close_location > 50) / COUNT (*)::NUMERIC * 100, 2) AS pct_closed_upper_half,
	ROUND(sum(rest_bullish) / COUNT(*)::NUMERIC * 100, 2) AS rest_bullish_pct
FROM ntile_agg
WHERE gap_size IS NOT NULL
GROUP BY gap_bucket, gap_direction

gap_bucket	gap_direction	count	avg_gap_size	avg_fh_range	avg_full_day_range	avg_fh_pct_of_day	pct_closed_upper_half	rest_bullish_pct
2	gap_down	35	117.94	219.65	362.36	0.62	68.57	60
3	gap_up	34	292.14	186.32	331.91	0.62	44.12	50
2	gap_up	27	112.92	194.95	327.02	0.63	48.15	48.15
1	gap_down	26	29.13	176.38	286.35	0.64	61.54	61.54
1	gap_up	36	34.15	166.98	291.76	0.65	55.56	44.44
3	gap_down	27	308.74	265.06	431.96	0.63	77.78	85.19


WITH daily_agg_gaps AS (
SELECT 
	d.trade_date,
	d.weekday ,
	d.high - d.low AS daily_range,
	r.fh_high - r.fh_low AS fh_range,
	(r.fh_high - r.fh_low) / (d.high - d.low) AS fh_pct_of_day,
	CASE WHEN r.r_close > r.r_open THEN 1 ELSE 0 END AS rest_bullish,
	CASE WHEN d.OPEN > LAG(d.CLOSE) OVER (ORDER BY d.trade_date) THEN 'gap_up' ELSE 'gap_down' END AS gap_direction,
	abs(d.OPEN - (LAG(d.CLOSE) OVER (ORDER BY d.trade_date))) AS gap_size,
	ROUND(abs((d.CLOSE - d.low) / (d.high - d.low))::NUMERIC * 100, 2) AS close_location,
	CASE WHEN d.CLOSE > d.OPEN THEN 'bullish' ELSE 'bearish' END AS day_direction
FROM nq_data.daily_ohlcv_rth d
JOIN nq_data.rth_firsthour_rest_ohlc_ranges r ON d.trade_date = r.trade_date
),
ntile_agg AS (
SELECT 
	*,
	ntile(3) OVER (ORDER BY gap_size) AS gap_bucket
FROM daily_agg_gaps
WHERE weekday = 'Friday'
)
SELECT 
	gap_bucket,
	gap_direction,
	COUNT(*),
	ROUND(AVG(gap_size), 2) AS avg_gap_size,
	ROUND(AVG(fh_range), 2) AS avg_fh_range,
	ROUND(AVG(daily_range), 2) AS avg_full_day_range,
	ROUND(AVG(fh_pct_of_day), 2) AS avg_fh_pct_of_day,
	ROUND(COUNT(*) FILTER (WHERE close_location > 50) / COUNT (*)::NUMERIC * 100, 2) AS pct_closed_upper_half,
	ROUND(sum(rest_bullish) / COUNT(*)::NUMERIC * 100, 2) AS rest_bullish_pct
FROM ntile_agg
WHERE gap_size IS NOT NULL
GROUP BY gap_bucket, gap_direction

gap_bucket	gap_direction	count	avg_gap_size	avg_fh_range	avg_full_day_range	avg_fh_pct_of_day	pct_closed_upper_half	rest_bullish_pct
1	gap_down	2	37.25	246.38	350	0.7	100	0
2	gap_down	7	133.04	247.04	379.5	0.64	71.43	57.14
1	gap_up	10	39.08	171.98	339.13	0.7	50	40
3	gap_down	6	321.33	298.75	492.17	0.62	66.67	66.67
3	gap_up	5	252.8	177.7	315.55	0.58	60	60
2	gap_up	5	112.85	184.6	338.2	0.56	20	40

---

## Submission Instructions

Paste your queries and results. Log query IDs: RTH-CLOSE-003, RTH-FH-007.
