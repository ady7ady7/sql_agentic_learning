# NQ Project — Week 28 Day 1

**Generated:** 2026-06-29
**Focus:** Gap magnitude vs fill rate + OR range vs rest-of-session range + overnight gap tendency by weekday

---

## Task 1: Gap Magnitude Buckets vs Fill Rate and Rest-of-Session Direction

**Scenario:**
RTH-GAP-002 established overall gap fill rates: 65.5% for gap-down, 58.2% for gap-up. But today's gap is ~300 pts — roughly 2x the dataset average (~145 pts). Does a larger gap fill less often? Does it produce a more directional or wider rest-of-session? This research directly answers whether gap size is a meaningful moderator of gap behavior. **(ID: RTH-GAP-003)**

Using `nq_data.daily_ohlcv_rth`:

Compute `gap_size = ABS(open - prev_close)` via LAG. Exclude flat opens (gap_size = 0).

Bucket gap_size into thirds using NTILE(3) — label them:
- `small` (Q1 — smallest third)
- `medium` (Q2)
- `large` (Q3 — largest third)

Gap filled = gap-up filled when `low <= prev_close`; gap-down filled when `high >= prev_close`.

**Output:**
- `gap_direction` — gap_up / gap_down
- `gap_bucket` — small / medium / large
- `days`
- `avg_gap_size` — so you know what each bucket actually represents in points
- `fill_pct` — % of days where gap filled
- `rest_bullish_pct` — % of days where RTH close > RTH open

Order by `gap_direction`, `gap_bucket`.

**Finding to answer:** Does fill rate drop as gap size increases? Is a large gap-up more or less likely to produce a bullish close than a small gap-up? Where does today's ~300 pt gap fall in the distribution (which bucket)?

**Tables:** `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 4/5

**(ID: RTH-GAP-003)**



WITH gaps_agg AS (
SELECT 
	*,
	LAG(close) OVER (ORDER BY trade_date) AS prev_close,
	ABS(OPEN - (LAG(close) OVER (ORDER BY trade_date))) AS gap_size
FROM nq_data.daily_ohlcv_rth dor
),
gap_direction_buckets_agg AS (
SELECT 
	*,
	CASE WHEN prev_close > OPEN THEN 'down' ELSE 'up' END AS gap_direction,
	NTILE(3) OVER (ORDER BY gap_size) AS gap_bucket
FROM gaps_agg
WHERE gap_size > 0
),
gap_fill_agg AS (
SELECT 
	*,
	CASE WHEN CLOSE > OPEN THEN 1 ELSE 0 END AS is_bullish,
	CASE 
		WHEN gap_direction = 'up' AND low <= prev_close THEN 1
		WHEN gap_direction = 'down' AND high >= prev_close THEN 1 ELSE 0
	END AS gap_filled
FROM gap_direction_buckets_agg
)
SELECT 
	gap_direction,
	gap_bucket,
	COUNT(*),
	SUM(gap_filled) AS gaps_filled,
	ROUND(AVG(gap_size), 2) AS avg_gap_size,
	ROUND(SUM(gap_filled) / COUNT(*)::NUMERIC * 100, 2) AS fill_pct,
	ROUND(SUM(is_bullish) / COUNT(*)::NUMERIC * 100, 2) AS rest_bullish_pct
FROM gap_fill_agg 
GROUP BY gap_direction, gap_bucket
ORDER BY gap_direction, gap_bucket


gap_direction	gap_bucket	count	gaps_filled	avg_gap_size	fill_pct	rest_bullish_pct
down	1	25	20	30.3	80	52
down	2	35	22	117.94	62.86	65.71
down	3	27	15	308.74	55.56	74.07
up	1	37	32	34.99	86.49	48.65
up	2	26	15	114.75	57.69	50
up	3	34	10	292.14	29.41	38.24

---

## Task 2: OR Range Size vs Rest-of-Session Range

**Scenario:**
The OR (09:30–10:00) range (or_high - or_low) is known at 10:00 — no look-ahead bias. Does a wider OR predict a wider or narrower rest-of-session (10:00–16:00)? Volatility compression vs expansion: does the market front-load its range in the first 30 minutes, or does a wide OR signal that the full session will be volatile too? **(ID: RTH-ORB-009)**

Using `nq_data.or_rest_ohlc_ranges`:

Compute:
- `or_range = or_high - or_low`
- `rest_range = r_high - r_low`
- `or_pct_of_day = or_range / (or_range + rest_range)` — fraction of combined OR+rest range set in the OR window

Bucket `or_range` into quartiles using NTILE(4) — label Q1 (tightest OR) through Q4 (widest OR).

**Output:**
- `or_range_bucket` — Q1 through Q4
- `days`
- `avg_or_range` — avg OR range in pts
- `avg_rest_range` — avg rest-of-session range in pts
- `avg_or_pct_of_day` — avg fraction of combined range set in OR window
- `rest_bullish_pct` — does a wider OR predict direction at all?

Order by `or_range_bucket`.

**Finding to answer:** Does Q4 (widest OR) produce a wider or narrower rest-of-session? Does `avg_or_pct_of_day` increase with OR size (front-loading) or stay flat? Any directional bias?

**Tables:** `nq_data.or_rest_ohlc_ranges`

**Difficulty Rating:** 3/5

**(ID: RTH-ORB-009)**

WITH or_rest_agg AS (
SELECT 
	*,
	ntile(4) OVER (ORDER BY (or_high - or_low)) AS or_range_bucket,
	or_high - or_low AS or_range,
	r_high - r_low AS rest_range,
	ROUND((or_high - or_low) / ((or_high - or_low) + (r_high - r_low)) * 100, 2) AS or_pct_of_day,
	CASE WHEN r_close > r_open THEN 1 ELSE 0 END AS is_bullish
FROM nq_data.or_rest_ohlc_ranges
)
SELECT 
	or_range_bucket,
	COUNT(*) AS days,
	ROUND(AVG(or_range), 2) AS avg_or_range,
	ROUND(AVG(rest_range), 2) AS avg_rest_range,
	ROUND(AVG(or_pct_of_day), 2) AS avg_or_pct_of_day,
	ROUND(SUM(is_bullish) / COUNT(*)::NUMERIC * 100, 2) AS rest_bullish_pct
FROM or_rest_agg
GROUP BY or_range_bucket
ORDER BY or_range_bucket



1	40	86.44	218.91	32.54	57.50
2	40	127.76	279.07	34.87	45.00
3	40	175.30	284.73	40.22	52.50
4	39	251.07	383.36	41.00	66.67




---

## Task 3: Overnight Gap Tendency by Exit Weekday

**Scenario:**
Is it worth holding NQ overnight? The question is: on each weekday, does NQ tend to open above or below the prior RTH close — and by how much? Grouped by the *exit day* (the day you wake up to), this tells you whether holding into Monday, Tuesday, etc. has historically been favorable or not. **(ID: RTH-GAP-004)**

Using `nq_data.daily_ohlcv_rth`:

Compute per row:
- `prev_close` — LAG(close) OVER (ORDER BY trade_date)
- `gap = open - prev_close` (signed — positive = gap up, negative = gap down)
- `gap_direction` — gap_up / gap_down / flat (exclude flat in aggregation)
- `exit_weekday` — weekday of the current row (the day you're opening into)

Then aggregate by `exit_weekday`:
- `days` — total non-flat gap days
- `avg_gap` — signed average gap (positive = tends to gap up into this day)
- `gap_up_pct` — % of days that opened above prior close
- `gap_down_pct` — % of days that opened below prior close
- `avg_gap_up_size` — avg gap size on gap-up days only (abs pts)
- `avg_gap_down_size` — avg gap size on gap-down days only (abs pts)

Order by `gap_up_pct DESC`.

**Finding to answer:** Which weekday tends to open with the most favorable gap for longs? Is there a weekday where holding overnight is consistently punished (large gap-downs)? Does the avg signed gap align with the close-to-close drift from RTH-CLOSE-001 (Monday +114 avg)?

**Tables:** `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 3/5

