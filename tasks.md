# NQ Project — Week 28 Day 3

**Generated:** 2026-07-01
**Focus:** Choppy day detection (tight OR + FH) + session high/low timing by 30-min window

---

## Task 1: Choppy Day Detection — Tight OR + FH Joint Filter

**Scenario:**
Yesterday was a choppy session. RTH-ORB-009 and RTH-FH-006 both showed that tight OR/FH ranges predict quieter afternoons. Can we define a "low-conviction day" as one where both the OR range AND the FH range are in their smallest quartile (Q1), and characterize what that predicts for rest-of-session range, direction, and close location? Practical value: identify days to size down or avoid trading entirely. **(ID: RTH-SESS-004)**

Using `nq_data.or_rest_ohlc_ranges` joined to `nq_data.rth_firsthour_rest_ohlc_ranges`:

Compute:
- `or_range = or_high - or_low`
- `fh_range = fh_high - fh_low`
- `or_range_bucket` — NTILE(4) OVER (ORDER BY or_range)
- `fh_range_bucket` — NTILE(4) OVER (ORDER BY fh_range)

Define 3 day types:
- `both_tight` — or_range_bucket = 1 AND fh_range_bucket = 1
- `both_wide` — or_range_bucket = 4 AND fh_range_bucket = 4
- `mixed` — all other combinations

For each day type aggregate:
- `days`
- `avg_or_range`
- `avg_fh_range`
- `avg_rest_range` — r_high - r_low
- `rest_bullish_pct` — r_close > r_open
- `avg_close_location` — (r_close - r_low) / NULLIF(r_high - r_low, 0) — where does the rest-of-session close within its own range?

Order by `day_type`.

**Finding to answer:** How much smaller is the rest-of-session range on both_tight days vs both_wide? Is rest direction more or less predictable on choppy days? What's the close location — do tight days drift randomly or still close near one extreme?

**Tables:** `nq_data.or_rest_ohlc_ranges`, `nq_data.rth_firsthour_rest_ohlc_ranges`

**Difficulty Rating:** 4/5

**(ID: RTH-SESS-004)**



WITH fh_or_ranges_agg AS (
SELECT 
	o.trade_date,
	o.or_high - o.or_low AS or_range,
	r.fh_high  - r.fh_low  AS fh_range,
	r.r_high - r.r_low AS rest_range,
	r.r_high,
	r.r_low,
	r.r_close,
	r.r_open,
	NTILE(4) OVER (ORDER BY o.or_high - o.or_low) AS or_range_bucket,
	NTILE(4) OVER (ORDER BY r.fh_high  - r.fh_low) AS fh_range_bucket
FROM nq_data.or_rest_ohlc_ranges o
JOIN nq_data.rth_firsthour_rest_ohlc_ranges r ON o.trade_date = r.trade_date
),
pre_final_agg AS (
SELECT 
	*,
	CASE 
		WHEN or_range_bucket = 1 and fh_range_bucket = 1 THEN 'both tight'
		WHEN or_range_bucket = 4 and fh_range_bucket = 4 THEN 'both wide'
	ELSE 'mixed' END AS day_type
FROM fh_or_ranges_agg
)
SELECT 
	day_type,
	COUNT(*) AS days,
	ROUND(AVG(or_range), 2) AS avg_or_range,
	ROUND(AVG(fh_range), 2) AS avg_fh_range,
	ROUND(AVG(rest_range), 2) AS avg_rest_range,
	ROUND(COUNT(*) FILTER (WHERE r_close > r_open) / COUNT(*)::NUMERIC * 100, 2) AS rest_bullish_pct,
	ROUND(AVG((r_close - r_low) / NULLIF(r_high - r_low, 0)), 2) AS avg_close_location
FROM pre_final_agg
GROUP BY day_type


day_type	days	avg_or_range	avg_fh_range	avg_rest_range	rest_bullish_pct	avg_close_location
both wide	28	267.6	329.73	339.45	67.86	0.61
both tight	31	82.47	94.11	197.14	54.84	0.58
mixed	100	153.23	198.46	265.16	55	0.54


---

## Task 2: Session High/Low Timing by 30-Minute Window

**Scenario:**
At what time of day does the RTH high and low most commonly form? Knowing whether the morning or afternoon tends to set the day's extreme directly informs when to expect turning points and whether to hold positions through midday. **(ID: RTH-SESS-005)**

Using `nq_data.ticks` joined to `nq_data.daily_ohlcv_rth`:

For each trading day, find the **first time** the daily high and daily low were touched:
- Join ticks to `daily_ohlcv_rth` on trade_date
- Filter to RTH only: `(ts_event AT TIME ZONE 'America/New_York')::time >= '09:30'` AND `< '16:00'`
- For each trade_date, find `MIN(ts_event)` WHERE `price = daily_high` → this is `high_time`
- For each trade_date, find `MIN(ts_event)` WHERE `price = daily_low` → this is `low_time`

