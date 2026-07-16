# NQ Project — Week 30 Day 4

**Generated:** 2026-07-16
**Focus:** EU high resistance weekday × gap breakdown on bearish days (RTH-GLOB-007b) + delta surge at key EU levels vs mid-range (RTH-VOL-016)

---

## Task 1: EU High Resistance — Weekday × Gap Direction on Bearish Days

**Scenario:**
RTH-GLOB-007 showed gap direction explains 26.5pp of EU high hold rate (gap-down bearish: 74% hold, gap-up bearish: 48% hold). RTH-GLOB-006b showed weekday also matters (Wed/Thu ~75% hold, Monday ~31%). Now combine both: which weekday × gap combination is the cleanest EU short entry, and which is the worst? Gap-down Tuesday bearish specifically — is EU high a near-guaranteed resistance level on that setup? **(ID: RTH-GLOB-007b)**

**Architecture:**
RTH-GLOB-007 query already exists. Add `weekday` to the SELECT and GROUP BY in the final aggregation alongside `gap_direction`. Sample sizes will be small (4–8 days per cell) — report all cells but flag those with < 5 days. Focus on directional patterns not statistical certainty.

Report per weekday × gap_direction:
- `days`
- `avg_rth_open_vs_eu_high`
- `pct_exceeded_eu_high`
- `pct_undercut_eu_low`

**Finding to answer:** Which specific weekday × gap combinations make EU high a reliable short entry (< 25% exceeded) vs a trap (> 60% exceeded)? Is gap-down Tuesday bearish the cleanest EU short setup? Is gap-up Monday bearish always a "wait for RTH" situation?

**Tables:** `nq_data.ticks`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 2/5

**(ID: RTH-GLOB-007b)**

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
	weekday,
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
GROUP BY weekday, gap_direction


weekday	gap_direction	days	avg_eu_range	avg_rth_open_vs_eu_high	avg_eu_range	avg_rth_open_location	pct_undercut_eu_low	pct_went_above_eu_high
Friday	gap_down	4	328.75	-237.38	100	31.39	100	25
Friday	gap_up	8	141.44	-54.06	100	59.93	87.5	75
Monday	gap_down	4	216.19	-138.19	100	46.8	100	50
Monday	gap_up	9	256.42	-55.28	88.89	77.02	66.67	77.78
Thursday	gap_down	11	195.98	-141.95	100	30.68	100	9.09
Thursday	gap_up	8	195.59	-32.22	87.5	78.64	62.5	50
Tuesday	gap_down	7	205.79	-120.68	100	40.67	71.43	57.14
Tuesday	gap_up	7	94.79	-52.68	100	42.24	100	28.57
Wednesday	gap_down	5	187	-129.85	100	36.76	100	0
Wednesday	gap_up	10	152.43	-52.7	100	60.06	100	30

---

## Task 2: OR Delta × FH Delta Combined Signal → Rest-of-Session Direction

**Scenario:**
RTH-VOL-004 established: positive FH delta (09:30–10:30) → rest bullish 60.3% vs negative → 54.7%. RTH-ORB-004 established: positive OR delta (09:30–10:00) → rest bullish 59.74% vs negative → 51.22%. Both were measured independently. The question: when both agree (OR and FH delta both positive, or both negative), does the edge stack beyond either standalone signal? When they diverge (OR positive but FH net negative, or vice versa), does it cancel to ~50%? This is the delta equivalent of RTH-ORB-005 (price direction version: agree-bearish → 59.7% rest bullish). **(ID: RTH-VOL-019)**

**Architecture:**
1. CTE 1 — from `rth_15min_buckets_agg`, compute per trade_date:
   - `or_delta` = SUM(bucket_delta) WHERE TO_CHAR(bucket_start, 'HH24:MI') < '10:00' (09:30 + 09:45 buckets)
   - `fh_delta` = SUM(bucket_delta) WHERE TO_CHAR(bucket_start, 'HH24:MI') < '10:30' (09:30 + 09:45 + 10:00 + 10:15 buckets)
