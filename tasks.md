# NQ Project — Week 25 Day 1

**Generated:** 2026-06-08
**Focus:** First-hour direction bias + gap fill rate + close location in range

---

## Task 1: First Hour Direction Bias

**Scenario:**
Does the direction of the first hour predict the rest of the session? For each trading day, classify the first hour as bullish (fh_close > fh_open) or bearish (fh_close < fh_open). Then, for each direction, show how often the rest of the session (10:30–16:00) also closed in the same direction relative to the first hour close. **(ID: RTH-FH-001)**

Use `nq_data.daily_ohlcv_rth` for the full-day close. For fh_open and fh_close, use `nq_data.ticks` with the aggregate-then-JOIN pattern you built in RTH-RANGE-002 (find MIN/MAX ts_event in the first hour, then point-lookup JOIN for price).

**Output — weekday-level summary broken down by first-hour direction:**
- `fh_direction` — 'bullish' or 'bearish'
- `days` — count of days in this group
- `rest_same_dir_days` — days where the rest-of-session also moved in the same direction (rest close > fh_close for bullish, rest close < fh_close for bearish)
- `same_dir_pct` — rounded to 1

Order by `fh_direction`, `same_dir_pct DESC`.

**Tables:** `nq_data.ticks`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 3/5


I've started by actually creating a view of daily rth ranges of the first hour and rest (ohlc), as it takes very long to execute these queries on this df:

CREATE MATERIALIZED VIEW nq_data.rth_firsthour_rest_ohlc_ranges AS
WITH fh_hl_times AS (
SELECT
    (ts_event AT TIME ZONE 'America/New_York')::date AS trade_date,
    MIN(ts_event) AS fh_open_time,
    MAX(ts_event) AS fh_close_time,
    MIN(price)    AS fh_low,
    MAX(price)    AS fh_high
FROM nq_data.ticks
WHERE (ts_event AT TIME ZONE 'America/New_York')::time >= '09:30'
AND (ts_event AT TIME ZONE 'America/New_York')::time < '10:30'
AND side != 'N'
GROUP BY (ts_event AT TIME ZONE 'America/New_York')::date
),
fh_agg AS (
SELECT DISTINCT ON (fh.trade_date)
	fh.trade_date,
	fh.fh_open_time,
	fh.fh_close_time,
	fh.fh_low,
	fh.fh_high,
	t1.price AS fh_open,
	t2.price AS fh_close
FROM fh_hl_times fh
JOIN nq_data.ticks t1 ON t1.ts_event = fh.fh_open_time
JOIN nq_data.ticks t2 ON t2.ts_event= fh.fh_close_time
),
rest_hl_times AS (
SELECT
    (ts_event AT TIME ZONE 'America/New_York')::date AS trade_date,
    MIN(ts_event) AS r_open_time,
    MAX(ts_event) AS r_close_time,
    MIN(price)    AS r_low,
    MAX(price)    AS r_high
FROM nq_data.ticks
WHERE (ts_event AT TIME ZONE 'America/New_York')::time >= '10:30'
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
FROM rest_hl_times r
JOIN nq_data.ticks t1 ON t1.ts_event = r.r_open_time
JOIN nq_data.ticks t2 ON t2.ts_event= r.r_close_time
)
SELECT 
	f.trade_date,
	f.fh_open,
	f.fh_high,
	f.fh_low,
	f.fh_close,
	r.r_open,
	r.r_high,
	r.r_low,
	r.r_close
FROM r_agg r
JOIN fh_agg f ON r.trade_date = f.trade_date


Then I was able to create the relevant query:


WITH days_directions AS (
SELECT 
	*,
	CASE WHEN fh_close > fh_open THEN 'bullish' ELSE 'bearish' END AS fh_direction,
	CASE WHEN r_close > r_open THEN 'bullish' ELSE 'bearish' END AS r_direction
FROM nq_data.rth_firsthour_rest_ohlc_ranges
),
days_dir_check AS (
SELECT 
	*,
	CASE WHEN fh_direction = r_direction THEN 1 ELSE 0 END AS same_dir
FROM days_directions
)
SELECT 
	fh_direction,
	COUNT(*) AS days,
	SUM(same_dir) AS rest_same_dir_days,
	ROUND(SUM(same_dir) / COUNT(*)::NUMERIC * 100, 1) AS same_dir_pct