Then bucket `high_time` and `low_time` into 30-minute windows:
- `DATE_TRUNC('hour', time_et) + INTERVAL '30 min' * FLOOR(EXTRACT(MINUTE FROM time_et) / 30)` → gives window start (09:30, 10:00, 10:30, ... 15:30)

Aggregate:
- For each 30-min window: count of days where high was first touched in that window, count where low was first touched
- Express as % of total trading days

**Output:**
- `window_start` — 09:30, 10:00, 10:30, ... 15:30
- `high_formed_count`
- `high_formed_pct`
- `low_formed_count`
- `low_formed_pct`

Order by `window_start`.

**Hint:** This will scan 56M rows — consider using `nq_data.daily_ohlcv_rth` to get the daily high/low first, then join back to ticks to find timing. That way you're only scanning ticks for rows that match the daily extreme price, not computing extremes from ticks directly.

**Finding to answer:** Which 30-min window most commonly produces the day's high? Which produces the day's low? Is the opening window (09:30–10:00) dominant for both, or does the afternoon contribute meaningfully? Does the high tend to form earlier or later than the low?

**Tables:** `nq_data.ticks`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 5/5

**(ID: RTH-SESS-005)**



I've done it in abit circular way, but also I think my approach to hod_set and lod_set times was smarter and more convenient to understand, and I've done everything without your assistance. 

Definitely though those window extracts needed me to look up some past queries, as that's too freaking complex to understand easily.


Also I opted for ET times as it's the ET times that I'm interested in.



WITH hod_set_times AS (
SELECT 
	d.trade_date,
	min(t.ts_event) AS high_set_time
FROM nq_data.daily_ohlcv_rth d
JOIN nq_data.ticks t ON d.trade_date = t.ts_event::date AND t.ts_event >= d.session_start AND t.ts_event <= d.session_end
AND t.price = d.high
GROUP BY d.trade_date
),
lod_set_times AS (
SELECT 
	d.trade_date,
	min(t.ts_event) AS low_set_time
FROM nq_data.daily_ohlcv_rth d
JOIN nq_data.ticks t ON d.trade_date = t.ts_event::date AND t.ts_event >= d.session_start AND t.ts_event <= d.session_end
AND t.price = d.LOW
GROUP BY d.trade_date
),
hod_lod_agg AS (
SELECT 
	h.trade_date,
	h.high_set_time,
	l.low_set_time,
	TO_CHAR(DATE_TRUNC('hour', h.high_set_time AT TIME ZONE 'America/New_York') + (EXTRACT(MINUTE FROM h.high_set_time AT TIME ZONE 'America/New_York')::int / 30 * INTERVAL '30 Minutes'), 'HH24:MI') AS high_formation_window_et,
	TO_CHAR(DATE_TRUNC('hour', l.low_set_time AT TIME ZONE 'America/New_York') + (EXTRACT(MINUTE FROM l.low_set_time AT TIME ZONE 'America/New_York')::int / 30 * INTERVAL '30 Minutes'), 'HH24:MI') AS low_formation_window_et
FROM hod_set_times h
JOIN lod_set_times l ON h.trade_date = l.trade_date
),
hod_agg AS (
SELECT
	high_formation_window_et,
	COUNT(*) AS high_formed_days,
	(SELECT COUNT(*) FROM hod_lod_agg) AS total_days,
	ROUND(COUNT(*) / (SELECT COUNT(*) FROM hod_lod_agg)::NUMERIC * 100, 2) AS high_formed_pct
FROM hod_lod_agg
GROUP BY high_formation_window_et
ORDER BY high_formation_window_et
),
lod_agg AS (
SELECT
	low_formation_window_et,
	COUNT(*) AS low_formed_days,
	(SELECT COUNT(*) FROM hod_lod_agg) AS total_days,
	ROUND(COUNT(*) / (SELECT COUNT(*) FROM hod_lod_agg)::NUMERIC * 100, 2) AS low_formed_pct
FROM hod_lod_agg
GROUP BY low_formation_window_et
ORDER BY low_formation_window_et
)
SELECT 
	h.high_formation_window_et AS time_window,
	h.high_formed_days,
	h.high_formed_pct,
	l.low_formed_days,
	l.low_formed_pct
FROM hod_agg h
JOIN lod_agg l ON h.high_formation_window_et = l.low_formation_window_et



time_window	high_formed_days	high_formed_pct	low_formed_days	low_formed_pct
09:30	53	33.33	59	37.11
10:00	13	8.18	13	8.18
10:30	5	3.14	13	8.18
11:00	9	5.66	9	5.66
11:30	6	3.77	11	6.92
12:00	6	3.77	7	4.4
12:30	7	4.4	3	1.89
13:00	4	2.52	5	3.14
13:30	7	4.4	11	6.92
14:00	2	1.26	2	1.26
14:30	10	6.29	4	2.52
15:00	13	8.18	3	1.89
15:30	24	15.09	19	11.95




---

## Submission Instructions

Paste your queries and results. Log query IDs: RTH-SESS-004, RTH-SESS-005.