2. CTE 2 — classify:
   - `or_delta_dir` = CASE WHEN or_delta > 0 THEN 'positive' ELSE 'negative' END
   - `fh_delta_dir` = CASE WHEN fh_delta > 0 THEN 'positive' ELSE 'negative' END
   - `combined_signal` = CASE: 'agree_bullish' (both positive), 'agree_bearish' (both negative), 'or_bull_fh_bear', 'or_bear_fh_bull'
3. CTE 3 — JOIN to `rth_firsthour_rest_ohlc_ranges` on trade_date. `rest_bullish` = CASE WHEN r_close > r_open THEN 1 ELSE 0 END
4. Final SELECT — GROUP BY `combined_signal`:
   - `days`
   - `rest_bullish_pct`
   - `avg_or_delta`
   - `avg_fh_delta`

**Finding to answer:** Does agree_bullish (both OR + FH delta positive) beat the standalone signals (>60.3%)? Does agree_bearish produce a bearish lean below 50%? Do divergence cases collapse to ~50%? Is delta a stronger stacking signal than price direction (RTH-ORB-005: agree-bearish 59.7%, agree-bullish 58.4%)?

**Tables:** `nq_data.rth_15min_buckets_agg`, `nq_data.rth_firsthour_rest_ohlc_ranges`

**Difficulty Rating:** 3/5

**(ID: RTH-VOL-019)**


WITH or_delta_agg AS (
SELECT 
	bucket_start::date AS trade_date,
	SUM(bucket_delta) AS daily_or_delta
FROM nq_data.rth_15min_buckets_agg
WHERE bucket_start::time >= '9:30' AND bucket_start::time < '10:00'
GROUP BY bucket_start::DATE
),
fh_delta_agg AS (
SELECT 
	bucket_start::date AS trade_date,
	SUM(bucket_delta) AS daily_fh_delta
FROM nq_data.rth_15min_buckets_agg
WHERE bucket_start::time >= '9:30' AND bucket_start::time < '10:45'
GROUP BY bucket_start::DATE
),
fh_or_combined_signal_agg AS (
SELECT 
	f.trade_date,
	f.daily_fh_delta,
	o.daily_or_delta,
	CASE WHEN f.daily_fh_delta > 0 THEN 'positive' ELSE 'negative' END AS fh_delta_direction,
	CASE WHEN o.daily_or_delta > 0 THEN 'positive' ELSE 'negative' END AS or_delta_direction,
	CASE 
		WHEN f.daily_fh_delta > 0 AND o.daily_or_delta > 0 THEN 'agree_bullish' 
		WHEN f.daily_fh_delta < 0 AND o.daily_or_delta < 0 THEN 'agree_bearish' 
		WHEN f.daily_fh_delta > 0 AND o.daily_or_delta < 0 THEN 'or_bear_fh_bull' 
		WHEN f.daily_fh_delta < 0 AND o.daily_or_delta > 0 THEN 'or_bull_fh_bear' 
	END AS combined_signal
FROM fh_delta_agg f
JOIN or_delta_agg o ON f.trade_date = o.trade_date
),
pre_final_agg AS (
SELECT 
	*,
	CASE WHEN r.r_close > r.r_open THEN 1 ELSE 0 END AS rest_bullish
FROM fh_or_combined_signal_agg f
JOIN nq_data.rth_firsthour_rest_ohlc_ranges r ON f.trade_date = r.trade_date
)
SELECT 
	combined_signal,
	COUNT(*) AS days,
	ROUND(AVG(daily_fh_delta), 2) AS avg_fh_delta,
	ROUND(AVG(daily_or_delta), 2) AS avg_or_delta,
	ROUND(SUM(rest_bullish) / COUNT(*)::NUMERIC * 100, 2) AS rest_bullish_pct
FROM pre_final_agg
GROUP BY combined_signal


combined_signal	days	avg_fh_delta	avg_or_delta	rest_bullish_pct
agree_bearish	69	-1,878.67	-1,407.28	57.97
agree_bullish	61	1,822.25	1,293.08	60.66
or_bear_fh_bull	13	866.54	-759.69	46.15
or_bull_fh_bear	16	-942.69	601.31	50

I've also done an extra aggregation by weekday :)) - make sure to chceck it and add/include it in the commit and findings with a proper tag as Task 3 before we move on, tag it as RTH-VOL-019b and include in nq_findings



