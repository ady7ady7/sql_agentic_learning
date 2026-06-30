# NQ Project — Week 28 Day 2

**Generated:** 2026-06-30
**Focus:** Gap-up Tuesday FH rejection depth + FH range vs rest-of-session range

---

## Task 1: Gap-Up Tuesday FH High Rejection — How Far Does the Afternoon Drop?

**Scenario:**
RTH-FH-004 showed that on gap-up Tuesdays, the FH sets the day HIGH 55.6% of the time. Today's setup: large gap-up Monday followed by Tuesday. If Tuesday opens with a bullish first hour, the FH high is likely the day's high — but how far does the afternoon typically drop from that peak? This gives actionable fade targets for a gap-up Tuesday with a bullish FH. **(ID: RTH-FH-005)**

Using `nq_data.rth_firsthour_rest_ohlc_ranges` joined to `nq_data.daily_ohlcv_rth`:

Filter to: `weekday = 'Tuesday'` AND gap-up (open > prev_close via LAG on `daily_ohlcv_rth`) AND FH sets day high (`fh_high >= daily_high`).

For each qualifying day compute:
- `fh_high_to_rest_low` — fh_high - r_low (how far from the FH high does the afternoon drop?)
- `fh_high_to_daily_close` — fh_high - close (how far below the FH high does the day close?)
- `reached_fh_open` — did `r_low <= fh_open`? (did the afternoon drop all the way back to where the FH started?)
- `reached_or_open` — did `r_low <= or_open`? (did the afternoon drop back to the RTH open = 09:30 price?)

Then aggregate:
- `days` — total qualifying days
- `avg_fh_high_to_rest_low` — avg drop from FH high to afternoon low
- `avg_fh_high_to_close` — avg drop from FH high to close
- `reached_fh_open_pct` — % that dropped back to FH open
- `reached_or_open_pct` — % that dropped back to OR open (09:30)

For `or_open`: use `or_open` from `nq_data.or_rest_ohlc_ranges` (JOIN on trade_date).

**Tables:** `nq_data.rth_firsthour_rest_ohlc_ranges`, `nq_data.daily_ohlcv_rth`, `nq_data.or_rest_ohlc_ranges`

**Difficulty Rating:** 4/5

**(ID: RTH-FH-005)**


WITH fh_rest_agg AS (
SELECT 
	d.trade_date,
	d.weekday,
	r.fh_open,
	r.fh_close,
	r.fh_high,
	r.fh_low,
	r.r_open,
	r.r_close,
	r.r_high,
	r.r_low,
	LAG(d.close) OVER (ORDER BY d.trade_date) AS prev_close,
	CASE WHEN LAG(d.close) OVER (ORDER BY d.trade_date) < d.OPEN THEN 'up' ELSE 'down' END AS gap_direction
FROM nq_data.rth_firsthour_rest_ohlc_ranges r
JOIN nq_data.daily_ohlcv_rth d ON r.trade_date = d.trade_date 
),
pre_final_agg AS (
SELECT 
	*
FROM fh_rest_agg
WHERE prev_close IS NOT NULL AND weekday = 'Tuesday' AND gap_direction = 'up' AND fh_high > r_high
),
final_agg AS (
SELECT 
	*,
	fh_high - r_low AS fh_high_to_rest_low,
	fh_high - r_close AS fh_high_to_close,
	CASE WHEN r_low <= fh_open THEN 1 ELSE 0 END AS reached_fh_open
FROM pre_final_agg
)
SELECT 
	COUNT(*) AS days,
	ROUND(AVG(fh_high_to_rest_low), 2) AS avg_distance_from_high_to_rest_low,
	ROUND(AVG(fh_high_to_close), 2) AS avg_distance_from_high_to_close,
	ROUND(COUNT(*) FILTER (WHERE reached_fh_open = 1) / COUNT(*)::NUMERIC * 100, 2) AS reached_fh_open_pct
FROM final_agg


or_open and fh_open are the same shit, so your aggregation logic doesn't make sense.


days	avg_distance_from_high_to_rest_low	avg_distance_from_high_to_close	reached_fh_open_pct
10	310.43	235.08	100

---

## Task 2: FH Range Size vs Rest-of-Session Range

**Scenario:**
RTH-ORB-009 showed that a wider OR (09:30–10:00) predicts a wider rest-of-session — no volatility compression. Does the same hold for the first hour (09:30–10:30)? Does a wide FH exhaust the day's move (compression), or does it signal a volatile afternoon (expansion)? The FH is twice as long as the OR — the relationship may differ. **(ID: RTH-FH-006)**

Using `nq_data.rth_firsthour_rest_ohlc_ranges`:

Compute:
- `fh_range = fh_high - fh_low`
- `rest_range = r_high - r_low`
- `fh_pct_of_day = fh_range / (fh_range + rest_range)` — fraction of combined FH+rest range set in the FH window

Bucket `fh_range` into quartiles using NTILE(4) — Q1 (tightest FH) through Q4 (widest FH).

**Output:**
- `fh_range_bucket` — Q1 through Q4
- `days`
- `avg_fh_range` — avg FH range in pts
- `avg_rest_range` — avg rest-of-session range in pts
- `avg_fh_pct_of_day` — avg fraction of combined range set in FH
- `rest_bullish_pct` — % where r_close > r_open

Order by `fh_range_bucket`.

**Finding to answer:** Does Q4 (widest FH) produce a wider or narrower rest-of-session vs Q1? Compare `avg_fh_pct_of_day` across buckets — does the FH front-load a larger share of the day when it's wide? Compare the pattern directly to RTH-ORB-009 (OR range buckets) — is the FH relationship stronger or weaker than the OR relationship?

**Tables:** `nq_data.rth_firsthour_rest_ohlc_ranges`

**Difficulty Rating:** 3/5

**(ID: RTH-FH-006)**





WITH daily_fh_rest_agg AS (
SELECT 
	*,
	ntile(4) OVER (ORDER BY (fh_high - fh_low)) AS fh_range_bucket,
	fh_high - fh_low AS fh_range,
	r_high - r_low AS rest_range,
	ROUND((fh_high - fh_low) / ((fh_high - fh_low) + (r_high - r_low)) * 100, 2) AS fh_pct_of_day
FROM nq_data.rth_firsthour_rest_ohlc_ranges
)
SELECT 
	fh_range_bucket,
	COUNT(*),
	ROUND(AVG(fh_range), 2) AS avg_fg_range,
	ROUND(AVG(rest_range), 2) AS avg_rest_range,
	ROUND(AVG(fh_pct_of_day), 2) AS avg_pct_of_day,
	ROUND(COUNT(*) FILTER (WHERE r_close > r_open) / COUNT(*)::NUMERIC * 100, 2) AS rest_bullish_pct
FROM daily_fh_rest_agg
GROUP BY fh_range_bucket
ORDER BY fh_range_bucket

fh_range_bucket	count	avg_fg_range	avg_rest_range	avg_pct_of_day	rest_bullish_pct
1	40	100.83	199.26	38.38	52.5
2	40	163.97	249.54	43.5	60
3	40	213.2	270.74	46.62	52.5
4	39	330.14	342.29	50.22	64.1


---

## Submission Instructions

Paste your queries and results. Log query IDs: RTH-FH-005, RTH-FH-006.
