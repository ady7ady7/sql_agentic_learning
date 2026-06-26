# NQ Project — Week 27 Day 5

**Generated:** 2026-06-26
**Focus:** Agree-bearish bounce targets by weekday + FH extreme × gap direction

---

## Task 1: Agree-Bearish Bounce Targets by Weekday

**Scenario:**
RTH-ORB-007 established aggregate bounce targets for all 62 agree-bearish days: OR close reached 84%, RTH open 44%, OR high 32%, avg +113 pts above OR close. But RTH-ORB-006 showed agree-bearish rest bullish % varies significantly by weekday (Tuesday 75% vs Friday 40%). Do the bounce targets also vary — i.e. on a Tuesday agree-bearish day, does the bounce reach RTH open more reliably than on a Wednesday? **(ID: RTH-ORB-008)**

Extend RTH-ORB-007 to add `weekday` as a grouping dimension.

Use:
- `nq_data.or_rest_ohlc_ranges` for OR open/close/high and rest high
- `nq_data.rth_firsthour_rest_ohlc_ranges` for FH open/close

Filter: agree-bearish days only (`or_close < or_open AND fh_close < fh_open`).

For each day compute:
- `reached_or_close` — `r.r_high >= o.or_close`
- `reached_rth_open` — `r.r_high >= o.or_open`
- `reached_or_high` — `r.r_high >= o.or_high`
- `rest_bullish` — `r.r_close > r.r_open`

**Output per weekday:**
- `weekday`
- `days`
- `rest_bullish_pct`
- `reached_or_close_pct`
- `reached_rth_open_pct`
- `reached_or_high_pct`

Order by `rest_bullish_pct DESC`.

**Finding to answer:** On Tuesday (75% rest bullish), do the bounce targets confirm — i.e. high reach rates for RTH open and OR high? On Friday (40% rest bullish), are the reach rates correspondingly low, suggesting the bounce doesn't even get back to OR close?

**Tables:** `nq_data.or_rest_ohlc_ranges`, `nq_data.rth_firsthour_rest_ohlc_ranges`

**Difficulty Rating:** 3/5

**(ID: RTH-ORB-008)**



WITH bearish_only_agg AS (
SELECT
	TRIM(TO_CHAR(o.trade_date, 'Day')) AS weekday,
	r.r_high,
	o.or_close,
	o.or_open,
	o.or_high,
	r.r_close,
	r.r_open
FROM nq_data.or_rest_ohlc_ranges o
JOIN nq_data.rth_firsthour_rest_ohlc_ranges r ON o.trade_date = r.trade_date
WHERE o.or_close < o.or_open AND r.fh_close < r.fh_open
),
scenarios_agg AS (
SELECT 
	*,
	CASE WHEN r_high >= or_close THEN TRUE ELSE FALSE END AS reached_or_close,
	CASE WHEN r_high >= or_open THEN TRUE ELSE FALSE END AS reached_rth_open,
	CASE WHEN r_high >= or_high THEN TRUE ELSE FALSE END AS reached_or_high,
	CASE WHEN r_close > r_open THEN TRUE ELSE FALSE END AS rest_bullish
FROM bearish_only_agg
)
SELECT
	WEEKDAY,
	count(*) AS days,
	ROUND(COUNT(*) FILTER (WHERE reached_or_close IS TRUE) / count(*)::NUMERIC * 100, 2) AS reached_or_close_pct,
	ROUND(COUNT(*) FILTER (WHERE reached_rth_open IS TRUE) / count(*)::NUMERIC * 100, 2) AS reached_rth_open_pct,
	ROUND(COUNT(*) FILTER (WHERE reached_or_high IS TRUE) / count(*)::NUMERIC * 100, 2) AS reached_or_high_pct,
	ROUND(COUNT(*) FILTER (WHERE rest_bullish IS TRUE) / count(*)::NUMERIC * 100, 2) AS rest_bullish_pct
FROM scenarios_agg
GROUP BY weekday

weekday	days	reached_or_close_pct	reached_rth_open_pct	reached_or_high_pct	rest_bullish_pct
Tuesday	12	91.67	75	58.33	75
Friday	10	90	60	30	40
Wednesday	12	83.33	25	25	58.33
Thursday	18	88.89	33.33	22.22	61.11
Monday	10	60	30	30	60

