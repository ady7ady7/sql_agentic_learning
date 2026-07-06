# NQ Project — Week 29 Day 1

**Generated:** 2026-07-07
**Focus:** Monday LOD depth from 09:30 open (RTH-SESS-008) + range expansion curve by weekday (RTH-VOL-013)

---

## Task 1: Monday LOD Depth from 09:30 Open

**Scenario:**
RTH-SESS-006 showed 51.5% of Monday LODs form in the 09:30 window. RTH-FH-004 showed gap-down Monday → FH sets day LOW 66.7%. The setup is clear: buy the Monday opening low. But how deep does that dip typically go? If the average dip is 20 pts below the 09:30 open, that's a tight stop. If it's 100+ pts, you need a much wider entry approach. Knowing the typical LOD depth defines the practical entry zone and stop placement for the Monday buy-the-open setup. **(ID: RTH-SESS-008)**

Using `nq_data.daily_ohlcv_rth` — for each Monday, compute:
- `open_to_low` — open - low (how far the LOD is below the 09:30 open, in points; positive = LOD below open)
- `open_to_high` — high - open (how far the HOD is above the 09:30 open)
- `open_to_close` — close - open (net session move from open)

Aggregate across all Mondays:
- `days`
- `avg_open_to_low` — average dip depth below open
- `avg_open_to_high` — average extension above open
- `avg_open_to_close` — average net close vs open
- `pct_low_below_open` — % of Mondays where the LOD is below the 09:30 open (open > low)
- `pct_close_above_open` — % of Mondays where RTH closes above the 09:30 open

Then add a second cut — split by gap direction (gap_up vs gap_down, using LAG(close) for prev_close):
- Same metrics per gap direction to see if the dip depth differs on gap-down vs gap-up Mondays

**Finding to answer:** How deep is the typical Monday dip below the 09:30 open? Does the LOD go significantly below the open even on gap-up Mondays? How does dip depth differ between gap-up and gap-down Mondays? What's the avg close vs open — confirming the bullish drift?

