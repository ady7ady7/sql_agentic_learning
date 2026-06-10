# NQ Project — Week 25 Day 3

**Generated:** 2026-06-10
**Focus:** First hour direction by weekday + gap × first-hour interaction

---

## Task 1: First Hour Direction by Weekday

**Scenario:**
We know the overall first-hour direction bias (RTH-FH-001). Now break it down by weekday: which days have the most bullish first hours? Which the most bearish? **(ID: RTH-FH-003)**

Using `nq_data.rth_firsthour_rest_ohlc_ranges`, classify each day's first hour as bullish (fh_close > fh_open) or bearish (fh_close < fh_open). Exclude flat days (fh_close = fh_open).

Join to `nq_data.daily_ohlcv_rth` on `trade_date` to get `weekday`.

**Output — grouped by weekday:**
- `weekday`
- `total_days`
- `bullish_fh_days`
- `bullish_fh_pct` — rounded to 1
- `avg_fh_range` — average `fh_high - fh_low` across all days for that weekday, rounded to 2

Order by `bullish_fh_pct DESC`.

**Tables:** `nq_data.rth_firsthour_rest_ohlc_ranges`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 3/5


WITH dates_fh_direction AS (
SELECT
	rf.trade_date,
	dor.weekday,
	fh_high,
	fh_low,
	CASE WHEN fh_open < fh_close THEN 'bullish' ELSE 'bearish' END AS fh_direction
FROM nq_data.rth_firsthour_rest_ohlc_ranges rf
JOIN nq_data.daily_ohlcv_rth dor ON rf.trade_date = dor.trade_date
WHERE fh_open != fh_close
)
SELECT 
	weekday,
	count(*) AS total_days,
	count(*) FILTER (WHERE fh_direction = 'bullish') AS bullish_fh_days,
	ROUND(count(*) FILTER (WHERE fh_direction = 'bullish') / COUNT(*) ::NUMERIC * 100, 2) AS bullish_fh_days_pct,
	count(*) FILTER (WHERE fh_direction = 'bearish') AS bearish_fh_days,
	ROUND(count(*) FILTER (WHERE fh_direction = 'bearish') / COUNT(*) ::NUMERIC * 100, 2) AS bearish_fh_days_pct,
	ROUND(AVG(fh_high - fh_low), 2) AS avg_fh_range
FROM dates_fh_direction
GROUP BY weekday
ORDER BY bullish_fh_days_pct DESC


Interesting, findings:

weekday	total_days	bullish_fh_days	bullish_fh_days_pct	bearish_fh_days	bearish_fh_days_pct	avg_fh_range
Monday	39	25	64.1	14	35.9	189.72
Tuesday	39	24	61.54	15	38.46	189.72
Friday	35	21	60	14	40	215.59
Wednesday	34	18	52.94	16	47.06	189.69
Thursday	39	16	41.03	23	58.97	213.84



---

## Task 2: Gap Direction × First Hour Direction Interaction

**Scenario:**
When the market gaps up AND the first hour is also bullish — does the rest of session continue? What about gap up + bearish first hour (fade)? This cross-analysis combines overnight context with intraday opening behavior. **(ID: RTH-SESS-001)**

Build a single query that joins `nq_data.daily_ohlcv_rth` with `nq_data.rth_firsthour_rest_ohlc_ranges` and computes four buckets: the combination of gap direction (gap_up / gap_down — exclude flat opens where `open = prev_close`) and first-hour direction (bullish / bearish — exclude flat FH where `fh_close = fh_open`).

**Output — one row per bucket:**
- `gap_direction` — 'gap_up' or 'gap_down'
- `fh_direction` — 'bullish' or 'bearish'
- `days`
- `avg_full_day_range` — rounded to 2
- `avg_close_location` — `(close - low) / NULLIF(high - low, 0) * 100`, rounded to 1
- `rest_continues_pct` — % of days where rest-of-session direction matches fh_direction (r_close > fh_close for bullish, r_close < fh_close for bearish), rounded to 1

Order by `gap_direction`, `fh_direction`.

**Tables:** `nq_data.daily_ohlcv_rth`, `nq_data.rth_firsthour_rest_ohlc_ranges`

**Difficulty Rating:** 5/5


WITH first_daily_range_agg AS (
SELECT 
	rf.trade_date,
	dor.weekday,
	dor.high AS daily_high,
	dor.low AS daily_low,
	LAG(dor.CLOSE) OVER (ORDER BY rf.trade_date) AS pd_close,
	fh_open,
	fh_high,
	fh_low,
	fh_close,
	r_open,
	r_close,
	CASE WHEN fh_open < fh_close THEN 'bullish' ELSE 'bearish' END AS fh_direction
FROM nq_data.rth_firsthour_rest_ohlc_ranges rf
JOIN nq_data.daily_ohlcv_rth dor ON rf.trade_date = dor.trade_date
WHERE fh_open != fh_close
),
gap_agg AS (
SELECT 
	*,
	CASE 
		WHEN fh_open > pd_close THEN 'gap_up'
		WHEN fh_open < pd_close THEN 'gap_down' ELSE 'no_gap'
	END AS gap_direction
FROM first_daily_range_agg
WHERE pd_close IS NOT NULL
),
gap_rest_continues_agg AS (
SELECT 
	*,
	CASE WHEN (fh_direction = 'bearish' AND r_close < fh_close) OR (fh_direction = 'bullish' AND r_close  > fh_close) THEN 1 ELSE 0 END AS rest_continues
FROM gap_agg
)
SELECT 
	gap_direction,
	fh_direction,
	COUNT(*) AS days,
	ROUND(AVG(daily_high - daily_low), 2) AS avg_full_day_range,
	ROUND(AVG((r_close - daily_low) / (daily_high - daily_low) * 100), 2) AS avg_close_location,
	ROUND(sum(rest_continues) / COUNT(*)::NUMERIC * 100, 2) AS rest_continues_pct
FROM gap_rest_continues_agg
WHERE gap_direction != 'no_gap'
GROUP BY gap_direction, fh_direction
ORDER BY gap_direction, fh_direction

As for the findings:

gap_direction	fh_direction	days	avg_full_day_range	avg_close_location	rest_continues_pct
gap_down	bearish	36	383.03	55.5	33.33
gap_down	bullish	51	352.34	67.11	70.59
gap_up	bearish	46	297.61	44.96	47.83
gap_up	bullish	51	331.91	58.58	45.1


---

## Submission Instructions

Paste your query and results for each task. Log query IDs: RTH-FH-003, RTH-SESS-001.
