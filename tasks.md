# NQ Project — Week 29 Day 5

**Generated:** 2026-07-11
**Focus:** EU low undercut by weekday × gap direction (RTH-GLOB-005) + post-LOD breakdown depth (RTH-SESS-009)

---

## Task 1: EU Low Undercut Rate by Weekday × Gap Direction (Bullish RTH Days)

**Scenario:**
RTH-GLOB-002 showed EU low undercut rates by weekday (Monday 40%, Thursday 66.7%) and RTH-GLOB-004 showed EU low undercut rates by gap direction (gap-down 60.98%, gap-up 34.09%) — but these two dimensions haven't been combined. The Friday gap-down question is: when Friday gaps down and ends bullish, does RTH undercut the EU low (RTH opened below EU range already, EU low is close by) or does the EU low hold as the actual day low? The weekday × gap direction cross-tab answers this for all five weekdays simultaneously. **(ID: RTH-GLOB-005)**

Extend RTH-GLOB-002 — add `gap_direction` (rth_open > prev_close = gap_up, else gap_down) via LAG(close). Filter to bullish RTH days (rth_close > rth_open). GROUP BY weekday + gap_direction.

Output columns:
- `weekday`
- `gap_direction`
- `days`
- `avg_rth_open_location`
- `pct_undercut_eu_mid`
- `pct_undercut_eu_low`
- `pct_above_eu_high`

**Finding to answer:** On gap-down Monday (bullish), does the EU low almost always hold (RTH opened below it, EU low = today's low)? On gap-down Thursday (bullish), does it always get undercut (structural dip)? On gap-down Friday (bullish), which side does it land on? Does the weekday × gap direction interaction produce clearly different EU level behavior than either dimension alone?

**Tables:** `nq_data.ticks`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 3/5

**(ID: RTH-GLOB-005)**


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
	CASE WHEN d.low < e.eu_midpoint THEN 1 ELSE 0 END AS reached_eu_midpoint,
	CASE WHEN d.low < e.eu_low THEN 1 ELSE 0 END AS reached_eu_low,
	CASE WHEN d.high > e.eu_high THEN 1 ELSE 0 END AS reached_eu_high
FROM eu_levels_aggregation e
JOIN nq_data.daily_ohlcv_rth d ON e.trade_date = d.trade_date
),
prev_day_direction_agg AS (
SELECT 
	*,
	CASE WHEN rth_open > rth_close THEN 'bullish' ELSE 'bearish' END AS day_direction,
	CASE WHEN rth_open > prev_close THEN 'gap_up' ELSE 'gap_down' END AS gap_direction
FROM eu_us_joint_agg
WHERE prev_close IS NOT NULL AND prev_open IS NOT NULL
)
SELECT
	gap_direction,
	weekday,
	COUNT(*) AS days,
	ROUND(AVG(eu_range), 2) AS avg_eu_range,
	ROUND(SUM(reached_eu_midpoint)/ COUNT(*)::NUMERIC * 100, 2) AS avg_eu_range,
	ROUND(AVG(rth_open_location), 2) AS avg_rth_open_location,
	ROUND(SUM(reached_eu_low)/ COUNT(*)::NUMERIC * 100, 2) AS pct_undercut_eu_low,
	ROUND(SUM(reached_eu_high)/ COUNT(*)::NUMERIC * 100, 2) AS pct_went_above_eu_high
FROM prev_day_direction_agg
GROUP BY gap_direction, weekday


I've done it not for bullish only at first:

gap_direction	weekday	days	avg_eu_range	avg_eu_range	avg_rth_open_location	pct_undercut_eu_low	pct_went_above_eu_high
gap_down	Friday	11	308.89	90.91	31.99	81.82	54.55
gap_down	Monday	14	201.07	78.57	54.13	64.29	71.43
gap_down	Thursday	18	208.65	100	30.39	88.89	33.33
gap_down	Tuesday	18	213.79	100	41.48	77.78	72.22
gap_down	Wednesday	11	163.66	81.82	45.76	54.55	54.55
gap_up	Friday	18	191.71	83.33	59.38	55.56	83.33
gap_up	Monday	18	206.21	66.67	71.87	50	88.89
gap_up	Thursday	13	184.27	92.31	73.31	61.54	61.54
gap_up	Tuesday	15	132.83	66.67	54.61	60	66.67
gap_up	Wednesday	22	161.94	86.36	58.91	63.64	68.18



