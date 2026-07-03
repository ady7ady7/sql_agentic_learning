# NQ Project — Week 28 Day 5

**Generated:** 2026-07-04
**Focus:** Cumulative intraday range expansion curve (RTH-VOL-011) + volatility regime × direction interaction (RTH-VOL-012)

---

## Task 1: Cumulative Intraday Range Expansion Curve

**Scenario:**
We know the first hour captures 58–67% of the full day's range (RTH-RANGE-002), but that's a single snapshot at 10:30. What does the full intraday range expansion curve look like across all 13 RTH 30-min windows? At 10:00, how much of the day's range is already locked in? At 13:00? This gives a real-time "how much range is left to expect" estimate at any point in the session — directly useful for target-setting and position sizing mid-day. **(ID: RTH-VOL-011)**

Using `nq_data.rth_15min_buckets_agg` (the materialized view already built for RTH-VOL-006/007/008 — per-bucket OHLC for all RTH 15-min windows) joined to `nq_data.daily_ohlcv_rth`:

For each 30-min window boundary (10:00, 10:30, 11:00, ... 16:00), compute the **running high and low** established across all buckets up to and including that window, then express it as a % of the full day's range.

Approach:
- Group 15-min buckets into 30-min windows: floor to 30-min boundary using the same pattern as RTH-SESS-005
- For each trade_date and each 30-min window, compute `running_high = MAX(bucket_high)` and `running_low = MIN(bucket_low)` across all buckets from session open up to that window (cumulative, not per-window)
- Join to `daily_ohlcv_rth` to get the full day's range (`high - low`)
- Compute `range_captured_pct = (running_high - running_low) / NULLIF(high - low, 0) * 100`
- Average across all trading days per window

**Output:**
- `time_window` — 10:00, 10:30, 11:00, ... 16:00 (13 rows)
- `avg_range_captured_pct` — average % of full day's range already established by this window
- `median_range_captured_pct` — PERCENTILE_CONT(0.5) for robustness against outlier days

Order by `time_window`.

**Hint:** Use a window function — `MAX(bucket_high) OVER (PARTITION BY trade_date ORDER BY bucket_start ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` to get the running high up to each bucket. Then aggregate to 30-min windows after.

**Finding to answer:** How quickly does the day's range expand? Is the curve steep early (most range locked in by 11:00) and flat midday, or does it expand gradually throughout? At what time is 50% of the day's range typically established? 75%?

**Tables:** `nq_data.rth_15min_buckets_agg`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 5/5

**(ID: RTH-VOL-011)**

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
	current_window_et AS time_window,
	ROUND(AVG(daily_range_pct), 2) AS avg_range_captured_pct,
	PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY daily_range_pct)
FROM running_agg
GROUP BY current_window_et


time_window	avg_range_captured_pct	percentile_cont
09:30	51.34	49.42
10:00	63.39	62.665
10:30	71.58	71.88
11:00	76.69	78.355
11:30	81.06	84.065
12:00	83.93	87.4
12:30	86.64	93.545
13:00	89.02	94.945
13:30	91.85	100
14:00	93.2	100
14:30	95.15	100
15:00	97.27	100
15:30	100	100
16:00	102	100


---

## Task 2: Volatility Regime × Direction Interaction

**Scenario:**
RTH-VOL-010 showed that wide days cluster — after a large day, tomorrow is likely wide too. But does the direction of yesterday's move matter? After a large bearish day, does the market mean-revert (today bullish) or continue (today bearish)? After a large bullish day, same question. This would turn the volatility clustering signal into a full directional + sizing framework: not just "expect a wide day" but "expect a wide day in which direction." **(ID: RTH-VOL-012)**

Using `nq_data.daily_ohlcv_rth`, extend RTH-VOL-010 by adding prior day direction:

For each trading day compute:
- `prev_range` — LAG(high - low)
- `prev_direction` — LAG(close > open) → 'bullish' / 'bearish' (exclude flat: close = open)
- `prev_range_bucket` — NTILE(3) OVER (ORDER BY prev_range): 'small', 'medium', 'large'
- `today_bullish` — close > open

Aggregate by `(prev_range_bucket, prev_direction)`:
- `days`
- `avg_prev_range`
- `avg_today_range` — high - low
- `today_bullish_pct` — % of days where today closed bullish

Focus the finding on the `large` bucket split by prior direction — that's where the actionable signal lives.

**Output:** one row per `(prev_range_bucket, prev_direction)`, ordered by prev_range_bucket DESC, prev_direction.

**Finding to answer:** After a large bullish day, does today lean bullish (continuation) or bearish (mean reversion)? After a large bearish day, which way? Is the directional signal stronger after large days than after small days? Does prior direction add any information beyond prior range size alone?

**Tables:** `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 4/5

**(ID: RTH-VOL-012)**



WITH days_rth_ranges AS (
SELECT 
	*,
	high - low AS daily_range,
	LAG((high - low)) OVER (ORDER BY trade_date) AS prev_range,
	LAG(close) OVER (ORDER BY trade_date) AS prev_close,
	LAG(open) OVER (ORDER BY trade_date) AS prev_open
FROM nq_data.daily_ohlcv_rth
),
prev_range_agg AS (
SELECT 
	*,
	CASE WHEN prev_close > prev_open THEN 'bullish' ELSE 'bearish' END AS prev_direction,
	CASE WHEN CLOSE > OPEN THEN 1 ELSE 0 END AS today_bullish,
	ntile(3) OVER (ORDER BY daily_range) AS daily_range_bucket,
	ntile(3) OVER (ORDER BY prev_range) AS prev_range_bucket
FROM days_rth_ranges
WHERE prev_range IS NOT NULL
)
SELECT 
	prev_range_bucket,
	prev_direction,
	COUNT(*) AS days,
	ROUND(AVG(daily_range), 2) AS avg_daily_range,
	ROUND(SUM(today_bullish) / COUNT(*)::NUMERIC * 100, 2) AS today_bullish_pct
FROM prev_range_agg
GROUP BY prev_range_bucket, prev_direction
ORDER BY prev_range_bucket


prev_range_bucket	prev_direction	days	avg_daily_range	today_bullish_pct
1	bearish	27	302.43	37.04
1	bullish	35	237.63	60
2	bearish	32	338.34	43.75
2	bullish	30	308.44	63.33
3	bearish	26	423.99	50
3	bullish	35	423.49	65.71



---

## Submission Instructions

Paste your queries and results. Log query IDs: RTH-VOL-011, RTH-VOL-012.