WITH or_delta_agg AS (
SELECT 
	bucket_start::date AS trade_date,
	SUM(bucket_delta) AS daily_or_delta
FROM nq_data.rth_15min_buckets_agg
WHERE bucket_start::time >= '9:30' AND bucket_start::time < '10:00'
GROUP BY bucket_start::DATE
),
fh_delta_agg AS (
SELECT 
	bucket_start::date AS trade_date,
	SUM(bucket_delta) AS daily_fh_delta
FROM nq_data.rth_15min_buckets_agg
WHERE bucket_start::time >= '9:30' AND bucket_start::time < '10:45'
GROUP BY bucket_start::DATE
),
fh_or_combined_signal_agg AS (
SELECT 
	f.trade_date,
	f.daily_fh_delta,
	o.daily_or_delta,
	CASE WHEN f.daily_fh_delta > 0 THEN 'positive' ELSE 'negative' END AS fh_delta_direction,
	CASE WHEN o.daily_or_delta > 0 THEN 'positive' ELSE 'negative' END AS or_delta_direction,
	CASE 
		WHEN f.daily_fh_delta > 0 AND o.daily_or_delta > 0 THEN 'agree_bullish' 
		WHEN f.daily_fh_delta < 0 AND o.daily_or_delta < 0 THEN 'agree_bearish' 
		WHEN f.daily_fh_delta > 0 AND o.daily_or_delta < 0 THEN 'or_bear_fh_bull' 
		WHEN f.daily_fh_delta < 0 AND o.daily_or_delta > 0 THEN 'or_bull_fh_bear' 
	END AS combined_signal
FROM fh_delta_agg f
JOIN or_delta_agg o ON f.trade_date = o.trade_date
),
pre_final_agg AS (
SELECT 
	*,
	TRIM(TO_CHAR(r.trade_date, 'Day')) AS weekday,
	CASE WHEN r.r_close > r.r_open THEN 1 ELSE 0 END AS rest_bullish
FROM fh_or_combined_signal_agg f
JOIN nq_data.rth_firsthour_rest_ohlc_ranges r ON f.trade_date = r.trade_date
)
SELECT 
	weekday,
	combined_signal,
	COUNT(*) AS days,
	ROUND(AVG(daily_fh_delta), 2) AS avg_fh_delta,
	ROUND(AVG(daily_or_delta), 2) AS avg_or_delta,
	ROUND(SUM(rest_bullish) / COUNT(*)::NUMERIC * 100, 2) AS rest_bullish_pct
FROM pre_final_agg
GROUP BY combined_signal, weekday


weekday	combined_signal	days	avg_fh_delta	avg_or_delta	rest_bullish_pct
Friday	agree_bearish	14	-1,670.36	-1,157.07	42.86
Monday	agree_bearish	13	-1,940.54	-1,211.08	46.15
Thursday	agree_bearish	14	-2,511.64	-2,064.64	64.29
Tuesday	agree_bearish	16	-1,490.25	-1,210	75
Wednesday	agree_bearish	12	-1,834.08	-1,407.83	58.33
Friday	agree_bullish	10	2,002	1,437.9	50
Monday	agree_bullish	16	1,745.56	1,170.69	68.75
Thursday	agree_bullish	12	1,509.92	1,164.17	58.33
Tuesday	agree_bullish	10	1,709.6	1,347.2	60
Wednesday	agree_bullish	13	2,153.31	1,409.69	61.54
Thursday	or_bear_fh_bull	5	563.8	-797.2	20
Tuesday	or_bear_fh_bull	3	1,614.33	-489.33	66.67
Wednesday	or_bear_fh_bull	5	720.6	-884.4	60
Friday	or_bull_fh_bear	5	-958.6	893.2	60
Monday	or_bull_fh_bear	4	-659.75	443	75
Tuesday	or_bull_fh_bear	4	-1,730	562.5	50
Wednesday	or_bull_fh_bear	3	-243.67	377.67	0

---

## Submission Instructions

Paste your queries and results. Log query IDs: RTH-GLOB-007b, RTH-VOL-019.
