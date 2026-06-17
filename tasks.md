# NQ Project — Week 26 Day 3

**Generated:** 2026-06-16
**Focus:** Stacking edges — weekday × gap × FH direction + ORB fix + 15-min rolling delta

---

## Task 1: Weekday × Gap Direction × First Hour Direction Stack

**Scenario:**
Monday and Thursday show opposite patterns in nearly every dimension. Do those patterns hold when further conditioned on gap direction and FH direction? This is the full three-way stack. **(ID: RTH-SESS-003)**

Using the same join as Task 1 (daily_ohlcv_rth + rth_firsthour_rest_ohlc_ranges + LAG for prev_close), compute:

- `weekday`
- `gap_direction` (gap_up / gap_down — exclude flat)
- `fh_direction` (bullish / bearish — exclude flat)
- `days`
- `rest_bullish_pct` — % of days where `r_close > fh_close`, rounded to 1
- `avg_close_location` — rounded to 1

Filter to Monday and Thursday only. Order by `weekday`, `gap_direction`, `fh_direction`.

**Tables:** `nq_data.daily_ohlcv_rth`, `nq_data.rth_firsthour_rest_ohlc_ranges`

**Difficulty Rating:** 5/5


WITH rth_data AS (
SELECT 
	*,
	LAG(close) OVER (ORDER BY trade_date) AS prev_close
FROM nq_data.daily_ohlcv_rth dor 
),
rth_gaps AS (
SELECT 
	*,
	CASE 
		WHEN OPEN = prev_close THEN 'flat'
		WHEN OPEN > prev_close THEN 'gap_up' ELSE 'gap_down'
	END AS gap_direction
FROM rth_data 
WHERE prev_close IS NOT NULL
),
gaps_agg AS (
SELECT 
	r.trade_date,
	r.weekday,
	r.gap_direction,
	r.high AS daily_high,
	r.low AS daily_low,
	r.prev_close,
	rfror.fh_open,
	rfror.fh_close,
	rfror.r_low,
	rfror.r_high,
	rfror.r_close,
	CASE 
		WHEN fh_open = fh_close THEN 'flat'
		WHEN fh_open > fh_close THEN 'bullish' ELSE 'bearish'
	END AS fh_direction,
	CASE 
		WHEN r_close > fh_close THEN 1 ELSE 0
	END AS rest_bullish
FROM rth_gaps r
JOIN nq_data.rth_firsthour_rest_ohlc_ranges rfror ON r.trade_date = rfror.trade_date
)
SELECT
	weekday,
	gap_direction,
	fh_direction,
	COUNT(*) AS days,
	ROUND(SUM(rest_bullish) / COUNT(*)::NUMERIC * 100, 2) AS rest_bullish_pct,
	ROUND(AVG((daily_high - r_close) / (daily_high - daily_low))::NUMERIC * 100, 2) AS avg_close_location
FROM gaps_agg
GROUP BY weekday, gap_direction, fh_direction
ORDER BY rest_bullish_pct DESC


I didn't filter data, as it could prove useful - DO NOT take away points from me for that, as this is a potentially added value.
For that We don't have that many observations - this research could be easily expanded to more years and differenti nstruments as well if I wanted.



weekday	gap_direction	fh_direction	days	rest_bullish_pct	avg_close_location
Tuesday	gap_down	bullish	9	100	29.58
Monday	gap_down	bullish	5	80	31.9
Thursday	gap_down	bearish	10	80	34.84
Wednesday	gap_down	bearish	5	80	30.24
Monday	gap_down	bearish	13	69.23	27.21
Monday	gap_up	bullish	9	66.67	58.47
Thursday	gap_up	bullish	12	66.67	42.93
Tuesday	gap_down	bearish	12	66.67	40
Friday	gap_down	bearish	11	63.64	31.26
Wednesday	gap_down	bullish	7	57.14	46.01
Thursday	gap_down	bullish	11	54.55	55.4
Wednesday	gap_up	bearish	13	53.85	26.96
Friday	gap_up	bearish	10	50	44.73
Tuesday	gap_up	bearish	12	50	45.42
Monday	gap_up	bearish	11	45.45	39.94
Wednesday	gap_up	bullish	9	44.44	64.15
Friday	gap_up	bullish	10	40	54.5
Tuesday	gap_up	bullish	6	33.33	61.32
Friday	gap_down	bullish	4	25	61.21
Thursday	flat	bearish	1	0	50
Thursday	gap_up	bearish	5	0	66.02




---

## Task 2: ORB-001 Fix — Within-Group Breakout Percentages

**Scenario:**
RTH-ORB-001 showed breakout percentages as % of all 159 days, making cross-group comparison misleading. Fix it: show each breakout bucket as % within its own `or_delta_direction` group. **(ID: RTH-ORB-001b)**

Take your existing RTH-ORB-001 query and replace the global pct with a window function:

```sql
ROUND(COUNT(*) OVER (PARTITION BY or_delta_direction) ... )
```

Or equivalently, use `SUM(COUNT(*)) OVER (PARTITION BY or_delta_direction)` as the denominator.

