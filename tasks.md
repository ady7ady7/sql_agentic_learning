# NQ Project — Week 26 Day 1

**Generated:** 2026-06-15
**Focus:** Tick volume — buy/sell imbalance by hour + first-hour cumulative delta vs day outcome

---

## Task 1: Buy/Sell Volume Imbalance by Hour

**Scenario:**
RTH-VOL-001 showed that buy/sell *tick counts* are nearly perfectly balanced by weekday. But tick count ≠ volume — a single institutional print of 50 contracts on the buy side outweighs 10 retail prints of 1 contract each. Does the same symmetry hold when measured in actual contracts traded? **(ID: RTH-VOL-003)**

Using `nq_data.ticks`, filter to RTH session and exclude N-side. Group by hour of day (ET).

**Output columns:**
- `hour_et` — integer hour (9 through 15)
- `buy_volume` — `SUM(size) FILTER (WHERE side = 'B')`
- `sell_volume` — `SUM(size) FILTER (WHERE side = 'A')`
- `total_volume`
- `buy_pct` — buy_volume / total_volume * 100, rounded to 2
- `delta` — buy_volume - sell_volume (signed, positive = buy dominance)
- `delta_pct` — delta / total_volume * 100, rounded to 2

Order by `hour_et`.

**Tables:** `nq_data.ticks`

**Difficulty Rating:** 3/5

WITH ticks_hours AS (
SELECT 
	*,
	ts_recv AT TIME ZONE 'America/New_York' AS time_et,
	EXTRACT(HOUR FROM ts_event AT TIME ZONE 'America/New_York')::int AS hour_et
FROM nq_data.ticks t
WHERE side != 'N'
)
SELECT
	hour_et,
	SUM(size) FILTER (WHERE side = 'B') AS buy_volume,
	SUM(size) FILTER (WHERE side = 'A') AS sell_volume,
	SUM(size) AS total_volume,
	ROUND(SUM(size) FILTER (WHERE side = 'B') / SUM(size)::NUMERIC * 100, 2) AS buy_pct,
	SUM(size) FILTER (WHERE side = 'B') - SUM(size) FILTER (WHERE side = 'A') AS buy_sell_delta,
	ROUND((SUM(size) FILTER (WHERE side = 'B') - SUM(size) FILTER (WHERE side = 'A')) / SUM(size)::NUMERIC * 100, 2) AS delta_pct
FROM ticks_hours
GROUP BY hour_et
ORDER BY hour_et



buy_volume	sell_volume	total_volume	buy_pct	buy_sell_delta	hour_et	delta_pct
293,333	295,939	589,272	49.78	-2,606	0	-0.44
353,772	349,806	703,578	50.28	3,966	1	0.56
408,325	405,725	814,050	50.16	2,600	2	0.32
596,494	595,255	1,191,749	50.05	1,239	3	0.1
599,881	595,251	1,195,132	50.19	4,630	4	0.39
487,932	485,161	973,093	50.14	2,771	5	0.28
519,752	517,904	1,037,656	50.09	1,848	6	0.18
766,885	760,541	1,527,426	50.21	6,344	7	0.42
1,177,866	1,166,046	2,343,912	50.25	11,820	8	0.5
5,983,060	6,001,275	11,984,335	49.92	-18,215	9	-0.15
6,628,334	6,622,008	13,250,342	50.02	6,326	10	0.05
4,897,516	4,904,542	9,802,058	49.96	-7,026	11	-0.07
3,698,342	3,697,781	7,396,123	50	561	12	0.01
3,288,285	3,299,291	6,587,576	49.92	-11,006	13	-0.17
3,080,125	3,085,468	6,165,593	49.96	-5,343	14	-0.09
4,320,973	4,355,740	8,676,713	49.8	-34,767	15	-0.4
1,351,240	1,351,996	2,703,236	49.99	-756	16	-0.03
563,393	558,874	1,122,267	50.2	4,519	18	0.4
470,385	468,953	939,338	50.08	1,432	19	0.15
585,978	569,812	1,155,790	50.7	16,166	20	1.4
439,182	434,800	873,982	50.25	4,382	21	0.5
353,745	357,132	710,877	49.76	-3,387	22	-0.48
287,434	285,661	573,095	50.15	1,773	23	0.31