---

## Task 2: FH High/Low as Day Extreme × Gap Direction

**Scenario:**
RTH-FH-002 showed that Thursday FH sets the day HIGH 54.8% of the time overall. But yesterday's setup was a gap-up Thursday — does gap direction amplify the FH-sets-high tendency? Intuitively: gap-up Thursday → FH is bullish more often (RTH-FH-003: 41% bullish FH on Thu) → but if the FH is the high, does a gap-up make that even more likely? This connects gap direction to the FH extreme-setting behaviour across all weekdays. **(ID: RTH-FH-004)**

Using `nq_data.rth_firsthour_rest_ohlc_ranges` joined to `nq_data.daily_ohlcv_rth`:

Gap direction: `CASE WHEN o.open > LAG(o.close) OVER (ORDER BY o.trade_date) THEN 'gap_up' WHEN o.open < LAG(o.close) OVER (...) THEN 'gap_down' END` — exclude flat opens.

FH sets day high: `fh_high >= daily_high` (where `daily_high` comes from `daily_ohlcv_rth`).
FH sets day low: `fh_low <= daily_low`.

**Output:**
- `weekday`
- `gap_direction`
- `days`
- `fh_sets_high_pct`
- `fh_sets_low_pct`

Filter to weekdays with at least 5 days in the bucket. Order by `weekday`, `gap_direction`.

**Finding to answer:** On gap-up Thursdays, does the FH-sets-high rate rise above the 54.8% overall Thursday baseline? On gap-down Thursdays, does FH-sets-low dominate instead? Does the gap direction × FH extreme pattern hold consistently across other weekdays too?

**Tables:** `nq_data.rth_firsthour_rest_ohlc_ranges`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 4/5

**(ID: RTH-FH-004)**



WITH gap_agg AS (
SELECT
	rf.trade_date,
	CASE WHEN dor.open > LAG(dor.close) OVER (ORDER BY dor.trade_date) THEN 'gap_up' WHEN dor.open < LAG(dor.close) OVER (ORDER BY dor.trade_date) THEN 'gap_down' END AS gap_direction,
	dor.high AS daily_high,
	dor.low AS daily_low,
	dor.weekday,
	fh_high,
	fh_low,
	CASE WHEN fh_open < fh_close THEN 'bullish' ELSE 'bearish' END AS fh_direction
FROM nq_data.rth_firsthour_rest_ohlc_ranges rf
JOIN nq_data.daily_ohlcv_rth dor ON rf.trade_date = dor.trade_date
),
days_agg AS (
SELECT 
	*,
	CASE WHEN fh_high >= daily_high THEN TRUE ELSE FALSE END AS fh_sets_high,
	CASE WHEN fh_low <= daily_low THEN TRUE ELSE FALSE END AS fh_sets_low
FROM gap_agg
WHERE gap_direction IS NOT NULL
)
SELECT 
	weekday,
	gap_direction,
	COUNT(*) AS days,
	ROUND(COUNT(*) FILTER (WHERE fh_sets_high IS TRUE) / COUNT(*)::NUMERIC * 100, 2) AS fh_sets_high_pct,
	ROUND(COUNT(*) FILTER (WHERE fh_sets_low IS TRUE) / COUNT(*)::NUMERIC * 100, 2) AS fh_sets_low_pct
FROM days_agg
GROUP BY weekday, gap_direction

weekday	gap_direction	days	fh_sets_high_pct	fh_sets_low_pct
Thursday	gap_down	21	57.14	38.1
Monday	gap_up	20	45	40
Thursday	gap_up	17	58.82	35.29
Tuesday	gap_down	21	19.05	52.38
Friday	gap_down	15	20	66.67
Friday	gap_up	20	50	50
Wednesday	gap_up	22	50	50
Wednesday	gap_down	12	33.33	41.67
Monday	gap_down	18	27.78	66.67
Tuesday	gap_up	18	55.56	22.22


---

## Submission Instructions

Paste your queries and results. Log query IDs: RTH-ORB-008, RTH-FH-004.
After submission: MODE C (session wrap + weekly recap).
