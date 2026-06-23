# NQ Project — Week 27 Day 2

**Generated:** 2026-06-24
**Focus:** FOMC reversal with spike-extreme anchoring + OR delta × weekday breakdown

---

## Task 1: FOMC Spike Reversal — Anchored to Spike Extreme

**Scenario:**
RTH-NEWS-003a established the FOMC spike framework: pre-event reference price, reaction range 14:00–15:00, spike direction. The naive reversal check (did price return to pre_event_price?) was identified as flawed — if price spiked up first, the relevant reversal question is: did it then come back DOWN through the pre_event_price, not just touch it on the way up. The correct anchor is the spike extreme. **(ID: RTH-NEWS-005)**

Build on RTH-NEWS-003a. After establishing `spike_direction`, `reaction_high_time`, `reaction_low_time`:

- If `spike_direction = 'up'`: the spike extreme is `reaction_high`. Reversal = did price trade **below** `pre_event_price` at any point **after** `reaction_high_time` before 16:00?
- If `spike_direction = 'down'`: the spike extreme is `reaction_low`. Reversal = did price trade **above** `pre_event_price` at any point **after** `reaction_low_time` before 16:00?

Add to the existing event_agg CTE:
```sql
reversed BOOLEAN — TRUE if price crossed back through pre_event_price after the spike extreme
```

Then in the final SELECT also add:
- `rth_close` — from `nq_data.daily_ohlcv_rth`
- `close_vs_pre_event` — rth_close - pre_event_price

**Output — one row per FOMC date:**
- All columns from RTH-NEWS-003a
- `reversed`
- `rth_close`
- `close_vs_pre_event`

Then add a summary aggregation at the bottom:
- `spike_direction`, `reversed`, `days`, `avg_close_vs_pre_event`

**Tables:** `nq_data.ticks`, `nq_data.news_events`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 5/5

**Hint for reversal check:** You need a new CTE that joins ticks back to event_agg to find whether any tick after the spike extreme timestamp satisfies the reversal condition. Use `EXISTS` or a `MIN(ts_event)` approach — whichever feels cleaner.

Complex query that took lots of time and processing as every query took time and I've made some mistakes in the process. And I've also created a VIEW with RTH-NEWS-003a before to meke things smoother (used VIEW, not MATERIALIZED VIEW - what are the differences/implications?). Also, I must add that spike magnitude in RTH-NEWS-003a is wrong, as it calculates the move from reaction_high to reaction_low, yet it should be from the pre_event_price to the spike_extreme. We can fix it next time and recreate the view.


WITH nq_fomc AS (
SELECT 
	*,
	CASE 
		WHEN spike_direction = 'up' THEN reaction_high_time_et
		WHEN spike_direction = 'down' THEN reaction_low_time_et
	END AS spike_extreme_time,
	CASE 
		WHEN spike_direction = 'up' THEN reaction_high
		WHEN spike_direction = 'down' THEN reaction_low 
	END AS spike_extreme
FROM nq_data.fomc_agg
),
spikes_agg AS (
SELECT
	*,
	CASE
		WHEN n.spike_direction = 'up' AND t.price < n.pre_event_price THEN 1
		WHEN n.spike_direction = 'down' AND t.price > n.pre_event_price THEN 1
	ELSE 0
	END AS crossed_the_spike_extreme
FROM nq_fomc n
JOIN nq_data.ticks t ON (t.ts_event AT TIME ZONE 'America/New_York') > n.spike_extreme_time
AND (t.ts_event AT TIME ZONE 'America/New_York')::time < '16:00'
AND (t.ts_event AT TIME ZONE 'America/New_York')::date = n.event_date 
),
post_fomc_spike_reversed_agg AS (
SELECT 
	event_date,
	pre_event_price,
	reaction_low,
	reaction_low_time_et,
	reaction_high,
	reaction_high_time_et,
	spike_direction,
	spike_extreme,
	MAX(crossed_the_spike_extreme) AS reversed
FROM spikes_agg s
GROUP BY event_date, pre_event_price, reaction_low, reaction_low_time_et, reaction_high, reaction_high_time_et, spike_direction, spike_extreme
),
fomc_rth_agg AS (
SELECT 
	d.trade_date,
	d.weekday,
	p.pre_event_price,
	p.reaction_low,
	p.reaction_low_time_et,
	p.reaction_high,
	p.reaction_high_time_et,
	p.spike_direction,
	p.spike_extreme,
	p.reversed,
	d.close AS rth_close,
	d.close - p.pre_event_price AS close_vs_pre_event
FROM post_fomc_spike_reversed_agg p
JOIN nq_data.daily_ohlcv_rth d ON p.event_date = d.trade_date
)
SELECT
	spike_direction,
	reversed,
	COUNT(*) AS days,
	ROUND(AVG(close_vs_pre_event), 2) AS avg_close_vs_pre_event
FROM fomc_rth_agg
GROUP BY spike_direction, reversed

Data's intersting:

spike_direction	reversed	days	avg_close_vs_pre_event
down	1	4	107.94
up	1	3	-56.17

---

## Task 2: OR Delta × Weekday Breakout Rates

**Scenario:**
RTH-ORB-001b showed that bullish OR delta → break_up 52%, bearish OR delta → break_down 51% across all days. But Tuesday is structurally bearish and Thursday fades — does the OR delta signal hold on those days, or does weekday character override it? **(ID: RTH-ORB-002)**

Extend the RTH-ORB-001b query to add `weekday` as a grouping dimension.

**Output:**
- `weekday`
- `or_delta_direction` — bullish / bearish
- `breakout_direction` — break_up / break_down / break_both / no_break
- `days`
- `within_group_pct` — days / total days in that weekday × or_delta_direction group, rounded to 1

