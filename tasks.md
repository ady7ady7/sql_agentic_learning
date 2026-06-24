# NQ Project — Week 27 Day 3

**Generated:** 2026-06-24
**Focus:** OR vs FH as rest-of-session predictors — direction bias + delta magnitude

---

## Task 1: OR Direction Bias vs Rest-of-Session Continuation

**Scenario:**
RTH-FH-001 established that bullish first hours continue into the rest of session 56.3% of the time, while bearish first hours reverse 58.3% of the time. The OR (09:30–10:00) is a tighter window than the FH (09:30–10:30). Does OR direction predict the rest-of-session (10:00–16:00) better or worse than FH does?

Define:
- **OR direction**: bullish if OR close > OR open, bearish if OR close < OR open. OR open = first tick price at or after 09:30; OR close = last tick price at or before 10:00.
- **Rest-of-session**: 10:00–16:00 ET. Rest direction = bullish if rest_close > rest_open (rest_open = first price at 10:00; rest_close = last price at or before 16:00).

Use `nq_data.ticks` directly with DISTINCT ON to extract open/close prices for each window.

**Output:**
- `or_direction` — bullish / bearish
- `days`
- `rest_bullish_days`
- `rest_bullish_pct`

Exclude flat OR days (OR open = OR close).

At the bottom of your findings, compare the result directly to RTH-FH-001 numbers. Does the OR signal give a stronger or weaker continuation bias?

**Tables:** `nq_data.ticks`

**Difficulty Rating:** 3/5

**(ID: RTH-ORB-003)**

First, I created a view - it wasn't that fast, as I hadt o make sure everythign is in tact and change some names etc:

CREATE MATERIALIZED VIEW nq_data.or_rest_ohlc_ranges AS
WITH or_hl_times AS (
SELECT
    (ts_event AT TIME ZONE 'America/New_York')::date AS trade_date,
    MIN(ts_event) AS or_open_time,
    MAX(ts_event) AS or_close_time,
    MIN(price)    AS or_low,
    MAX(price)    AS or_high
FROM nq_data.ticks
WHERE (ts_event AT TIME ZONE 'America/New_York')::time >= '09:30'
AND (ts_event AT TIME ZONE 'America/New_York')::time < '10:00'
AND side != 'N'
GROUP BY (ts_event AT TIME ZONE 'America/New_York')::date
),
or_agg AS (
SELECT DISTINCT ON (o.trade_date)
	o.trade_date,
	o.or_open_time,
	o.or_close_time,
	o.or_low,
	o.or_high,
	t1.price AS or_open,
	t2.price AS or_close
FROM or_hl_times o
JOIN nq_data.ticks t1 ON t1.ts_event = o.or_open_time
JOIN nq_data.ticks t2 ON t2.ts_event= o.or_close_time
),
rest_or_times AS (
SELECT
    (ts_event AT TIME ZONE 'America/New_York')::date AS trade_date,
    MIN(ts_event) AS r_open_time,
    MAX(ts_event) AS r_close_time,
    MIN(price)    AS r_low,
    MAX(price)    AS r_high
FROM nq_data.ticks
WHERE (ts_event AT TIME ZONE 'America/New_York')::time >= '10:00'
AND (ts_event AT TIME ZONE 'America/New_York')::time < '16:00'
AND side != 'N'
GROUP BY (ts_event AT TIME ZONE 'America/New_York')::date
),
r_agg AS (
SELECT DISTINCT ON (r.trade_date)
	r.trade_date,
	r.r_open_time,
	r.r_close_time,
	r.r_low,
	r.r_high,
	t1.price AS r_open,
	t2.price AS r_close
FROM rest_or_times r
JOIN nq_data.ticks t1 ON t1.ts_event = r.r_open_time
JOIN nq_data.ticks t2 ON t2.ts_event= r.r_close_time
)
SELECT 
	f.trade_date,
	f.or_open,
	f.or_high,
	f.or_low,
	f.or_close,
	r.r_open,
	r.r_high,
	r.r_low,
	r.r_close
FROM r_agg r
JOIN or_agg f ON r.trade_date = f.trade_date



