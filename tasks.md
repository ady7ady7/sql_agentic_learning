# NQ Project — Week 26 Day 2

**Generated:** 2026-06-16
**Focus:** Volume imbalance by hour x weekday + ORB breakout direction vs volume delta

---

## Task 1: Buy/Sell Volume Imbalance by Hour × Weekday

**Scenario:**
Yesterday's RTH-VOL-003 showed that RTH volume is nearly balanced at the hourly level. Now break it down by weekday × hour: does Thursday's closing hour (15) show heavier sell pressure than Monday's? Is Monday's post-open (hour 10) the most buy-dominated? **(ID: RTH-VOL-005)**

Using `nq_data.ticks`, filter strictly to RTH (09:30–16:00 ET, side != 'N'). Group by weekday and hour_et.

To get weekday, join to `nq_data.daily_ohlcv_rth` on `(ts_event AT TIME ZONE 'America/New_York')::date = trade_date`.

**Output columns:**
- `weekday`
- `hour_et`
- `buy_volume`
- `sell_volume`
- `delta` — buy_volume - sell_volume
- `delta_pct` — delta / total_volume * 100, rounded to 2

Order by `weekday`, `hour_et`.

**Tables:** `nq_data.ticks`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 4/5

WITH ticks_date_rth AS (
SELECT 
	*,
	(ts_event AT TIME ZONE 'America/New_York')::date AS trade_date,
	ts_recv AT TIME ZONE 'America/New_York' AS time_et,
	TRIM(TO_CHAR(ts_recv AT TIME ZONE 'America/New_York', 'Day')) AS day_of_week,
	EXTRACT('Hour' FROM ts_recv AT TIME ZONE 'America/New_York') AS hour_,
	EXTRACT('DOW' FROM ts_recv AT TIME ZONE 'America/New_York') AS weekday
FROM nq_data.ticks t
WHERE (ts_event AT TIME ZONE 'America/New_York')::TIME >= '9:30' AND (ts_event AT TIME ZONE 'America/New_York')::TIME <= '16:00'
AND side != 'N'
)
SELECT
	day_of_week,
	weekday,
	hour_,
	SUM(size) FILTER (WHERE side = 'B') AS buy_volume,
	SUM(size) FILTER (WHERE side = 'A') AS sell_volume,
	SUM(size) FILTER (WHERE side = 'B') - SUM(size) FILTER (WHERE side = 'A') AS delta,
	ROUND((SUM(size) FILTER (WHERE side = 'B') - SUM(size) FILTER (WHERE side = 'A')) / SUM(size)::NUMERIC * 100, 2) AS delta_pct
FROM ticks_date_rth
GROUP BY day_of_week, weekday, hour_

I didn't use ORDER BY, it will be much easier and way better in terms of memory efficiency for you to order data, findigns below:

day_of_week	weekday	hour_	buy_volume	sell_volume	delta	delta_pct
Friday	5	9	1,011,503	1,008,857	2,646	0.13
Friday	5	10	1,406,882	1,421,771	-14,889	-0.53
Friday	5	11	1,061,982	1,066,966	-4,984	-0.23
Friday	5	12	760,257	759,506	751	0.05
Friday	5	13	706,639	705,793	846	0.06
Friday	5	14	569,392	568,032	1,360	0.12
Friday	5	15	846,739	861,403	-14,664	-0.86
Monday	1	9	982,977	978,218	4,759	0.24
Monday	1	10	1,182,725	1,185,679	-2,954	-0.12
Monday	1	11	843,534	838,912	4,622	0.27
Monday	1	12	629,418	626,810	2,608	0.21
Monday	1	13	525,995	532,894	-6,899	-0.65
Monday	1	14	507,056	506,596	460	0.05
Monday	1	15	827,850	830,309	-2,459	-0.15
Thursday	4	9	1,042,225	1,061,146	-18,921	-0.9
Thursday	4	10	1,409,875	1,401,452	8,423	0.3
Thursday	4	11	1,041,700	1,037,960	3,740	0.18
Thursday	4	12	788,172	792,114	-3,942	-0.25
Thursday	4	13	714,533	717,430	-2,897	-0.2
Thursday	4	14	657,294	661,299	-4,005	-0.3
Thursday	4	15	836,603	846,229	-9,626	-0.57
Tuesday	2	9	1,075,081	1,080,187	-5,106	-0.24
Tuesday	2	10	1,330,688	1,330,818	-130	0
Tuesday	2	11	1,004,997	1,010,126	-5,129	-0.25
Tuesday	2	12	786,812	782,156	4,656	0.3
Tuesday	2	13	719,379	719,988	-609	-0.04
Tuesday	2	14	607,661	613,952	-6,291	-0.51
Tuesday	2	15	898,256	908,762	-10,506	-0.58
Wednesday	3	9	984,199	986,056	-1,857	-0.09
Wednesday	3	10	1,298,164	1,282,288	15,876	0.62
Wednesday	3	11	945,303	950,578	-5,275	-0.28
Wednesday	3	12	733,683	737,195	-3,512	-0.24
Wednesday	3	13	621,739	623,186	-1,447	-0.12
Wednesday	3	14	738,722	735,589	3,133	0.21
Wednesday	3	15	911,525	909,037	2,488	0.14



