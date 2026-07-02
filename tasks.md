# NQ Project — Week 28 Day 4

**Generated:** 2026-07-03
**Focus:** Session high/low timing by weekday (RTH-SESS-006) + OR/FH range thresholds by weekday (RTH-SESS-007)

---

## Task 1: Session High/Low Timing by Weekday

**Scenario:**
RTH-SESS-005 showed that 33.3% of day highs and 37.1% of day lows form in the 09:30 window across all days. But Thursday's structural pattern (FH sets HOD 54.8% of the time) and Monday's pattern (FH sets LOD 54.6%) suggest the timing distribution varies significantly by weekday. Does Thursday front-load its HOD in the first 30 minutes more than other days? Does Monday's LOD form later? Knowing the weekday-specific timing gives a better entry map for fading or holding into extremes. **(ID: RTH-SESS-006)**

Using the same approach as RTH-SESS-005 — `nq_data.ticks` joined to `nq_data.daily_ohlcv_rth` via `session_start`/`session_end` — extend the analysis by adding a weekday dimension.

For each `(weekday, 30-min window)` combination, compute:
- `days_high` — count of days where the HOD first formed in that window
- `high_pct` — as % of total days for that weekday
- `days_low` — count of days where the LOD first formed in that window
- `low_pct` — as % of total days for that weekday

**Output:** one row per `(weekday, time_window)`, ordered by weekday (Mon→Fri) then time_window.

**Hint:** Reuse the `hod_set_times` / `lod_set_times` CTE structure from RTH-SESS-005. The weekday is already in `daily_ohlcv_rth.weekday` — join back to it rather than recomputing from ts_event. For within-weekday percentages, use `SUM(COUNT(*)) OVER (PARTITION BY weekday)` as the denominator.

**Finding to answer:** Does Thursday's HOD form earlier (more concentrated in 09:30) than the overall average? Does Monday's LOD form later in the session (consistent with its bullish drift)? Which weekday has the most spread-out timing (least predictable when extreme forms)?

**Tables:** `nq_data.ticks`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 4/5

**(ID: RTH-SESS-006)**


This took a lot of time to adapt to weekday logic tbh XD


WITH hod_set_times AS (
SELECT 
	d.trade_date,
	d.weekday,
	min(t.ts_event) AS high_set_time
FROM nq_data.daily_ohlcv_rth d
JOIN nq_data.ticks t ON d.trade_date = t.ts_event::date AND t.ts_event >= d.session_start AND t.ts_event <= d.session_end
AND t.price = d.high
GROUP BY d.trade_date, d.weekday
),
lod_set_times AS (
SELECT 
	d.trade_date,
	d.weekday ,
	min(t.ts_event) AS low_set_time
FROM nq_data.daily_ohlcv_rth d
JOIN nq_data.ticks t ON d.trade_date = t.ts_event::date AND t.ts_event >= d.session_start AND t.ts_event <= d.session_end
AND t.price = d.LOW
GROUP BY d.trade_date, d.weekday
),
hod_lod_agg AS (
SELECT 
	h.trade_date,
	h.weekday,
	h.high_set_time,
	l.low_set_time,
	TO_CHAR(DATE_TRUNC('hour', h.high_set_time AT TIME ZONE 'America/New_York') + (EXTRACT(MINUTE FROM h.high_set_time AT TIME ZONE 'America/New_York')::int / 30 * INTERVAL '30 Minutes'), 'HH24:MI') AS high_formation_window_et,
	TO_CHAR(DATE_TRUNC('hour', l.low_set_time AT TIME ZONE 'America/New_York') + (EXTRACT(MINUTE FROM l.low_set_time AT TIME ZONE 'America/New_York')::int / 30 * INTERVAL '30 Minutes'), 'HH24:MI') AS low_formation_window_et
FROM hod_set_times h
JOIN lod_set_times l ON h.trade_date = l.trade_date
),
hod_agg_1 AS (
SELECT
	weekday,
	high_formation_window_et,
	COUNT(*) AS high_formed_days,
	SUM(COUNT(*)) OVER (PARTITION BY weekday) AS total_days
FROM hod_lod_agg
GROUP BY high_formation_window_et, weekday
ORDER BY high_formation_window_et, weekday
),
lod_agg_1 AS (
SELECT
	weekday,
	low_formation_window_et,
	COUNT(*) AS low_formed_days,
	SUM(COUNT(*)) OVER (PARTITION BY weekday) AS total_days
FROM hod_lod_agg
GROUP BY low_formation_window_et, weekday
ORDER BY low_formation_window_et, weekday
),
hod_agg AS (
SELECT
	weekday,
	high_formation_window_et,
	high_formed_days,
	ROUND(high_formed_days / total_days::NUMERIC * 100, 2) AS high_formed_pct
FROM hod_agg_1
),
lod_agg AS (
SELECT
	weekday,
	low_formation_window_et,
	low_formed_days,
	ROUND(low_formed_days / total_days::NUMERIC * 100, 2) AS low_formed_pct
FROM lod_agg_1
)
SELECT DISTINCT ON (h.weekday, high_formation_window_et)
	h.weekday,
	h.high_formation_window_et AS time_window,
	h.high_formed_days,
	h.high_formed_pct,
	l.low_formed_days,
	l.low_formed_pct
FROM hod_agg h
JOIN lod_agg l ON h.high_formation_window_et = l.low_formation_window_et
ORDER BY h.weekday, time_window