**(ID: RTH-GAP-004)**


WITH gaps_agg AS (
SELECT 
	*,
	LAG(close) OVER (ORDER BY trade_date) AS prev_close,
	ABS(OPEN - (LAG(close) OVER (ORDER BY trade_date))) AS gap_size
FROM nq_data.daily_ohlcv_rth dor
),
gap_direction_agg AS (
SELECT 
	*,
	weekday AS exit_weekday,
	CASE WHEN prev_close > OPEN THEN 'down' ELSE 'up' END AS gap_direction
FROM gaps_agg
WHERE gap_size > 0
)
SELECT 
	exit_weekday,
	COUNT(*) AS days,
	ROUND(AVG(gap_size), 2) AS avg_gap_size,
	ROUND(COUNT(*) FILTER (WHERE gap_direction = 'up') / COUNT(*)::NUMERIC * 100, 2) AS gap_up_pct,
	ROUND(COUNT(*) FILTER (WHERE gap_direction = 'down') / COUNT(*)::NUMERIC * 100, 2) AS gap_down_pct,
	ROUND(AVG(gap_size) FILTER (WHERE gap_direction = 'up'), 2) AS avg_gap_up_size,
	ROUND(AVG(gap_size) FILTER (WHERE gap_direction = 'down'), 2) AS avg_gap_down_size
FROM gap_direction_agg
GROUP BY exit_weekday


exit_weekday	days	avg_gap_size	gap_up_pct	gap_down_pct	avg_gap_up_size	avg_gap_down_size
Monday	38	197.8	52.63	47.37	222.95	169.86
Friday	35	147.22	57.14	42.86	110.95	195.58
Thursday	38	155.24	44.74	55.26	167.25	145.52
Wednesday	34	114.81	64.71	35.29	138.44	71.48
Tuesday	39	127.19	46.15	53.85	91.33	157.92



---

## Submission Instructions

Paste your queries and results. Log query IDs: RTH-GAP-003, RTH-ORB-009, RTH-GAP-004.