**Tables:** `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 3/5

**(ID: RTH-SESS-008)**


WITH mondays_fh_rest AS (
SELECT DISTINCT ON (r.trade_date)
	r.trade_date,
	d.weekday,
	r.fh_open,
	r.fh_close,
	r.fh_low,
	r.fh_high,
	d.CLOSE AS rest_close,
	d.low AS daily_low,
	ABS(fh_low - fh_open) AS open_to_low,
	ABS(fh_open - fh_high) AS open_to_high,
	ABS(fh_open - fh_close) AS open_to_close
FROM nq_data.rth_firsthour_rest_ohlc_ranges r
JOIN nq_data.daily_ohlcv_rth d ON r.trade_date = d.trade_date 
WHERE d.weekday = 'Monday'
)
SELECT 
 COUNT(*) AS days,
 ROUND(AVG(open_to_low), 2) AS avg_open_to_low,
 ROUND(AVG(open_to_high), 2) AS open_to_high,
 ROUND(AVG(open_to_close), 2) AS avg_open_to_close,
 ROUND(COUNT(*) FILTER (WHERE daily_low < fh_open) / COUNT(*)::NUMERIC * 100, 2) AS pct_low_below_open,
 ROUND(COUNT(*) FILTER (WHERE rest_close > fh_open) / COUNT(*)::NUMERIC * 100, 2) AS pct_close_above_open
FROM mondays_fh_rest



days	avg_open_to_low	open_to_high	avg_open_to_close	pct_low_below_open	round
33	74.55	114.71	113.22	100	60.61


And now the second cut - I hope it's done properly as I was running it down without testing each step, editing the whole query in one go :))


WITH mondays_fh_rest AS (
SELECT DISTINCT ON (r.trade_date)
	r.trade_date,
	d.weekday,
	r.fh_open,
	r.fh_close,
	r.fh_low,
	r.fh_high,
	d.CLOSE AS rest_close,
	d.low AS daily_low,
	LAG(d.close) OVER (ORDER BY r.trade_date) AS prev_close,
	ABS(fh_low - fh_open) AS open_to_low,
	ABS(fh_open - fh_high) AS open_to_high,
	ABS(fh_open - fh_close) AS open_to_close
FROM nq_data.rth_firsthour_rest_ohlc_ranges r
JOIN nq_data.daily_ohlcv_rth d ON r.trade_date = d.trade_date 
),
gap_calc AS (
SELECT 
	*,
	CASE WHEN prev_close > fh_open THEN 'gap_down' ELSE 'gap_up' END AS gap_direction
FROM mondays_fh_rest
),
monday_gap_agg AS (
SELECT * FROM gap_calc
WHERE weekday = 'Monday'
),
first_agg AS (
SELECT
	 gap_direction,
	 COUNT(*) AS days,
	 ROUND(AVG(open_to_low), 2) AS avg_open_to_low,
	 ROUND(AVG(open_to_high), 2) AS open_to_high,
	 ROUND(AVG(open_to_close), 2) AS avg_open_to_close,
	 ROUND(COUNT(*) FILTER (WHERE daily_low < fh_open) / COUNT(*)::NUMERIC * 100, 2) AS pct_low_below_open,
	 ROUND(COUNT(*) FILTER (WHERE rest_close > fh_open) / COUNT(*)::NUMERIC * 100, 2) AS pct_close_above_open
FROM monday_gap_agg
GROUP BY gap_direction
)
SELECT * FROM first_agg



gap_direction	days	avg_open_to_low	open_to_high	avg_open_to_close	pct_low_below_open	pct_close_above_open
gap_down	14	65.27	140.82	108.93	100	71.43
gap_up	19	81.39	95.47	116.38	100	52.63



---

## Task 2: Range Expansion Curve by Weekday

**Scenario:**
RTH-VOL-011 gave the aggregate cumulative range expansion curve (51% captured by 09:30, median fully ranged by 13:30). But weekdays have structurally different intraday patterns — Thursday front-loads its HOD (45% in 09:30 window), Monday's HOD forms late (21% at 15:30). Does Thursday's range expansion curve look steeper early than Monday's? Does Monday keep expanding later into the session? A weekday split on the existing curve gives a more precise real-time sizing reference for each specific day. **(ID: RTH-VOL-013)**

Reuse the RTH-VOL-011 query structure — the same 5-CTE tick-based approach — and add `d.weekday` to the grouping.

Output: one row per `(weekday, time_window)`, ordered by weekday then time_window.

Compute the same two metrics per row:
- `avg_range_captured_pct`
- `median_range_captured_pct` — PERCENTILE_CONT(0.5)

**Note:** With ~29–39 days per weekday (vs 159 total), the per-window per-weekday cells will have small samples. Focus findings on the dominant shape of the curve per weekday rather than individual window values.

**Finding to answer:** Does Thursday's curve peak earlier (steeper 09:30–10:00 rise) than other weekdays? Does Monday's curve stay flatter early and keep expanding later? Which weekday has the most range locked in by 11:00, and which the least?

**Tables:** `nq_data.ticks`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 4/5

**(ID: RTH-VOL-013)**




WITH nq_ticks_hrs_dates AS (
SELECT 
	*,
	ts_event::date AS trade_date,
	TO_CHAR(DATE_TRUNC('hour', ts_recv AT TIME ZONE 'America/New_York') + (EXTRACT(MINUTE FROM ts_recv AT TIME ZONE 'America/New_York')::int / 30 * INTERVAL '30 Minutes'), 'HH24:MI') AS current_window_et
FROM nq_data.ticks t
WHERE (t.ts_event AT TIME ZONE 'America/New_York')::time >= '9:30' AND (t.ts_event AT TIME ZONE 'America/New_York')::time <= '16:30'
),
nq_ticks_hl_oc_times AS (
SELECT 
	trade_date,
	current_window_et,
	MIN(ts_event) AS window_open_time,
	MAX(ts_event) AS window_close_time,
	MAX(price) AS window_high,
	MIN(price) AS window_low
FROM nq_ticks_hrs_dates
GROUP BY trade_date, current_window_et
),
windows_ohlc_agg AS (
SELECT DISTINCT ON (trade_date, current_window_et)
	n.trade_date,
	n.current_window_et,
	t1.price AS window_open,
	t2.price AS window_close,
	n.window_high,
	n.window_low,
	n.window_high - n.window_low AS window_range
FROM nq_ticks_hl_oc_times n
JOIN nq_data.ticks t1 ON n.window_open_time = t1.ts_event
JOIN nq_data.ticks t2 ON n.window_close_time = t2.ts_event
),
ranges_running_agg AS (
SELECT 
	w.trade_date,
	d.weekday,
	w.current_window_et,
	w.window_open,
	w.window_close,
	w.window_high,
	w.window_low,
	w.window_range,
	MAX(window_high) OVER (PARTITION BY w.trade_date ORDER BY current_window_et) AS running_high,
	MIN(window_low) OVER (PARTITION BY w.trade_date ORDER BY current_window_et) AS running_low,
	d.high - d.low AS daily_range
FROM windows_ohlc_agg w
JOIN nq_data.daily_ohlcv_rth d ON w.trade_date = d.trade_date
),
running_agg AS (
SELECT 
	*,
	running_high - running_low AS running_range,
	ROUND((running_high - running_low) / daily_range::NUMERIC * 100, 2) AS daily_range_pct
FROM ranges_running_agg
)
SELECT 
	weekday,
	current_window_et AS time_window,
	ROUND(AVG(daily_range_pct), 2) AS avg_range_captured_pct,
	PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY daily_range_pct) AS median_range_captured_pct
FROM running_agg
GROUP BY weekday, current_window_et

weekday	time_window	avg_range_captured_pct	median_range_captured_pct
Friday	09:30	47.68	49.11
Friday	10:00	64.04	60.32
Friday	10:30	74.19	82.31
Friday	11:00	79.76	84.4
Friday	11:30	83.61	90.93
Friday	12:00	86.48	94.5
Friday	12:30	89.81	97.62
Friday	13:00	92.64	100
Friday	13:30	96.68	100
Friday	14:00	97.42	100
Friday	14:30	97.67	100
Friday	15:00	98.37	100
Friday	15:30	100	100
Friday	16:00	100.72	100
Monday	09:30	56.1	57.25
Monday	10:00	66.85	68.83
Monday	10:30	73.7	73.56
Monday	11:00	78.81	78.59
Monday	11:30	82.25	81.96
Monday	12:00	86.3	90.2
Monday	12:30	88.3	95.06
Monday	13:00	88.89	91.7
Monday	13:30	90.49	95.06
Monday	14:00	91.82	97.58
Monday	14:30	93.59	100
Monday	15:00	97.08	100
Monday	15:30	100	100
Monday	16:00	100.37	100
Thursday	09:30	55.84	52.81
Thursday	10:00	65.87	62.54
Thursday	10:30	76.67	81.18
Thursday	11:00	80.11	83.71
Thursday	11:30	83.77	92.72
Thursday	12:00	85.6	93.35
Thursday	12:30	88.4	96.23
Thursday	13:00	90.4	97.88
Thursday	13:30	94.13	100
Thursday	14:00	95.5	100
Thursday	14:30	96.35	100
Thursday	15:00	98.59	100
Thursday	15:30	100	100
Thursday	16:00	101.47	100
Tuesday	09:30	52.41	53.2
Tuesday	10:00	62.32	63.98
Tuesday	10:30	67.12	69.71
Tuesday	11:00	73.81	73.37
Tuesday	11:30	78.12	77.11
Tuesday	12:00	81.4	79.34
Tuesday	12:30	84.15	83.2
Tuesday	13:00	88.22	87.4
Tuesday	13:30	91.6	96.19
Tuesday	14:00	92.43	96.19
Tuesday	14:30	93.48	96.19
Tuesday	15:00	94.94	99.67
Tuesday	15:30	100	100
Tuesday	16:00	102.18	100
Wednesday	09:30	43.25	41.38
Wednesday	10:00	57.13	54.33
Wednesday	10:30	65.75	65.905
Wednesday	11:00	70.48	69.895
Wednesday	11:30	77.35	81.12
Wednesday	12:00	79.59	85.295
Wednesday	12:30	82.32	87.845
Wednesday	13:00	84.83	89.835
Wednesday	13:30	86.18	99.11
Wednesday	14:00	88.77	100
Wednesday	14:30	94.9	100
Wednesday	15:00	97.66	100
Wednesday	15:30	100	100
Wednesday	16:00	105.41	100

---

## Submission Instructions

Paste your queries and results. Log query IDs: RTH-SESS-008, RTH-VOL-013.
