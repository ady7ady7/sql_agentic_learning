# NQ Project — Week 29 Day 4

**Generated:** 2026-07-10
**Focus:** EU session entry depth study — prior day direction (RTH-GLOB-003) + gap direction (RTH-GLOB-004) + bearish RTH mirror (RTH-GLOB-002b)

---

## Task 1: Prior Day Direction × EU Entry Quality

**Scenario:**
RTH-GLOB-002 showed that on bullish RTH days, 46.5% see the EU low undercut post-open. But does yesterday's RTH direction predict whether today's EU levels will hold? After a bullish prior day (strong context), EU lows may hold more often — continuation momentum carries through overnight. After a bearish prior day (weak context), RTH may dip harder post-open, undercutting EU levels more often. You know yesterday's close before EU even opens — making this a genuine pre-EU signal. **(ID: RTH-GLOB-003)**

Extend the RTH-GLOB-002 query structure — add `LAG(close > open) OVER (ORDER BY trade_date)` to `daily_ohlcv_rth` to get `prev_bullish` (true/false), then label as `prev_day_bullish` / `prev_day_bearish`.

Filter to bullish RTH days (close > open) as before. Aggregate by `prev_day_direction`:
- `days`
- `avg_eu_range`
- `avg_rth_open_location`
- `pct_undercut_eu_midpoint`
- `pct_undercut_eu_low`

**Finding to answer:** After a bullish prior day, do EU lows hold more often on bullish RTH days (continuation context)? After a bearish prior day, does RTH dip harder and undercut EU levels more frequently? Does prior day direction meaningfully split the RTH-GLOB-002 aggregate?

**Tables:** `nq_data.ticks`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 3/5

**(ID: RTH-GLOB-003)**

The bullish RTH day filter doesn't make sense at the beginning, as it makes EVERY day a fucking bullish day in the previous day direciton and ita lso filters out all bearish days at the beginning, giving us skewed results - it doesn't make sense.

I've created another filter to properly check that:


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
	CASE WHEN rth_close > rth_open THEN 'bullish' ELSE 'bearish' END AS day_direction,
	CASE WHEN prev_close > prev_open THEN 'bullish' ELSE 'bearish' END AS prev_day_direction
FROM eu_us_joint_agg
WHERE prev_close IS NOT NULL AND prev_open IS NOT NULL
)
SELECT
	prev_day_direction,
	COUNT(*) AS days,
	ROUND(AVG(eu_range), 2) AS avg_eu_range,
	ROUND(SUM(reached_eu_midpoint)/ COUNT(*)::NUMERIC * 100, 2) AS avg_eu_range,
	ROUND(AVG(rth_open_location), 2) AS avg_rth_open_location,
	ROUND(SUM(reached_eu_low)/ COUNT(*)::NUMERIC * 100, 2) AS pct_undercut_eu_low,
	ROUND(SUM(reached_eu_high)/ COUNT(*)::NUMERIC * 100, 2) AS pct_went_above_eu_high
FROM prev_day_direction_agg
WHERE day_direction = 'bullish'
GROUP BY prev_day_direction

prev_day_direction	days	avg_eu_range	avg_eu_range	avg_rth_open_location	pct_undercut_eu_low	pct_went_above_eu_high
bearish	37	213.26	78.38	52.69	43.24	89.19
bullish	48	186.19	70.83	53.56	50	87.5

---

## Task 2: Gap Direction × EU Entry Quality

**Scenario:**
On bullish RTH days, gap direction into the RTH open likely explains a large portion of whether EU levels get undercut. On gap-down bullish days, RTH opens below EU range — the EU low was already the day's low before RTH started, so it almost certainly held. On gap-up bullish days, RTH opens above EU range or near EU high — the EU low is far below and RTH may dip back toward it post-open. Splitting RTH-GLOB-002 by gap direction may reveal that the "EU low undercut" finding is almost entirely driven by gap-up days. **(ID: RTH-GLOB-004)**

Extend the RTH-GLOB-002 query — add `gap_direction` (open > prev_close = gap_up, else gap_down) using LAG(close). Filter to bullish RTH days. Aggregate by `gap_direction`:
- `days`
- `avg_gap_size`
- `avg_rth_open_location`
- `pct_undercut_eu_midpoint`
- `pct_undercut_eu_low`

**Finding to answer:** On gap-down bullish days, does the EU low almost never get undercut (RTH opened below it already)? On gap-up bullish days, does RTH frequently dip back below EU levels post-open? Does gap direction explain most of the variance in the RTH-GLOB-002 aggregate?


**Tables:** `nq_data.ticks`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 3/5

**(ID: RTH-GLOB-004)**

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
	CASE WHEN rth_open < rth_close THEN 'bullish' ELSE 'bearish' END AS day_direction,
	CASE WHEN rth_open > prev_close THEN 'gap_up' ELSE 'gap_down' END AS gap_direction
FROM eu_us_joint_agg
WHERE prev_close IS NOT NULL AND prev_open IS NOT NULL
)
SELECT
	gap_direction,
	COUNT(*) AS days,
	ROUND(AVG(eu_range), 2) AS avg_eu_range,
	ROUND(SUM(reached_eu_midpoint)/ COUNT(*)::NUMERIC * 100, 2) AS avg_eu_range,
	ROUND(AVG(rth_open_location), 2) AS avg_rth_open_location,
	ROUND(SUM(reached_eu_low)/ COUNT(*)::NUMERIC * 100, 2) AS pct_undercut_eu_low,
	ROUND(SUM(reached_eu_high)/ COUNT(*)::NUMERIC * 100, 2) AS pct_went_above_eu_high