weekday	time_window	high_formed_days	high_formed_pct	low_formed_days	low_formed_pct
Friday	09:30	10	34.48	9	27.27
Friday	10:00	1	3.45	2	6.06
Friday	11:00	3	10.34	2	6.06
Friday	11:30	1	3.45	2	6.06
Friday	12:00	3	10.34	1	3.45
Friday	12:30	1	3.45	1	3.45
Friday	13:00	1	3.45	1	3.03
Friday	13:30	3	10.34	1	3.03
Friday	14:00	1	3.45	1	3.23
Friday	14:30	1	3.45	2	6.45
Friday	15:00	2	6.9	3	9.09
Friday	15:30	2	6.9	3	9.09
Monday	09:30	9	27.27	17	51.52
Monday	10:00	3	9.09	2	6.06
Monday	11:00	1	3.03	3	9.68
Monday	11:30	1	3.03	5	15.15
Monday	12:00	1	3.03	2	6.06
Monday	12:30	2	6.06	2	6.06
Monday	13:30	2	6.06	3	10.34
Monday	14:30	4	12.12	2	6.06
Monday	15:00	3	9.09	3	9.09
Monday	15:30	7	21.21	3	9.09
Thursday	09:30	14	45.16	12	41.38
Thursday	10:00	3	9.68	2	6.06
Thursday	10:30	3	9.68	2	6.9
Thursday	11:00	3	9.68	2	6.06
Thursday	11:30	2	6.45	2	6.9
Thursday	12:00	1	3.23	1	3.03
Thursday	14:30	2	6.45	2	6.06
Thursday	15:00	1	3.23	3	9.09
Thursday	15:30	2	6.45	4	13.79
Tuesday	09:30	8	24.24	9	27.27
Tuesday	10:00	3	9.09	2	6.06
Tuesday	10:30	1	3.03	2	6.9
Tuesday	11:00	2	6.06	2	6.06
Tuesday	11:30	1	3.03	5	15.15
Tuesday	12:30	2	6.06	2	6.06
Tuesday	13:00	2	6.06	1	3.03
Tuesday	13:30	2	6.06	1	3.03
Tuesday	14:30	2	6.06	2	6.45
Tuesday	15:00	1	3.03	3	9.09
Tuesday	15:30	9	27.27	6	19.35
Wednesday	09:30	12	36.36	14	42.42
Wednesday	10:00	3	9.09	4	12.9
Wednesday	10:30	1	3.03	3	9.68
Wednesday	11:30	1	3.03	2	6.06
Wednesday	12:00	1	3.03	2	6.06
Wednesday	12:30	2	6.06	2	6.06
Wednesday	13:00	1	3.03	1	3.03
Wednesday	14:00	1	3.03	1	3.03
Wednesday	14:30	1	3.03	2	6.06
Wednesday	15:00	6	18.18	3	9.09
Wednesday	15:30	4	12.12	4	13.79


I don't think if these are statistically valid numbers but maybe there are some tendencies

---

## Task 2: Day-over-Day Range Continuity

**Scenario:**
We know wide ORs predict wide afternoons (RTH-ORB-009) and wide FHs predict wide rest-of-sessions (RTH-FH-006) within a single day. But does yesterday's full RTH range predict today's RTH range? If wide days cluster together, you can carry a "high volatility regime" expectation into the next morning and size accordingly. If ranges are independent day-to-day, yesterday's range tells you nothing about today. **(ID: RTH-VOL-010)**

Using `nq_data.daily_ohlcv_rth`, for each trading day compute:
- `prev_range` — prior day's RTH range via LAG(high - low) OVER (ORDER BY trade_date)
- Bucket `prev_range` into thirds using NTILE(3): `small` (Q1), `medium` (Q2), `large` (Q3)

Then aggregate by `prev_range_bucket`:
- `days`
- `avg_prev_range` — average of yesterday's range in that bucket
- `avg_today_range` — average of today's RTH range (high - low) for days following a day in that bucket
- `pct_today_wide` — % of today's days where today's range is in the top third of all ranges (use a subquery or CTE to get the global Q3 threshold, then FILTER)

**Finding to answer:** Do wide days cluster? If yesterday was a large-range day, is today's expected range meaningfully higher than if yesterday was a small-range day? Is there autocorrelation in NQ daily ranges, or are they effectively independent?

**Tables:** `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 4/5

**(ID: RTH-VOL-010)**


WITH days_rth_ranges AS (
SELECT 
	*,
	high - low AS daily_range,
	LAG((high - low)) OVER (ORDER BY trade_date) AS prev_range
FROM nq_data.daily_ohlcv_rth
),
prev_range_agg AS (
SELECT 
	*,
	ntile(3) OVER (ORDER BY daily_range) AS daily_range_bucket,
	ntile(3) OVER (ORDER BY prev_range) AS prev_range_bucket
FROM days_rth_ranges
WHERE prev_range IS NOT NULL
)
SELECT 
	prev_range_bucket,
	COUNT(*) AS days,
	round(AVG(prev_range), 2) AS avg_prev_range,
	ROUND(AVG(daily_range), 2) AS avg_daily_range,
	round(COUNT(*) FILTER (WHERE daily_range >= 380) / COUNT(*)::numeric * 100, 2) AS pct_today_wide
FROM prev_range_agg
GROUP BY prev_range_bucket


prev_range_bucket	days	avg_prev_range	avg_daily_range	pct_today_wide
3	61	521.8	423.7	55.74
2	62	306.01	323.88	27.42
1	62	184.67	265.85	16.13

---

## Submission Instructions

Paste your queries and results. Log query IDs: RTH-SESS-006, RTH-VOL-010.
