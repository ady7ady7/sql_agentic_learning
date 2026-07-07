# NQ Project — Week 29 Day 2

**Generated:** 2026-07-08
**Focus:** Overnight direction vs RTH direction — aggregate + weekday breakdown (RTH-GLOB-001)

---

## Task 1: Overnight Direction vs RTH Session Direction

**Scenario:**
RTH-GAP-003 showed that large gap-downs produce 74% bullish RTH closes — but that's bucketed by magnitude, not direction alone. The cleaner question is: does the overnight direction (gap-up vs gap-down, regardless of size) predict the RTH close direction? And does that relationship change by weekday? We already know Tuesday gap-down + bearish FH = 100% rest bullish, but that stacks multiple filters. This is the pure overnight-only signal: knowing only whether tonight's session closed above or below yesterday's RTH close, what does that tell you about today's RTH close direction? **(ID: RTH-GLOB-001)**

Using `nq_data.daily_ohlcv_rth`, for each trading day compute:
- `prev_close` — LAG(close) OVER (ORDER BY trade_date)
- `gap_direction` — CASE WHEN open > prev_close THEN 'gap_up' WHEN open < prev_close THEN 'gap_down' ELSE 'flat' END
- `gap_size` — ABS(open - prev_close)
- `rth_bullish` — close > open (RTH session direction, open→close)
- `rth_close_vs_prev` — close > prev_close (full overnight+RTH net direction)

**Cut 1 — Aggregate (all weekdays):**
Aggregate by `gap_direction`:
- `days`
- `avg_gap_size`
- `rth_bullish_pct` — % where RTH close > RTH open
- `rth_close_above_prev_pct` — % where RTH close > prior RTH close (net positive from yesterday's close)

**Cut 2 — By weekday:**
Same aggregation, grouped by `(weekday, gap_direction)`. Exclude flat opens. Order by weekday, gap_direction.

**Finding to answer:** Does gap direction predict RTH close direction at all? Is the signal stronger on gap-downs (mean reversion) or gap-ups (continuation)? Which weekday shows the strongest overnight→RTH directional relationship? Does the net close vs prior close (rth_close_above_prev_pct) tell a different story than the intraday open→close direction?

**Tables:** `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 3/5

**(ID: RTH-GLOB-001)**

Cut 1:

WITH rth_ohlcv_prev AS (
SELECT 
	*,
	lag(close) OVER (ORDER BY trade_date) AS prev_close
FROM nq_data.daily_ohlcv_rth
),
prefinal_agg AS (
SELECT
	*,
	CASE WHEN OPEN > prev_close THEN 'gap_up' ELSE 'gap_down' END AS gap_direction,
	ABS(open - prev_close) AS gap_size,
	CASE WHEN CLOSE > OPEN THEN 1 ELSE 0 END AS rth_bullish,
	CASE WHEN CLOSE > prev_close THEN 1 ELSE 0 END AS rth_close_higher_than_pd
FROM rth_ohlcv_prev
WHERE prev_close IS NOT NULL
)
SELECT 
	gap_direction,
	ROUND(AVG(gap_size), 2) AS avg_gap_size,
	round(SUM(rth_bullish) / COUNT(*)::NUMERIC * 100, 2) AS rth_bullish_pct,
	round(SUM(rth_close_higher_than_pd) / COUNT(*)::NUMERIC * 100, 2) AS rth_close_above_prev_pct
FROM prefinal_agg
GROUP BY gap_direction


gap_direction	avg_gap_size	rth_bullish_pct	rth_close_above_prev_pct
gap_down	150.24	63.64	37.5
gap_up	146.51	45.36	64.95



Cut 2:

WITH rth_ohlcv_prev AS (
SELECT 
	*,
	lag(close) OVER (ORDER BY trade_date) AS prev_close
FROM nq_data.daily_ohlcv_rth
),
prefinal_agg AS (
SELECT
	*,
	CASE WHEN OPEN > prev_close THEN 'gap_up' ELSE 'gap_down' END AS gap_direction,
	ABS(open - prev_close) AS gap_size,
	CASE WHEN CLOSE > OPEN THEN 1 ELSE 0 END AS rth_bullish,
	CASE WHEN CLOSE > prev_close THEN 1 ELSE 0 END AS rth_close_higher_than_pd
FROM rth_ohlcv_prev
WHERE prev_close IS NOT NULL
)
SELECT 
	weekday,
	gap_direction,
	ROUND(AVG(gap_size), 2) AS avg_gap_size,
	round(SUM(rth_bullish) / COUNT(*)::NUMERIC * 100, 2) AS rth_bullish_pct,
	round(SUM(rth_close_higher_than_pd) / COUNT(*)::NUMERIC * 100, 2) AS rth_close_above_prev_pct
FROM prefinal_agg
GROUP BY weekday, gap_direction

weekday	gap_direction	avg_gap_size	rth_bullish_pct	rth_close_above_prev_pct
Thursday	gap_down	138.91	45.45	18.18
Monday	gap_up	222.95	45	80
Thursday	gap_up	167.25	29.41	52.94
Tuesday	gap_down	157.92	66.67	33.33
Friday	gap_down	195.58	73.33	46.67
Friday	gap_up	110.95	50	65
Wednesday	gap_up	138.44	54.55	77.27
Wednesday	gap_down	71.48	58.33	50
Monday	gap_down	169.86	77.78	50
Tuesday	gap_up	91.33	44.44	44.44

---

## Submission Instructions

Paste your queries and results. Log query ID: RTH-GLOB-001.
