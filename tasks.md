# SQL Tasks — Week 32 Day 5

**Generated:** 2026-07-31
**Dataset:** nq_data
**Focus:** EU levels analysis, Friday after Thursday behavior

---

## Task A: RTH-GLOB-008 — EU Levels vs RTH by Day Direction
**Difficulty: 4/5**

**Business question:**
For each weekday × gap direction combination, compute:
- On **bullish RTH days** (close > open): how often does RTH undercut EU midpoint? EU low?
- On **bearish RTH days** (close < open): how often does RTH exceed EU high?

EU session = 02:00–09:30 ET ticks. Gap direction = rth_open vs prior RTH close (LAG).
Include N per cell. HAVING >= 5.

**Expected output columns:**
`weekday, gap_direction, day_direction, n, pct_undercut_eu_mid, pct_undercut_eu_low, pct_exceeded_eu_high`

Hint: `pct_exceeded_eu_high` is only meaningful on bearish days; `pct_undercut_*` only on bullish days — you can NULL out the irrelevant columns per row, or just compute all three and let the reader filter mentally.

**Difficulty: 4/5**

WITH eu_agg_prev_open AS (
SELECT 
	* ,
	LAG(rth_close) OVER (ORDER BY trade_date) AS prev_close
FROM nq_data.eu_rth_session_agg
),
eu_agg_final AS (
SELECT 
	*,
	CASE WHEN rth_open > prev_close THEN 'gap_up' ELSE 'gap_down' END AS gap_direction,
	CASE WHEN rth_close > rth_open THEN 'bullish' ELSE 'bearish' END AS day_direction,
	CASE WHEN rth_low < eu_low THEN 1 ELSE 0 END AS eu_low_undercut,
	CASE WHEN rth_high < eu_high THEN 1 ELSE 0 END AS eu_high_uppercut,
	CASE WHEN rth_low < eu_midpoint THEN 1 ELSE 0 END AS rth_low_below_eu_midpoint,
	CASE WHEN rth_high > eu_midpoint THEN 1 ELSE 0 END AS rth_high_above_eu_midpoint
FROM eu_agg_prev_open
WHERE prev_close IS NOT NULL
)
SELECT
	weekday,
	gap_direction,
	day_direction,
	COUNT(*) AS n,
	ROUND(SUM(rth_low_below_eu_midpoint) / COUNT(*)::NUMERIC * 100, 2) AS pct_undercut_eu_mid,
	ROUND(SUM(rth_high_above_eu_midpoint) / COUNT(*)::NUMERIC * 100, 2) AS pct_uppercut_eu_mid,
	ROUND(SUM(eu_low_undercut) / COUNT(*)::NUMERIC * 100, 2) AS pct_undercut_eu_low,
	ROUND(SUM(eu_high_uppercut) / COUNT(*)::NUMERIC * 100, 2) AS pct_uppercut_eu_high
FROM eu_agg_final
GROUP BY weekday, gap_direction, day_direction
ORDER BY weekday


Look, I've included stats relevant for both bullish and bearish days.
They are signed so it's easy to distinguish them, AND EVEN THOUGH obviously pct_uppercut_eu_mid is quite irrelevan t on a bullish day, it's obvious it's not meant to be relevant here, but it's useful for bearish days, same thing for undercut_eu_mid on bearish days. I think it's the best and most reasonable approach

And also fuck it, I've included all Ns - we will reuse this query in the future as I download more data, BUT for now simply you will warn me of n when summing up the data - it's easy as that, I will be the final judge anyway.

Stats:


weekday	gap_direction	day_direction	n	pct_undercut_eu_mid	pct_uppercut_eu_mid	pct_undercut_eu_low	pct_uppercut_eu_high
Friday	gap_down	bearish	4	100	75	100	75
Friday	gap_down	bullish	7	85.71	85.71	71.43	28.57
Friday	gap_up	bearish	8	100	87.5	87.5	25
Friday	gap_up	bullish	10	70	100	30	10
Monday	gap_down	bearish	4	100	75	100	50
Monday	gap_down	bullish	10	70	100	50	20
Monday	gap_up	bearish	9	88.89	100	66.67	22.22
Monday	gap_up	bullish	9	44.44	100	33.33	0
Thursday	gap_down	bearish	11	100	45.45	100	90.91
Thursday	gap_down	bullish	7	100	100	71.43	28.57
Thursday	gap_up	bearish	8	87.5	100	62.5	50
Thursday	gap_up	bullish	5	100	100	60	20
Tuesday	gap_down	bearish	7	100	100	71.43	42.86
Tuesday	gap_down	bullish	11	100	100	81.82	18.18
Tuesday	gap_up	bearish	7	100	71.43	100	71.43
Tuesday	gap_up	bullish	8	37.5	100	25	0
Wednesday	gap_down	bearish	5	100	60	100	100
Wednesday	gap_down	bullish	6	66.67	100	16.67	0
Wednesday	gap_up	bearish	10	100	100	100	70
Wednesday	gap_up	bullish	12	75	100	33.33	0

---

## Task B: RTH-GLOB-009 — Friday After Bullish vs Bearish Thursday
**Difficulty: 4/5**

**Business question:**
Does Thursday's direction affect Friday's behavior? For each group (prior Thursday bullish vs bearish), show:
- How often Friday gaps up vs down
- Average gap size
- How often Friday closes above its open
- Average RTH range

LAG on Thursday's close direction to tag each Friday.

**Expected output columns:**
`prior_thursday_direction, n, pct_gap_up, avg_gap_size, pct_friday_bullish, avg_rth_range`

**Difficulty: 4/5**



WITH eu_agg_prev_open AS (
SELECT 
	* ,
	LAG(rth_close) OVER (ORDER BY trade_date) AS prev_close,
	LAG(trade_date) OVER (ORDER BY trade_date) AS prev_date
FROM nq_data.eu_rth_session_agg
),
eu_agg_final AS (
SELECT 
	*,
	LAG(weekday) OVER (ORDER BY trade_date) AS prev_day,
	CASE WHEN rth_open > prev_close THEN 'gap_up' ELSE 'gap_down' END AS gap_direction,
	CASE WHEN rth_close > rth_open THEN 'bullish' ELSE 'bearish' END AS day_direction,
FROM eu_agg_prev_open
WHERE prev_close IS NOT NULL
AND weekday IN ('Thursday', 'Friday')
ORDER BY trade_date
),
thu_fri_agg AS (
SELECT 
	*,
	ABS(rth_open - prev_close) AS gap_size,
	rth_high - rth_low AS rth_range,
	LAG(day_direction) OVER (ORDER BY trade_date) AS prev_day_direction
FROM eu_agg_final
)
SELECT 
	prev_day_direction AS prior_thurday_directions,
	COUNT(*) AS n,
	ROUND(COUNT(*) FILTER (WHERE gap_direction = 'gap_up') / count(*)::NUMERIC * 100, 2) AS pct_gap_up,
	ROUND(AVG(gap_size), 2) AS avg_gap_size,
	ROUND(COUNT(*) FILTER (WHERE day_direction = 'bullish') / count(*)::NUMERIC * 100, 2) AS pct_friday_bullish,
	ROUND(AVG(rth_range), 2) AS avg_rth_range
FROM thu_fri_agg
WHERE weekday = 'Friday' AND prev_day = 'Thursday'
GROUP BY prev_day_direction



I've done it step by step and I've left some unused code, but I'm sure it should properly show everything.
I've also added filters to filter out Fridays without thursday before (e.g. because there was a holiday break or sth.)


Stats:


prior_thurday_directions	n	pct_gap_up	avg_gap_size	pct_friday_bullish	avg_rth_range
bearish	19	57.89	128.86	57.89	408.37
bullish	9	66.67	142.92	66.67	282.19
---

## Submission Instructions

Paste your queries below each task.