---

## Task 2: Opening Range Breakout Direction vs Volume Delta

**Scenario:**
The Opening Range (OR) is the high/low established in the first 30 minutes of RTH (09:30–10:00 ET). A breakout above the OR high is a bullish signal; below the OR low is bearish. Does the volume delta during the opening range (net buy vs net sell pressure in those 30 minutes) align with which direction the breakout occurs? **(ID: RTH-ORB-001)**

**Step 1 — compute OR per day from ticks (09:30–10:00 ET):**
- `or_high` = MAX(price)
- `or_low` = MIN(price)
- `or_delta` = SUM(size) FILTER (WHERE side='B') - SUM(size) FILTER (WHERE side='A')
- `or_delta_direction` = 'positive' / 'negative' (exclude flat)

**Step 2 — determine breakout direction using rest-of-session ticks (10:00–16:00 ET):**
For each day, compute:
- `rest_high` = MAX(price) during 10:00–16:00
- `rest_low` = MIN(price) during 10:00–16:00

Then classify:
- `'break_up'` — rest_high > or_high (price exceeded OR high after 10:00)
- `'break_down'` — rest_low < or_low (price broke OR low after 10:00)
- `'break_both'` — both conditions true (two-sided breakout)
- `'no_break'` — price stayed inside OR all session

**Step 3 — summary by or_delta_direction × breakout_direction:**
- `or_delta_direction`
- `breakout_direction`
- `days`
- `pct` — % of days in this bucket out of all days with same or_delta_direction, rounded to 1

Order by `or_delta_direction`, `days DESC`.

**Tables:** `nq_data.ticks`

**Difficulty Rating:** 5/5

**Note:** This query hits raw ticks twice (OR window + rest window). If it's too slow, consider building a materialized view for the 30-min OR first — same pattern as `rth_firsthour_rest_ohlc_ranges`.


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
	COUNT(*) AS days,
	ROUND(COUNT(*) / (SELECT COUNT(trade_date) FROM or_rest_agg)::NUMERIC * 100, 2) AS days_pct
FROM or_rest_agg
GROUP BY or_delta_direction, breakout_direction
)
SELECT 
	*
FROM breaks_agg



Findings:

or_delta_direction	breakout_direction	days	days_pct
bullish	break_up	40	25.16
bearish	break_up	19	11.95
bearish	break_both	21	13.21
bearish	break_down	42	26.42
bullish	break_both	26	16.35
bullish	break_down	11	6.92



That's a very interesting topic.
And it would also be more than beneficial to create a view for that - not as a task - it requires changing 4 numbers in the current fh_rest view.

I'd love to explore whether delta equates to bullish/bearish price as well.
Also, I'd love to explore whether there's a collocation between the first_hour and the rest, no matter the direction. E.g how much chances are there that a given directionb prevails given ORB/fh/delta etc. I really need to check such things.

And it would be lovely to explore this more thoroughly and check 15-min delta's for next 15 mins etc and similar windows. What I'm looking for is ANY and every pattern that can be useful and give me an edge.



---

## Submission Instructions

Paste your query and results for each task. Log query IDs: RTH-VOL-005, RTH-ORB-001.