WITH or_rest_agg AS (
SELECT 
	*,
	CASE 
		WHEN or_close = or_open THEN 'flat'
		WHEN or_close > or_open THEN 'bullish' ELSE 'bearish' 
	END AS or_direction,
	CASE 
		WHEN r_close > r_open THEN 'bullish' ELSE 'bearish'
	END AS rest_direction
FROM nq_data.or_rest_ohlc_ranges
)
SELECT 
	or_direction,
	COUNT(*) AS total,
	COUNT(*) FILTER (WHERE rest_direction = 'bullish') AS rest_bullish_days,
	ROUND(COUNT(*) FILTER (WHERE rest_direction = 'bullish') / COUNT(*)::NUMERIC * 100, 2) AS bullish_days_pct
FROM or_rest_agg
GROUP BY or_direction




or_direction	total	rest_bullish_days	bullish_days_pct
bearish	72	36	50
bullish	87	52	59.77





---

## Task 2: OR Delta Magnitude Buckets vs Rest-of-Session Direction

**Scenario:**
RTH-VOL-004 showed that FH delta direction (positive vs negative) gives a modest ~6pp edge (60.3% vs 54.7%) for rest-of-session direction. Does OR delta *magnitude* matter — i.e., does a larger absolute OR delta translate into a more decisive rest-of-session move?

Using the same OR window (09:30–10:00), calculate:
- `or_delta` — buy volume minus sell volume (side='B' minus side='A') within the OR window
- `abs_or_delta` — ABS(or_delta)
- Bucket `abs_or_delta` into quartiles using NTILE(4) — label them Q1 (smallest) through Q4 (largest)
- `or_delta_direction` — bullish / bearish based on sign of or_delta (exclude flat)

Join to `nq_data.daily_ohlcv_rth` for the RTH close.

For the rest-of-session direction: use rest_close > rest_open logic as in Task 1 (or reuse the same CTE pattern).

**Output:**
- `delta_quartile` — Q1 through Q4
- `or_delta_direction` — bullish / bearish
- `days`
- `rest_bullish_pct`
- `avg_abs_or_delta` — average absolute delta in that quartile (so you know what Q1 vs Q4 actually represents in contracts)

Order by `delta_quartile`, `or_delta_direction`.

**Finding to answer:** Does Q4 (largest OR delta) produce higher rest-of-session continuation than Q1? Does the effect hold for both bullish and bearish delta, or only one side?

**Tables:** `nq_data.ticks`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 4/5

**(ID: RTH-ORB-004)**


Bro, instead I've redone the query from the RTH-VOL-004 TO the OR veariant for now - I think we should honestly start from that first...

WITH ticks_dates_or AS (
SELECT 
	*,
	(ts_event AT TIME ZONE 'America/New_York')::date AS trade_date,
	ts_recv AT TIME ZONE 'America/New_York' AS time_et
FROM nq_data.ticks t
WHERE (ts_event AT TIME ZONE 'America/New_York')::TIME >= '9:30' AND (ts_event AT TIME ZONE 'America/New_York')::TIME <= '10:00'
AND side != 'N'
),
or_vol_agg AS (
SELECT
	 trade_date,
	 SUM(size) FILTER (WHERE side = 'A') AS or_sell_vol,
	 SUM(size) FILTER (WHERE side = 'B') AS or_buy_vol,
	 SUM(size) FILTER (WHERE side = 'B') - SUM(size) FILTER (WHERE side = 'A') AS or_delta_vol
FROM ticks_dates_or
GROUP BY trade_date
),
or_vol_delta_agg AS (
SELECT 
	*,
	CASE WHEN or_delta_vol = 0 THEN 'flat' WHEN or_delta_vol > 0 THEN 'positive' ELSE 'negative' END AS or_delta_direction
FROM or_vol_agg
),
or_rest_agg AS (
SELECT 
	o.trade_date,
	o.or_delta_direction,
	o.or_delta_vol,
	r.or_open,
	r.or_close,
	r.r_open,
	r.r_close,
	r.r_open - r.r_close AS rest_range,
	CASE WHEN r.r_close - r.r_open > 0 THEN 'bullish' ELSE 'bearish' END AS rest_direction
FROM or_vol_delta_agg o
JOIN nq_data.or_rest_ohlc_ranges r ON o.trade_date = r.trade_date
WHERE or_delta_direction != 'flat'
)
SELECT 
	or_delta_direction,
	COUNT(*) AS days,
	ROUND(AVG(or_delta_vol), 2) AS avg_or_delta,
	COUNT(*) FILTER (WHERE rest_direction = 'bullish') AS bullish_day_days,
	ROUND(COUNT(*) FILTER (WHERE rest_direction = 'bullish') / COUNT(*)::NUMERIC * 100, 2) AS bullish_day_pct