FROM days_dir_check
GROUP BY fh_direction


And get the followings findings:


fh_direction	days	rest_same_dir_days	same_dir_pct
bearish	72	30	41.7
bullish	87	49	56.3
---

## Task 2: Gap Fill Rate

**Scenario:**
When NQ gaps up or down at the RTH open (open vs prior RTH close), how often does price trade back through the prior close during the same RTH session? This is the "gap fill" question. **(ID: RTH-GAP-002)**

A gap-up day: `open > prev_close`. Filled if the session `low <= prev_close`.
A gap-down day: `open < prev_close`. Filled if the session `high >= prev_close`.
Flat open (open = prev_close): exclude from analysis.

Using `nq_data.daily_ohlcv_rth`:
- CTE 1: compute `prev_close` with LAG, classify gap direction, compute gap size (`open - prev_close`)
- CTE 2: determine whether each gap was filled

**Output — summary by gap direction:**
- `gap_direction` — 'gap_up' or 'gap_down'
- `days` — total days with a gap in that direction
- `filled_days`
- `fill_pct` — rounded to 1
- `avg_gap_size` — average absolute gap size in points, rounded to 2

Order by `gap_direction`.

**Tables:** `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 4/5


WITH rth_prev_close AS (
SELECT 
	trade_date,
	weekday,
	OPEN,
	high,
	low,
	CLOSE,
	LAG(close) OVER (ORDER BY trade_date) AS prev_close
FROM nq_data.daily_ohlcv_rth
),
rth_gaps AS (
SELECT 
	*,
	CASE WHEN prev_close > OPEN THEN 'gap_down' ELSE 'gap_up' END AS gap_direction
FROM rth_prev_close
WHERE prev_close IS NOT NULL
),
gap_fills AS (
SELECT 
	*,
	abs(OPEN - prev_close) AS gap_size,
	CASE WHEN (OPEN > prev_close AND low <= prev_close) OR (OPEN < prev_close AND high >= prev_close) THEN 1 ELSE 0 END AS gap_filled
FROM rth_gaps
)
SELECT 
	gap_direction,
	COUNT(*) AS days,
	SUM(gap_filled) AS filled_days,
	ROUND(SUM(gap_filled) / COUNT(*)::NUMERIC * 100, 1) AS next_day_fill_pct,
	ROUND(AVG(gap_size), 2) AS avg_gap_size
FROM gap_fills
GROUP BY gap_direction

And the findings:


gap_down	87	57	65.5	151.97
gap_up	98	57	58.2	145.01




---

## Task 3: Close Location in Day's Range

**Scenario:**
Where does the RTH close fall within the day's high-low range? A value of 0% means the day closed at its exact low; 100% means it closed at its exact high. This is called the "close location" or "close percentile." **(ID: RTH-CLOSE-002)**

Formula: `(close - low) / NULLIF(high - low, 0) * 100`

Produce two summaries in one query using UNION ALL or two separate GROUP BY steps inside CTEs:

**Part A — by weekday:**
- `group_type` = 'weekday'
- `group_value` = weekday name
- `days`
- `avg_close_location` — rounded to 1
- `pct_closed_upper_half` — % of days where close location > 50, rounded to 1

**Part B — by gap direction (gap_up / gap_down / flat):**
- `group_type` = 'gap_direction'
- `group_value` = 'gap_up', 'gap_down', or 'flat'
- same metrics as above

Combine both parts with UNION ALL into a single result set ordered by `group_type`, `avg_close_location DESC`.

**Tables:** `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 5/5

---

## Submission Instructions

Paste your query and results for each task. Log query IDs: RTH-FH-001, RTH-GAP-002, RTH-CLOSE-002.