And then for bullish only RTH days:

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
	CASE WHEN d.low < e.eu_midpoint THEN 1 ELSE 0 END AS reached_eu_midpoint,
	CASE WHEN d.low < e.eu_low THEN 1 ELSE 0 END AS reached_eu_low,
	CASE WHEN d.high > e.eu_high THEN 1 ELSE 0 END AS reached_eu_high
FROM eu_levels_aggregation e
JOIN nq_data.daily_ohlcv_rth d ON e.trade_date = d.trade_date
),
prev_day_direction_agg AS (
SELECT 
	*,
	CASE WHEN rth_open > rth_close THEN 'bullish' ELSE 'bearish' END AS day_direction,
	CASE WHEN rth_open > prev_close THEN 'gap_up' ELSE 'gap_down' END AS gap_direction
FROM eu_us_joint_agg
WHERE prev_close IS NOT NULL AND prev_open IS NOT NULL
)
SELECT
	gap_direction,
	weekday,
	COUNT(*) AS days,
	ROUND(AVG(eu_range), 2) AS avg_eu_range,
	ROUND(SUM(reached_eu_midpoint)/ COUNT(*)::NUMERIC * 100, 2) AS avg_eu_range,
	ROUND(AVG(rth_open_location), 2) AS avg_rth_open_location,
	ROUND(SUM(reached_eu_low)/ COUNT(*)::NUMERIC * 100, 2) AS pct_undercut_eu_low,
	ROUND(SUM(reached_eu_high)/ COUNT(*)::NUMERIC * 100, 2) AS pct_went_above_eu_high
FROM prev_day_direction_agg
WHERE rth_close > rth_open
GROUP BY gap_direction, weekday





gap_direction	weekday	days	avg_eu_range	avg_eu_range	avg_rth_open_location	pct_undercut_eu_low	pct_went_above_eu_high
gap_down	Friday	7	297.54	85.71	32.33	71.43	71.43
gap_down	Monday	10	195.03	70	57.06	50	80
gap_down	Thursday	7	228.57	100	29.93	71.43	71.43
gap_down	Tuesday	11	218.89	100	42	81.82	81.82
gap_down	Wednesday	6	144.21	66.67	53.26	16.67	100
gap_up	Friday	10	231.93	70	58.94	30	90
gap_up	Monday	9	156	44.44	66.73	33.33	100
gap_up	Thursday	5	166.15	100	64.78	60	80
gap_up	Tuesday	8	166.13	37.5	65.43	25	100
gap_up	Wednesday	12	169.88	75	57.95	33.33	100


---

## Task 2: Post-LOD Breakdown Depth and Recovery

**Scenario:**
The day's low is set in the 09:30 window 33-37% of the time (RTH-SESS-005). But what happens when price has NOT yet set the day's low by 10:30 and then breaks below the pre-10:30 low later in the session? This is the "late LOD breakdown" — a genuine intraday event. Once price crosses below the pre-10:30 session low after 10:30 ET, how deep does the extension go, and does price recover back to the RTH open or day high before close? This gives a framework for late-session LOD breakdowns — know whether to fade or hold through. **(ID: RTH-SESS-009)**

**Architecture:**
1. CTE 1 — compute the pre-10:30 session low per trade_date from raw ticks (MIN(price) where ts_event ET time < 10:30), joined via session_start/session_end from daily_ohlcv_rth to avoid AT TIME ZONE on 56M rows
2. CTE 2 — find the first tick after 10:30 ET where price < pre_1030_low (the breakdown moment). Use MIN(ts_event) with a filter on price < pre_1030_low and time >= 10:30
3. CTE 3 — from the breakdown tick onward, compute: MIN(price) as breakdown_low (max depth), and MAX(price) as post_breakdown_high (recovery)
4. Final SELECT — filter to days where a breakdown actually occurred. Compute:
   - `breakdown_depth` = pre_1030_low - breakdown_low (how far below the pre-10:30 low it extended)
   - `pct_recovered_to_rth_open` = did post_breakdown_high >= rth_open
   - `pct_recovered_to_day_high` = did post_breakdown_high >= daily_high (from daily_ohlcv_rth)
   - `avg_breakdown_depth`, `days_with_breakdown`, `pct_days_with_breakdown`

**Finding to answer:** How often does a late-session LOD breakdown occur? How deep is the typical extension below the pre-10:30 low? What % of breakdowns recover back to the RTH open before close? What % recover to the day's high? Is this a fade setup (recovery likely) or a continuation setup (extension likely)?

**Tables:** `nq_data.ticks`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 5/5

**(ID: RTH-SESS-009)**


