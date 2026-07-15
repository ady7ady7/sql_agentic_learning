# NQ Project — Week 30 Day 3

**Generated:** 2026-07-15
**Focus:** EU high resistance weekday breakdown (RTH-GLOB-006b) + gap direction × bearish day × EU high (RTH-GLOB-007)

---

## Task 1: EU High Resistance on Bearish Days — Weekday Breakdown

**Scenario:**
RTH-GLOB-006 established that EU high holds ~60% of the time on bearish RTH days at the aggregate level. But the aggregate hides weekday variation — RTH-GLOB-002b already showed Wednesday bearish = 20% above EU high, Monday bearish = 69% above EU high. The question now: does the same weekday pattern hold when we look specifically at the EU high resistance question? Is Tuesday specifically a day where EU high is a reliable short entry, or does RTH regularly spike above it before selling off? **(ID: RTH-GLOB-006b)**

**Architecture:**
You already have the RTH-GLOB-006 query. Minimal change: add `d.weekday` to the SELECT and GROUP BY in the final aggregation. Remove the `prev_day_direction` grouping (or keep both). Report:
- `weekday`
- `days`
- `avg_rth_open_vs_eu_high`
- `pct_exceeded_eu_high` (= pct_went_above_eu_high from yesterday)
- `pct_undercut_eu_low`

**Finding to answer:** On bearish RTH days, which weekdays see EU high respected as resistance (RTH never trades above it) vs which weekdays regularly spike above EU high before selling? Is Tuesday's EU high a reliable short entry or a trap?

**Tables:** `nq_data.ticks`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 2/5

**(ID: RTH-GLOB-006b)**

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
	weekday,
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
GROUP BY prev_day_direction, weekday


weekday	prev_day_direction	days	avg_eu_range	avg_rth_open_vs_eu_high	avg_eu_range	avg_rth_open_location	pct_undercut_eu_low	pct_went_above_eu_high
Friday	bearish	8	188.13	-88.63	100	55.25	100	62.5
Monday	bearish	6	191.79	-21.17	100	90.19	66.67	83.33
Thursday	bearish	9	231.94	-133	88.89	40.9	77.78	33.33
Tuesday	bearish	5	193.2	-117.15	100	41.44	100	20
Wednesday	bearish	9	167.58	-67.17	100	58.19	100	22.22
Friday	bullish	4	235.38	-168.25	100	40.76	75	50
Monday	bullish	7	288.82	-131.89	85.71	48.45	85.71	57.14
Thursday	bullish	10	163.3	-62.23	100	59.85	90	20
Tuesday	bullish	9	126.44	-69.75	100	41.46	77.78	55.56
Wednesday	bullish	6	158.5	-95.29	100	43.46	100	16.67

---

## Task 2: Gap Direction × Bearish RTH Day × EU High

**Scenario:**
RTH-GLOB-006b gives the weekday breakdown. But gap direction was the dominant predictor on the bullish side (RTH-GLOB-004: 26.9pp spread). The same question applies to bearish days: does a gap-up bearish day behave differently from a gap-down bearish day in terms of EU high? A gap-up bearish day (opened above prior close, then sold off all day) may see RTH spike well above EU high at the open, while a gap-down bearish day may never recover to EU high at all. Understanding this splits the EU short entry thesis cleanly. **(ID: RTH-GLOB-007)**

**Architecture:**
Extension of RTH-GLOB-006 query — add `gap_direction` = CASE WHEN rth_open > prev_close THEN 'gap_up' ELSE 'gap_down' END (using LAG(close) for prev_close, same pattern as RTH-GLOB-004/005). GROUP BY `gap_direction` in final SELECT, optionally also by `weekday` if sample sizes allow.

Report:
- `gap_direction`
- `days`
- `avg_rth_open_vs_eu_high`
- `pct_exceeded_eu_high`
- `pct_undercut_eu_low`
- optionally: `weekday` as secondary group

**Finding to answer:** On bearish RTH days, does gap direction predict whether EU high gets exceeded? Gap-up bearish → RTH opens above EU levels, likely spikes above EU high early (short entry from EU high is suboptimal, wait for RTH). Gap-down bearish → RTH opens deep in EU range, EU high is far above, rarely reached (EU high short entry was already optimal during EU session).

**Tables:** `nq_data.ticks`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 3/5

**(ID: RTH-GLOB-007)**


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
	CASE WHEN prev_close > prev_open THEN 'bullish' ELSE 'bearish' END AS prev_day_direction,
	CASE WHEN rth_open > prev_close THEN 'gap_up' ELSE 'gap_down' END AS gap_direction
FROM eu_us_joint_agg
WHERE prev_close IS NOT NULL AND prev_open IS NOT NULL
)
SELECT
	gap_direction,
	COUNT(*) AS days,
	ROUND(AVG(eu_range), 2) AS avg_eu_range,
	ROUND(AVG(rth_open_vs_eu_high), 2) AS avg_rth_open_vs_eu_high,
	ROUND(SUM(reached_eu_midpoint)/ COUNT(*)::NUMERIC * 100, 2) AS avg_eu_range,
	ROUND(AVG(rth_open_location), 2) AS avg_rth_open_location,
	ROUND(SUM(reached_eu_low)/ COUNT(*)::NUMERIC * 100, 2) AS pct_undercut_eu_low,
	ROUND(SUM(reached_eu_high)/ COUNT(*)::NUMERIC * 100, 2) AS pct_went_above_eu_high
FROM prev_day_direction_agg
WHERE day_direction = 'bearish'
GROUP BY gap_direction

gap_direction	days	avg_eu_range	avg_rth_open_vs_eu_high	avg_eu_range	avg_rth_open_location	pct_undercut_eu_low	pct_went_above_eu_high
gap_down	31	216.48	-147.02	100	36.09	93.55	25.81
gap_up	42	171.23	-49.61	95.24	64.24	83.33	52.38




---

## Submission Instructions

Paste your queries and results. Log query IDs: RTH-GLOB-006b, RTH-GLOB-007.