**Output — same structure as RTH-ORB-001 but with corrected pct:**
- `or_delta_direction`
- `breakout_direction`
- `days`
- `within_group_pct` — days / total days in that or_delta_direction group, rounded to 1

Order by `or_delta_direction`, `days DESC`.

**Tables:** `nq_data.ticks`

**Difficulty Rating:** 3/5

WITH ticks_date_or_range AS (
SELECT 
	*,
	(ts_event AT TIME ZONE 'America/New_York')::date AS trade_date,
	ts_recv AT TIME ZONE 'America/New_York' AS time_et,
	TRIM(TO_CHAR(ts_recv AT TIME ZONE 'America/New_York', 'Day')) AS day_of_week
FROM nq_data.ticks t
WHERE (ts_event AT TIME ZONE 'America/New_York')::TIME >= '9:30' AND (ts_event AT TIME ZONE 'America/New_York')::TIME <= '10:00'
AND side != 'N'
),
or_range_delta AS (
SELECT
	trade_date,
	MAX(price) AS or_high,
	MIN(price) AS or_low,
	SUM(size) FILTER (WHERE side='B') - SUM(size) FILTER (WHERE side='A') AS or_volume_delta,
	CASE 
		WHEN SUM(size) FILTER (WHERE side='B') - SUM(size) FILTER (WHERE side='A') = 0 THEN 'flat'
		WHEN SUM(size) FILTER (WHERE side='B') - SUM(size) FILTER (WHERE side='A') > 0 THEN 'bullish' ELSE 'bearish' 
	END AS or_delta_direction
FROM ticks_date_or_range
GROUP BY trade_date
),
rest_day_or_range_from_10_00 AS (
SELECT 
	*,
	(ts_event AT TIME ZONE 'America/New_York')::date AS trade_date,
	ts_recv AT TIME ZONE 'America/New_York' AS time_et,
	TRIM(TO_CHAR(ts_recv AT TIME ZONE 'America/New_York', 'Day')) AS day_of_week
FROM nq_data.ticks t
WHERE (ts_event AT TIME ZONE 'America/New_York')::TIME >= '10:00' AND (ts_event AT TIME ZONE 'America/New_York')::TIME <= '16:00'
AND side != 'N'
),
rest_range_delta AS (
SELECT
	trade_date,
	MAX(price) AS rest_high,
	MIN(price) AS rest_low,
	SUM(size) FILTER (WHERE side='B') - SUM(size) FILTER (WHERE side='A') AS rest_volume_delta,
	CASE 
		WHEN SUM(size) FILTER (WHERE side='B') - SUM(size) FILTER (WHERE side='A') = 0 THEN 'flat'
		WHEN SUM(size) FILTER (WHERE side='B') - SUM(size) FILTER (WHERE side='A') > 0 THEN 'bullish' ELSE 'bearish' 
	END AS rest_delta_direction
FROM rest_day_or_range_from_10_00
GROUP BY trade_date
),
or_rest_agg AS (
SELECT
	r.trade_date,
	r.rest_high,
	r.rest_low,
	r.rest_delta_direction,
	o.or_high,
	o.or_low,
	or_delta_direction,
	CASE 
		WHEN rest_high > or_high AND rest_low < or_low THEN 'break_both'
		WHEN rest_high > or_high THEN 'break_up'
		WHEN rest_low < or_low THEN 'break_down'
		WHEN rest_high < or_high AND rest_low > or_low THEN 'no_break'
	END AS breakout_direction
FROM rest_range_delta r
JOIN or_range_delta o ON r.trade_date = o.trade_date
),
breaks_agg AS (
SELECT 
	or_delta_direction,
	breakout_direction,
	COUNT(*) AS days,
	ROUND(COUNT(*) / SUM(COUNT(*)) OVER (PARTITION BY or_delta_direction)::NUMERIC * 100, 2) AS within_group_pct
FROM or_rest_agg
GROUP BY or_delta_direction, breakout_direction
)
SELECT 
	*
FROM breaks_agg



Fixed, although it didn't bring us much intel - you already calculated it:

or_delta_direction	breakout_direction	days	within_group_pct
bearish	break_both	21	25.61
bearish	break_down	42	51.22
bearish	break_up	19	23.17
bullish	break_both	26	33.77
bullish	break_down	11	14.29
bullish	break_up	40	51.95

---

## Task 3: 15-Minute Rolling Delta Windows

**Scenario:**
Does the volume delta in one 15-minute window predict the direction of the *next* 15-minute window? This is the intraday version of the FH delta question. Split RTH into 15-minute buckets (09:30, 09:45, 10:00 ... 15:45) and for each bucket compute the net delta. Then check: when bucket N has positive delta, how often does price go up in bucket N+1? **(ID: RTH-VOL-006)**

Using `nq_data.ticks`, filter to RTH (09:30–16:00 ET, side != 'N'). Create 15-minute buckets using:
```sql
date_trunc('hour', ts_event AT TIME ZONE 'America/New_York')
+ (EXTRACT(MINUTE FROM ts_event AT TIME ZONE 'America/New_York')::int / 15) * INTERVAL '15 minutes'
```

