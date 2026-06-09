# NQ Project — Week 25 Day 2

**Generated:** 2026-06-09
**Focus:** Close location in range + first hour as day's extreme

---

## Task 1: Close Location in Day's Range by Weekday

**Scenario:**
Where does the RTH close fall within the day's high-low range? A value near 0% means the day closed near its low; near 100% means it closed near its high. Aggregate this by weekday to see which days tend to close strong vs weak within their own range. **(ID: RTH-CLOSE-002)**

Formula: `(close - low) / NULLIF(high - low, 0) * 100`

**Output columns:**
- `weekday`
- `days`
- `avg_close_location` — average close location %, rounded to 1
- `pct_closed_upper_half` — % of days where close location > 50, rounded to 1

Order by `avg_close_location DESC`.

**Tables:** `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 3/5

WITH daily_close_locations AS (
SELECT 
	trade_date,
	weekday,
	OPEN,
	high,
	low,
	CLOSE,
	round((CLOSE - low) / (high - low) * 100, 1) AS close_location,
	CASE WHEN round((CLOSE - low) / (high - low) * 100, 1) > 50 THEN 1 ELSE 0 END AS closed_upper_half
FROM nq_data.daily_ohlcv_rth
)
SELECT 
	weekday,
	count(*) AS days,
	sum(closed_upper_half) AS sum_closed_upper_half,
	ROUND(sum(closed_upper_half) / count(*)::NUMERIC * 100, 2) AS pct_closed_upper_half,
	round(AVG(close_location), 2) AS avg_close_locatio
FROM daily_close_locations 
GROUP BY weekday

Findings:

weekday	days	sum_closed_upper_half	pct_closed_upper_half	avg_close_location
Monday	39	26	66.67	62.02
Friday	35	20	57.14	54.84
Thursday	39	21	53.85	52.49
Wednesday	34	21	61.76	58.79
Tuesday	39	22	56.41	57.45

I've added more metrics and changed the rounding because it made much more sense - don't you dare punish me for that!

---

## Task 2: First Hour High/Low as Day's Extreme

**Scenario:**
How often does the first hour (09:30–10:30) set the high or low for the entire RTH day? And how often does the full session stay entirely within the first-hour range — an "inside day" relative to the first hour? **(ID: RTH-FH-002)**

Join `nq_data.rth_firsthour_rest_ohlc_ranges` to `nq_data.daily_ohlcv_rth` on `trade_date`.

**Per-day flags to compute:**
- `fh_set_high` — 1 if `fh_high >= daily_high` (first hour high = day high)
- `fh_set_low` — 1 if `fh_low <= daily_low` (first hour low = day low)
- `inside_fh` — 1 if both conditions true (full day inside first-hour range)

**Output — overall summary (single row):**
- `total_days`
- `fh_set_high_days` + `fh_set_high_pct` — rounded to 1
- `fh_set_low_days` + `fh_set_low_pct` — rounded to 1
- `inside_fh_days` + `inside_fh_pct` — rounded to 1

Then a second query: break `fh_set_high_pct` and `fh_set_low_pct` down by weekday.

**Tables:** `nq_data.rth_firsthour_rest_ohlc_ranges`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 4/5


WITH fh_daily_low_high_agg AS (
SELECT DISTINCT ON (f.trade_date)
	f.trade_date,
	d.weekday,
	f.fh_open,
	f.fh_high,
	f.fh_low,
	f.fh_close,
	d.high AS daily_high,
	d.low AS daily_low,
	CASE WHEN fh_high >= d.high THEN 1 ELSE 0 END AS fh_set_high,
	CASE WHEN fh_low <= d.low THEN 1 ELSE 0 END AS fh_set_low,
	CASE WHEN fh_high >= d.high AND fh_low <= d.low THEN 1 ELSE 0 END AS inside_fh
FROM nq_data.rth_firsthour_rest_ohlc_ranges f
JOIN nq_data.daily_ohlcv_rth d ON f.trade_date = d.trade_date
),
total_agg AS (
SELECT 
	COUNT(*) AS total_days,
	SUM(fh_set_high) AS first_hour_sets_high,
	ROUND(SUM(fh_set_high) / COUNT(*)::NUMERIC * 100, 2) AS fh_sets_high_pct,
	SUM(fh_set_low) AS first_hour_sets_low,
	ROUND(SUM(fh_set_LOW) / COUNT(*)::NUMERIC * 100, 2) AS fh_sets_low_pct,
	SUM(inside_fh) AS inside_first_hour_days,
	ROUND(SUM(inside_fh) / COUNT(*)::NUMERIC * 100, 2) AS inside_fh_pct
FROM fh_daily_low_high_agg
)
SELECT
	weekday,
	COUNT(*) AS total_days,
	SUM(fh_set_high) AS first_hour_sets_high,
	ROUND(SUM(fh_set_high) / COUNT(*)::NUMERIC * 100, 2) AS fh_sets_high_pct,
	SUM(fh_set_low) AS first_hour_sets_low,
	ROUND(SUM(fh_set_LOW) / COUNT(*)::NUMERIC * 100, 2) AS fh_sets_low_pct,
	SUM(inside_fh) AS inside_first_hour_days,
	ROUND(SUM(inside_fh) / COUNT(*)::NUMERIC * 100, 2) AS inside_fh_pct
FROM fh_daily_low_high_agg
GROUP BY weekday


And the findings:

total_days	first_hour_sets_high	fh_sets_high_pct	first_hour_sets_low	fh_sets_low_pct	inside_first_hour_days	inside_fh_pct
159	66	41.51	72	45.28	5	3.14

weekday	total_days	first_hour_sets_high	fh_sets_high_pct	first_hour_sets_low	fh_sets_low_pct	inside_first_hour_days	inside_fh_pct
Thursday	31	17	54.84	11	35.48	2	6.45
Friday	29	11	37.93	15	51.72	1	3.45
Tuesday	33	11	33.33	12	36.36	0	0
Wednesday	33	15	45.45	16	48.48	1	3.03
Monday	33	12	36.36	18	54.55	1	3.03



---

## Submission Instructions

Paste your query and results for each task. Log query IDs: RTH-CLOSE-002, RTH-FH-002.
