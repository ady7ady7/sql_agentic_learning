# NQ Project — Week 29 Day 3

**Generated:** 2026-07-09
**Focus:** EU session levels vs RTH price action on bullish days (RTH-GLOB-002)

---

## Task 1: EU Session Levels vs RTH on Bullish Days

**Scenario:**
On days where NQ closes bullish (RTH close > RTH open), was the EU session (02:00–09:30 ET) a useful entry window, or does RTH routinely undercut EU levels after the open — giving a better price to anyone who just waited? If RTH takes out the EU low on 60% of bullish days, entering long during EU was suboptimal. If RTH rarely undercuts the EU 50% level, EU entries above midrange were well-placed. **(ID: RTH-GLOB-002)**

**Step 1 — EU session aggregates**
Filter ticks to EU session: `(ts_event AT TIME ZONE 'America/New_York')::time >= '02:00' AND < '09:30'`. Trade_date = `ts_event::date` (EU ticks on 2026-07-09 02:00–09:30 belong to trade_date 2026-07-09).

Per trade_date compute:
- `eu_high` — MAX(price)
- `eu_low` — MIN(price)
- `eu_range` — eu_high - eu_low
- `eu_midpoint` — (eu_high + eu_low) / 2

**Step 2 — RTH open location in EU range**
Join to `nq_data.daily_ohlcv_rth` on trade_date. Compute:
- `rth_open_location` — (open - eu_low) / NULLIF(eu_range, 0) — where does the RTH open sit within the EU range? 0 = at EU low, 1 = at EU high, 0.5 = at EU midpoint

**Step 3 — Post-open price action**
For each trade_date, find the RTH session MIN(price) after 09:30 (using ticks filtered to RTH: `ts_event >= session_start AND ts_event <= session_end` from `daily_ohlcv_rth`). This is `rth_low_after_open`.

Compute:
- `undercut_eu_low` — rth_low_after_open < eu_low (1/0)
- `undercut_eu_midpoint` — rth_low_after_open < eu_midpoint (1/0)

**Step 4 — Filter to bullish RTH days and aggregate**
Filter: RTH close > RTH open. Aggregate across all bullish days:
- `days`
- `avg_eu_range`
- `avg_rth_open_location` — avg position of RTH open within EU range
- `pct_undercut_eu_low` — % of bullish days where RTH traded below EU low after open
- `pct_undercut_eu_midpoint` — % of bullish days where RTH traded below EU midpoint after open

**Then add a weekday breakdown** — same metrics grouped by weekday.

**Finding to answer:** On bullish RTH days, does RTH routinely undercut the EU low or EU midpoint after the open? If yes — waiting for RTH open gives a better long entry than entering during EU. If no — the EU low/midpoint held and EU session entries were at better prices than anything available after 09:30.

**Tables:** `nq_data.ticks`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 5/5

**(ID: RTH-GLOB-002)**


Cut1:

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
	CASE WHEN d.low < e.eu_low THEN 1 ELSE 0 END AS reached_eu_low
FROM eu_levels_aggregation e
JOIN nq_data.daily_ohlcv_rth d ON e.trade_date = d.trade_date
WHERE d."close" > d.OPEN
)
SELECT
	COUNT(*) AS days,
	ROUND(AVG(eu_range), 2) AS avg_eu_range,
	ROUND(AVG(rth_open_location), 2) AS avg_rth_open_location,
	ROUND(SUM(reached_eu_midpoint) / COUNT(*)::NUMERIC * 100, 2) AS pct_undercut_eu_midpoint,
	ROUND(SUM(reached_eu_low) / COUNT(*)::NUMERIC * 100, 2)pct_undercut_eu_low
FROM eu_us_joint_agg

days	avg_eu_range	avg_rth_open_location	pct_undercut_eu_midpoint	pct_undercut_eu_low
86	196.91	53.67	73.26	46.51




Cut2:

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
	CASE WHEN d.low < e.eu_low THEN 1 ELSE 0 END AS reached_eu_low
FROM eu_levels_aggregation e
JOIN nq_data.daily_ohlcv_rth d ON e.trade_date = d.trade_date
WHERE d."close" > d.OPEN
)
SELECT
	weekday,
	COUNT(*) AS days,
	ROUND(AVG(eu_range), 2) AS avg_eu_range,
	ROUND(AVG(rth_open_location), 2) AS avg_rth_open_location,
	ROUND(SUM(reached_eu_midpoint) / COUNT(*)::NUMERIC * 100, 2) AS pct_undercut_eu_midpoint,
	ROUND(SUM(reached_eu_low) / COUNT(*)::NUMERIC * 100, 2)pct_undercut_eu_low
FROM eu_us_joint_agg
GROUP BY weekday


weekday	days	avg_eu_range	avg_rth_open_location	pct_undercut_eu_midpoint	pct_undercut_eu_low
Friday	17	258.94	47.98	76.47	47.06
Monday	20	173.06	63.31	55	40
Thursday	12	202.56	44.45	100	66.67
Tuesday	19	196.67	51.86	73.68	57.89
Wednesday	18	161.32	56.39	72.22	27.78



---

## Submission Instructions

Paste your query and results. Log query ID: RTH-GLOB-002.