---

## Task 2: First Hour Cumulative Delta vs Day Outcome

**Scenario:**
Cumulative delta = running sum of (buy_volume - sell_volume) tick by tick. The *total* first-hour delta tells you whether buyers or sellers were more aggressive in contracts during 09:30–10:30. Does a positive first-hour delta (net buying pressure) predict a bullish full-day close? **(ID: RTH-VOL-004)**

Using `nq_data.ticks`, compute the net first-hour delta per day: `SUM(size) FILTER (WHERE side='B') - SUM(size) FILTER (WHERE side='A')` for ticks between 09:30 and 10:30 ET.

Then join to `nq_data.daily_ohlcv_rth` to get the full-day close direction (`close > open` = bullish day).

Classify each day's first-hour delta as:
- `'positive'` — net buying (delta > 0)
- `'negative'` — net selling (delta < 0)
- `'flat'` — delta = 0 (exclude from summary)

**Output — summary by delta direction:**
- `fh_delta_direction` — 'positive' or 'negative'
- `days`
- `avg_fh_delta` — average net delta in contracts, rounded to 0
- `bullish_day_days` — days where daily close > daily open
- `bullish_day_pct` — rounded to 1

Order by `fh_delta_direction`.

**Tables:** `nq_data.ticks`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 5/5


WITH ticks_dates_first_hour AS (
SELECT 
	*,
	(ts_event AT TIME ZONE 'America/New_York')::date AS trade_date,
	ts_recv AT TIME ZONE 'America/New_York' AS time_et
FROM nq_data.ticks t
WHERE (ts_event AT TIME ZONE 'America/New_York')::TIME >= '9:30' AND (ts_event AT TIME ZONE 'America/New_York')::TIME <= '10:30'
AND side != 'N'
),
fh_vol_agg AS (
SELECT
	 trade_date,
	 SUM(size) FILTER (WHERE side = 'A') AS first_hour_sell_vol,
	 SUM(size) FILTER (WHERE side = 'B') AS first_hour_buy_vol,
	 SUM(size) FILTER (WHERE side = 'B') - SUM(size) FILTER (WHERE side = 'A') AS fh_delta_vol
FROM ticks_dates_first_hour
GROUP BY trade_date
),
fh_vol_delta_agg AS (
SELECT 
	*,
	CASE WHEN fh_delta_vol = 0 THEN 'flat' WHEN fh_delta_vol > 0 THEN 'positive' ELSE 'negative' END AS fh_delta_direction
FROM fh_vol_agg
),
fh_rest_agg AS (
SELECT 
	f.trade_date,
	f.fh_delta_direction,
	f.fh_delta_vol,
	r.fh_open,
	r.fh_close,
	r.r_open,
	r.r_close,
	r.r_open - r.r_close AS rest_range,
	CASE WHEN r.r_close - r.r_open > 0 THEN 'bullish' ELSE 'bearish' END AS rest_direction
FROM fh_vol_delta_agg f
JOIN nq_data.rth_firsthour_rest_ohlc_ranges r ON f.trade_date = r.trade_date
WHERE fh_delta_direction != 'flat'
)
SELECT 
	fh_delta_direction,
	COUNT(*) AS days,
	ROUND(AVG(fh_delta_vol), 2) AS avg_fh_delta,
	COUNT(*) FILTER (WHERE rest_direction = 'bullish') AS bullish_day_days,
	ROUND(COUNT(*) FILTER (WHERE rest_direction = 'bullish') / COUNT(*)::NUMERIC * 100, 2) AS bullish_day_pct
FROM fh_rest_agg
GROUP BY fh_delta_direction



Not sure if I've done 100% right here, but it seems like it. I've done my way of aggregation that's perfectly logical and working :)).

As for the results:

fh_delta_direction	days	avg_fh_delta	bullish_day_days	bullish_day_pct
negative	86	-1,561.69	47	54.65
positive	73	1,601.82	44	60.27


It's interesting to see how the delta affects the bullish/bearish day pcts :)).
I'd love to somehow explore if there's any correlation for ORB and volume deltas etc.
Not sure how to approach this, but it could be interesting - how do pros do that?


---

## Submission Instructions

Paste your query and results for each task. Log query IDs: RTH-VOL-003, RTH-VOL-004.