FROM prev_day_direction_agg
WHERE day_direction = 'bullish'
GROUP BY gap_direction

gap_direction	days	avg_eu_range	avg_eu_range	avg_rth_open_location	pct_undercut_eu_low	pct_went_above_eu_high
gap_down	41	217.22	85.37	43.61	60.98	80.49
gap_up	44	180.03	63.64	62.11	34.09	95.45



---

## Task 3 (Auxiliary): EU Session Levels vs RTH on Bearish Days

**Scenario:**
Mirror of RTH-GLOB-002 for bearish RTH days (RTH close < RTH open). On days where NQ closes bearish, does RTH trade above the EU high post-open — giving a better short entry than anything available during EU? **(ID: RTH-GLOB-002b)**

Identical query to RTH-GLOB-002 with two changes:
- Filter: `close < open` (bearish days)
- Undercut flags become `reached_eu_high` (d.high > eu_high) and `reached_eu_midpoint` (d.high > eu_midpoint)
- `rth_open_location` same formula — where does RTH open sit in EU range

Aggregate overall + by weekday.

**Finding to answer:** On bearish RTH days, does RTH trade above the EU high post-open (giving a better short than EU offered)? Which weekday sees the EU high most often exceeded after the open on bearish days?

**Tables:** `nq_data.ticks`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 2/5 (one character change + flag inversion)

**(ID: RTH-GLOB-002b)**

With weekday split:


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
	ROUND((d.OPEN - e.eu_low) / e.eu_range::NUMERIC * 100, 2) AS rth_open_location,
	CASE WHEN d.low < e.eu_midpoint THEN 1 ELSE 0 END AS reached_eu_midpoint,
	CASE WHEN d.low < e.eu_low THEN 1 ELSE 0 END AS reached_eu_low,
	CASE WHEN d.high > e.eu_high THEN 1 ELSE 0 END AS reached_eu_high
FROM eu_levels_aggregation e
JOIN nq_data.daily_ohlcv_rth d ON e.trade_date = d.trade_date
WHERE d."close" < d.OPEN
)
SELECT
	weekday,
	COUNT(*) AS days,
	ROUND(AVG(eu_range), 2) AS avg_eu_range,
	ROUND(AVG(rth_open_location), 2) AS avg_rth_open_location,
	ROUND(SUM(reached_eu_midpoint) / COUNT(*)::NUMERIC * 100, 2) AS pct_undercut_eu_midpoint,
	ROUND(SUM(reached_eu_low) / COUNT(*)::NUMERIC * 100, 2)pct_undercut_eu_low,
	ROUND(SUM(reached_eu_high) / COUNT(*)::NUMERIC * 100, 2)pct_reached_eu_high
FROM eu_us_joint_agg
GROUP BY weekday


weekday	days	avg_eu_range	avg_rth_open_location	pct_undercut_eu_midpoint	pct_undercut_eu_low	pct_reached_eu_high
Friday	12	203.88	50.42	100	91.67	58.33
Monday	13	244.04	67.72	92.31	76.92	69.23
Thursday	19	195.82	50.87	94.74	84.21	26.32
Tuesday	14	150.29	41.45	100	85.71	42.86
Wednesday	15	163.95	52.29	100	100	20


without the weekday split:

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
	e.eu_high,
	e.eu_low,
	e.eu_midpoint,
	e.eu_range,
	d.OPEN AS rth_open,
	d."close" AS rth_close,
	d.low AS rth_low,
	d.high AS rth_high,
	ROUND((d.OPEN - e.eu_low) / e.eu_range::NUMERIC * 100, 2) AS rth_open_location,
	CASE WHEN d.low < e.eu_midpoint THEN 1 ELSE 0 END AS reached_eu_midpoint,
	CASE WHEN d.low < e.eu_low THEN 1 ELSE 0 END AS reached_eu_low,
	CASE WHEN d.high > e.eu_high THEN 1 ELSE 0 END AS reached_eu_high
FROM eu_levels_aggregation e
JOIN nq_data.daily_ohlcv_rth d ON e.trade_date = d.trade_date
WHERE d."close" < d.OPEN
)
SELECT
	COUNT(*) AS days,
	ROUND(AVG(eu_range), 2) AS avg_eu_range,
	ROUND(AVG(rth_open_location), 2) AS avg_rth_open_location,
	ROUND(SUM(reached_eu_midpoint) / COUNT(*)::NUMERIC * 100, 2) AS pct_undercut_eu_midpoint,
	ROUND(SUM(reached_eu_low) / COUNT(*)::NUMERIC * 100, 2)pct_undercut_eu_low,
	ROUND(SUM(reached_eu_high) / COUNT(*)::NUMERIC * 100, 2)pct_reached_eu_high
FROM eu_us_joint_agg

days	avg_eu_range	avg_rth_open_location	pct_undercut_eu_midpoint	pct_undercut_eu_low	pct_reached_eu_high
73	190.45	52.28	97.26	87.67	41.1


---

## Submission Instructions

Paste your queries and results. Log query IDs: RTH-GLOB-003, RTH-GLOB-004, RTH-GLOB-002b.