Filter out `or_delta_direction = 'flat'`. Order by `weekday`, `or_delta_direction`, `days DESC`.

Focus your findings commentary on: does the OR delta → breakout alignment hold on Tuesday and Thursday, or do those weekdays show divergence?

**Tables:** `nq_data.ticks`

**Difficulty Rating:** 4/5


WITH ticks_date_or_range AS (
SELECT 
	*,
	(ts_event AT TIME ZONE 'America/New_York')::date AS trade_date,
	ts_recv AT TIME ZONE 'America/New_York' AS time_et,
	TRIM(TO_CHAR(ts_recv AT TIME ZONE 'America/New_York', 'Day')) AS day_of_week
FROM nq_data.ticks t
WHERE (ts_event AT TIME ZONE 'America/New_York')::TIME >= '9:30' AND (ts_event AT TIME ZONE 'America/New_York')::TIME <= '10:00'
AND side != 'N'
),
or_range_delta AS (
SELECT
	trade_date,
	MAX(price) AS or_high,
	MIN(price) AS or_low,
	SUM(size) FILTER (WHERE side='B') - SUM(size) FILTER (WHERE side='A') AS or_volume_delta,
	CASE 
		WHEN SUM(size) FILTER (WHERE side='B') - SUM(size) FILTER (WHERE side='A') = 0 THEN 'flat'
		WHEN SUM(size) FILTER (WHERE side='B') - SUM(size) FILTER (WHERE side='A') > 0 THEN 'bullish' ELSE 'bearish' 
	END AS or_delta_direction
FROM ticks_date_or_range
GROUP BY trade_date
),
rest_day_or_range_from_10_00 AS (
SELECT 
	*,
	(ts_event AT TIME ZONE 'America/New_York')::date AS trade_date,
	ts_recv AT TIME ZONE 'America/New_York' AS time_et,
	TRIM(TO_CHAR(ts_recv AT TIME ZONE 'America/New_York', 'Day')) AS day_of_week
FROM nq_data.ticks t
WHERE (ts_event AT TIME ZONE 'America/New_York')::TIME >= '10:00' AND (ts_event AT TIME ZONE 'America/New_York')::TIME <= '16:00'
AND side != 'N'
),
rest_range_delta AS (
SELECT
	trade_date,
	MAX(price) AS rest_high,
	MIN(price) AS rest_low,
	SUM(size) FILTER (WHERE side='B') - SUM(size) FILTER (WHERE side='A') AS rest_volume_delta,
	CASE 
		WHEN SUM(size) FILTER (WHERE side='B') - SUM(size) FILTER (WHERE side='A') = 0 THEN 'flat'
		WHEN SUM(size) FILTER (WHERE side='B') - SUM(size) FILTER (WHERE side='A') > 0 THEN 'bullish' ELSE 'bearish' 
	END AS rest_delta_direction
FROM rest_day_or_range_from_10_00
GROUP BY trade_date
),
or_rest_agg AS (
SELECT
	r.trade_date,
	TRIM(TO_CHAR(r.trade_date AT TIME ZONE 'America/New_York', 'Day')) AS day_of_week,
	r.rest_high,
	r.rest_low,
	r.rest_delta_direction,
	o.or_high,
	o.or_low,
	or_delta_direction,
	CASE 
		WHEN rest_high > or_high AND rest_low < or_low THEN 'break_both'
		WHEN rest_high > or_high THEN 'break_up'
		WHEN rest_low < or_low THEN 'break_down'
		WHEN rest_high < or_high AND rest_low > or_low THEN 'no_break'
	END AS breakout_direction
FROM rest_range_delta r
JOIN or_range_delta o ON r.trade_date = o.trade_date
),
breaks_agg AS (
SELECT 
	or_delta_direction,
	breakout_direction,
	day_of_week,
	COUNT(*) AS days,
	ROUND(COUNT(*) / SUM(COUNT(*)) OVER (PARTITION BY or_delta_direction, day_of_week)::NUMERIC * 100, 2) AS within_group_pct
FROM or_rest_agg
GROUP BY or_delta_direction, breakout_direction, day_of_week
)
SELECT 
	*
FROM breaks_agg
ORDER BY day_of_week




or_delta_direction	breakout_direction	day_of_week	days	within_group_pct
bearish	break_both	Monday	8	42.11
bearish	break_up	Monday	6	31.58
bearish	break_down	Monday	5	26.32
bullish	break_both	Monday	8	57.14
bullish	break_down	Monday	3	21.43
bullish	break_up	Monday	3	21.43
bearish	break_up	Sunday	4	30.77
bullish	break_both	Sunday	6	30
bullish	break_up	Sunday	13	65
bullish	break_down	Sunday	1	5
bearish	break_down	Sunday	8	61.54
bearish	break_both	Sunday	1	7.69
bearish	break_up	Thursday	3	21.43
bearish	break_both	Thursday	4	28.57
bearish	break_down	Thursday	7	50
bullish	break_both	Thursday	3	20
bullish	break_down	Thursday	3	20
bullish	break_up	Thursday	9	60
bearish	break_down	Tuesday	10	58.82
bullish	break_up	Tuesday	10	62.5
bullish	break_down	Tuesday	2	12.5
bullish	break_both	Tuesday	4	25
bearish	break_both	Tuesday	3	17.65
bearish	break_up	Tuesday	4	23.53
bearish	break_down	Wednesday	12	63.16
bearish	break_up	Wednesday	2	10.53
bearish	break_both	Wednesday	5	26.32
bullish	break_both	Wednesday	5	41.67
bullish	break_down	Wednesday	2	16.67
bullish	break_up	Wednesday	5	41.67

---

## Submission Instructions

Paste your query and results. Log query IDs: RTH-NEWS-005, RTH-ORB-002.