FROM or_rest_agg
GROUP BY or_delta_direction


or_delta_direction	days	avg_or_delta	bullish_day_days	bullish_day_pct
negative	82	-1,304.61	42	51.22
positive	77	1,149.34	46	59.74

---

## Task 3: OR vs FH — Side-by-Side Indicator Comparison (5/5)

**Scenario:**
You now have FH → rest continuation data (RTH-FH-001) and will have OR → rest continuation data (RTH-ORB-003). But a cleaner comparison is: on the same set of days, which signal is more aligned with actual rest-of-session direction — the OR direction, the FH direction, or do they agree?

Build a single query that, for each trading day, computes:
- `or_direction` — bullish / bearish (OR close vs OR open, same definition as Task 1)
- `fh_direction` — bullish / bearish (FH close vs FH open; FH = 09:30–10:30; use `nq_data.rth_firsthour_rest_ohlc_ranges` for FH data)
- `rest_direction` — bullish / bearish (from the same materialized view: rest_close vs rest_open)
- `signals_agree` — TRUE if or_direction = fh_direction

Then aggregate:

| Scenario | Days | Rest Bullish % |
|---|---|---|
| OR and FH both bullish | | |
| OR and FH both bearish | | |
| OR bullish, FH bearish | | |
| OR bearish, FH bullish | | |

This tells you: when signals diverge (OR and FH point different directions), which one wins? And when they agree, how decisive is the combined signal?

For OR open/close: use DISTINCT ON from `nq_data.ticks` (same as Task 1).

**Output:** The 4-row aggregation above, ordered by rest_bullish_pct DESC.

**Difficulty Rating:** 5/5

**(ID: RTH-ORB-005)**



WITH directions_agg AS (
SELECT 
	r.trade_date,
	CASE WHEN o.or_close = o.or_open THEN 'flat' WHEN o.or_close > o.or_open THEN 'bullish' ELSE 'bearish' END AS or_direction,
	CASE WHEN r.fh_close = r.fh_open THEN 'flat' WHEN r.fh_close > r.fh_open THEN 'bullish' ELSE 'bearish' END AS fh_direction,
	CASE WHEN r.r_close = r.r_open THEN 'flat' WHEN r.r_close > r.r_open THEN 'bullish' ELSE 'bearish' END AS r_direction
FROM nq_data.rth_firsthour_rest_ohlc_ranges r
JOIN nq_data.or_rest_ohlc_ranges o ON r.trade_date = o.trade_date
),
pre_agg AS (
SELECT 
	*,
	CASE WHEN or_direction = fh_direction THEN TRUE ELSE FALSE END AS signals_agree
FROM directions_agg
)
SELECT 
	or_direction,
	fh_direction,
	COUNT(*) AS days,
	COUNT(*) FILTER (WHERE r_direction = 'bullish') AS bullish_days,
	ROUND(COUNT(*) FILTER (WHERE r_direction = 'bullish') / COUNT(*)::NUMERIC * 100, 2) AS bullish_days_pct
FROM pre_agg
GROUP BY or_direction, fh_direction



or_direction	fh_direction	days	bullish_days	bullish_days_pct
bearish	bearish	62	37	59.68
bullish	bullish	77	45	58.44
bullish	bearish	10	5	50
bearish	bullish	10	4	40


---

## Submission Instructions

Paste your query and results. Log query IDs: RTH-ORB-003, RTH-ORB-004, RTH-ORB-005.