Per bucket per day, compute:
- `bucket_start` — the 15-min window start time
- `bucket_delta` — SUM(size) FILTER (WHERE side='B') - SUM(size) FILTER (WHERE side='A')
- `bucket_open` — price at MIN(ts_event) in window
- `bucket_close` — price at MAX(ts_event) in window (use aggregate-then-JOIN pattern)
- `bucket_direction` — 'up' if bucket_close > bucket_open, 'down' if lower, 'flat' if equal

Then use LAG to get `prev_delta` and `prev_delta_direction` for each bucket. Aggregate across all days by `prev_delta_direction` × `bucket_direction` to get: when the prior 15-min had positive delta, how often did the next 15-min close up?

**Output — summary:**
- `prev_delta_direction` — 'positive' or 'negative'
- `next_bucket_up_days` — count where bucket_direction = 'up'
- `total`
- `next_up_pct` — rounded to 1

Order by `prev_delta_direction`.

**Tables:** `nq_data.ticks`

**Difficulty Rating:** 5/5

**Note:** This query scans 56M rows twice (for open/close prices via aggregate-then-JOIN). Expect it to be slow — consider materializing the per-bucket aggregation first if needed.



WITH buckets_hr_min AS (
SELECT 
	*,
	TO_CHAR(bucket_start, 'YYYY-MM-DD')::DATE AS trade_day,
	TO_CHAR(bucket_start, 'HH24:MI')::TIME AS hour_min 
FROM nq_data.rth_15min_buckets_agg
),
buckets_prev_agg AS (
SELECT 
	*,
	trade_day,
	lag(bucket_delta) OVER (PARTITION BY trade_day ORDER BY trade_day, bucket_start) AS prev_delta_direction,
	lag(bucket_direction) OVER (PARTITION BY trade_day ORDER BY trade_day, bucket_start) AS prev_bucket_direction
FROM buckets_hr_min
),
buckets_prev2_agg AS (
SELECT 
	*,
	CASE WHEN prev_delta_direction > 0 THEN 'positive' ELSE 'negative' END AS prev_delta_direction_indicator
FROM buckets_prev_agg
WHERE prev_delta_direction IS NOT NULL
)
SELECT 
	prev_delta_direction_indicator,
	COUNT(*) FILTER (WHERE bucket_direction = 'up'),
	COUNT(*) AS total,
	ROUND(COUNT(*) FILTER (WHERE bucket_direction = 'up') / COUNT(*)::NUMERIC * 100, 2) AS next_up_pct
FROM buckets_prev2_agg
GROUP BY prev_delta_direction_indicator


It's already after creating a materialized view, which was rather complex...



CREATE MATERIALIZED VIEW rth_15min_buckets_agg AS
WITH ticks_buckets AS (
SELECT 
	*,
	ts_event AT TIME ZONE 'America/New_York' AS time_et,
	DATE_TRUNC('Hour', ts_event AT TIME ZONE 'America/New_York') + (EXTRACT(MINUTE FROM ts_event AT TIME ZONE 'America/New_York')::int / 15 * INTERVAL '15 Minutes') AS bucket_start
	FROM nq_data.ticks t
WHERE side != 'N' AND ts_event AT TIME ZONE 'America/New_York' > '9:30' AND ts_event AT TIME ZONE 'America/New_York' < '16:00'
),
buckets_first_agg AS (
SELECT 
	bucket_start,
	SUM(size) FILTER (WHERE side = 'B') - SUM(size) FILTER (WHERE side = 'A') AS bucket_delta,
	MIN(ts_event) AS bucket_start_time,
	MAX(ts_event) AS bucket_end_time
FROM ticks_buckets
GROUP BY bucket_start
),
buckets_second_agg AS (
SELECT DISTINCT ON (b.bucket_start)
	b.bucket_start,
	bucket_delta,
	t1.price AS bucket_open,
	t2.price AS bucket_close,
	CASE 
		WHEN t2.price = t1.price THEN 'flat'
		WHEN t2.price > t1.price THEN 'up' ELSE 'down'
	END AS bucket_direction
FROM buckets_first_agg b
JOIN ticks_buckets t1 ON b.bucket_start_time = t1.ts_event
JOIN ticks_buckets t2 ON b.bucket_end_time = t2.ts_event
)
SELECT * FROM buckets_second_agg


The findings:

prev_delta_direction_indicator	count	total	next_up_pct
negative	1,026	2,010	51.04
positive	961	1,907	50.39


No useful info from here - it's quite obvious that delta wouldn't matter across all observations.
However, I want to run a specified check for hour_min and prev bucket directions/prev delta checks - maybe that has more odds on specific hour_min windows.

The only thing that requires modification is the final SELECT to check that.




---

## Submission Instructions

Paste your query and results. Log query IDs: RTH-SESS-003, RTH-ORB-001b, RTH-VOL-006.
Do as many as you can — no pressure to finish all three today.