WITH nq_ticks_hrs_dates AS (
SELECT 
	*,
	ts_event::date AS trade_date,
	TO_CHAR(DATE_TRUNC('hour', ts_recv AT TIME ZONE 'America/New_York') + (EXTRACT(MINUTE FROM ts_recv AT TIME ZONE 'America/New_York')::int / 30 * INTERVAL '30 Minutes'), 'HH24:MI') AS current_window_et
FROM nq_data.ticks t
WHERE (t.ts_event AT TIME ZONE 'America/New_York')::time < '10:30' AND (t.ts_event AT TIME ZONE 'America/New_York')::time > '9:30'
),
pre_1030_lows_dates AS (
SELECT 
	trade_date,
	MIN(price) AS pre_1030_low
FROM nq_ticks_hrs_dates
GROUP BY trade_date
),
low_breakdowns AS (
SELECT 
	*,
	ts_event::date AS trade_date,
	TO_CHAR(DATE_TRUNC('hour', ts_recv AT TIME ZONE 'America/New_York') + (EXTRACT(MINUTE FROM ts_recv AT TIME ZONE 'America/New_York')::int / 30 * INTERVAL '30 Minutes'), 'HH24:MI') AS current_window_et
FROM nq_data.ticks t
WHERE (t.ts_event AT TIME ZONE 'America/New_York')::time >= '10:30' AND (t.ts_event AT TIME ZONE 'America/New_York')::time < '16:00'
),
post_1030_lows_dates AS (
SELECT
	trade_date,
	MIN(price) AS post_1030_low,
	MAX(price) AS day_high
FROM low_breakdowns
GROUP BY trade_date
),
post_1030_lows_timestamps AS (
SELECT DISTINCT ON (p.trade_date, p.post_1030_low)
	p.trade_date,
	p.post_1030_low,
	day_high,
	t.ts_event AS post_1030_low_timestamp
FROM post_1030_lows_dates p
JOIN nq_data.ticks t ON p.post_1030_low = t.price AND p.trade_date = t.ts_event::date
),
post_breakdown_agg AS (
SELECT 
	t.ts_event::date AS trade_date,
	MAX(price) AS post_breakdown_high
FROM nq_data.ticks t
JOIN post_1030_lows_timestamps p ON t.ts_event::date = p.trade_date
WHERE (t.ts_event AT TIME ZONE 'America/New_York')::time >= p.post_1030_low_timestamp::time AND t.ts_event::date = p.trade_date
GROUP BY t.ts_event::DATE
),
double_low_agg AS (
SELECT 
	p.trade_date,
	p2.pre_1030_low,
	p.post_1030_low,
	p.post_1030_low_timestamp,
	p3.post_breakdown_high,
	CASE WHEN post_1030_low < pre_1030_low THEN 'broken' ELSE 'unbroken' END AS low_broken
FROM post_1030_lows_timestamps p
JOIN pre_1030_lows_dates p2 ON p.trade_date = p2.trade_date
JOIN post_breakdown_agg p3 ON p3.trade_date = p2.trade_date
),
pre_final_agg AS (
SELECT 
	d.trade_date,
	d.pre_1030_low,
	d.post_1030_low,
	ABS(d.pre_1030_low - d.post_1030_low) AS breakdown_depth,
	CASE WHEN d.post_breakdown_high > r.fh_open THEN 1 ELSE 0 END AS recovered_to_day_open,
	CASE WHEN d.post_breakdown_high > r.fh_high THEN 1 ELSE 0 END AS recovered_to_fh_high
FROM double_low_agg d
JOIN nq_data.rth_firsthour_rest_ohlc_ranges r ON d.trade_date = r.trade_date
WHERE post_1030_low < pre_1030_low
)
SELECT 
	COUNT(*) AS days,
	ROUND(AVG(breakdown_depth), 2) AS avg_breakdown_depth,
	ROUND(SUM(recovered_to_day_open) / COUNT(*)::NUMERIC * 100, 2) AS recovered_to_day_open_pct,
	ROUND(SUM(recovered_to_fh_high) / COUNT(*)::NUMERIC * 100, 2) AS recovered_to_fh_high_pct
FROM pre_final_agg


A very complex query that took some time - I didn't follow your instructions 1:1 and I added some things back as I proceeded. It all seems logical to me.

days	avg_breakdown_depth	recovered_to_day_open_pct	recovered_to_fh_high_pct
87	138.36	74.71	55.17

---

## Submission Instructions

Paste your queries and results. Log query IDs: RTH-GLOB-005, RTH-SESS-009.
