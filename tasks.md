# NQ Project — Week 26 Day 5

**Generated:** 2026-06-19
**Focus:** FOMC reaction spike + reversal pattern + 15-min bucket magnitude

---

## Task 1: FOMC Reaction — Spike, Reversal, and Close Location

**Scenario:**
FOMC decisions release at 14:00 ET mid-session. The typical pattern is a sharp directional spike in the minutes after the release, followed by a potential reversal as the market digests the news. We want to quantify: how large is the spike, does it reverse, and where does RTH close relative to the pre-announcement price? **(ID: RTH-NEWS-003a)**

Using `nq_data.ticks` and `nq_data.news_events` (WHERE event_type = 'FOMC'), for each FOMC date:

1. **Pre-event reference price** — last traded price strictly before 14:00 ET (`(ts_event AT TIME ZONE 'America/New_York')::time < '14:00'` on that trade_date, take MAX(ts_event) then point-lookup price)
2. **Reaction range** — high and low in the 14:00–15:00 ET window (first hour after release)
3. **Spike direction** — did price spike up or down first? Use which extreme (high or low) was reached first by MIN(ts_event) within 14:00–15:00
4. **Reversal check** — did price cross back through the pre-event reference price before RTH close (16:00)?
5. **RTH close** — from `nq_data.daily_ohlcv_rth`

**Output — one row per FOMC date:**
- `event_date`
- `pre_event_price` — last price before 14:00
- `reaction_high` — high in 14:00–15:00
- `reaction_low` — low in 14:00–15:00
- `spike_direction` — 'up' if high was hit before low, 'down' if low was hit first
- `spike_magnitude` — reaction_high - reaction_low (total range of the reaction window)
- `reversed` — TRUE if price returned to pre_event_price before 16:00
- `rth_close`
- `close_vs_pre_event` — rth_close - pre_event_price (positive = closed above pre-announcement level)

Order by `event_date`.

**Tables:** `nq_data.ticks`, `nq_data.news_events`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 5/5

**Notes:**
- Only FOMC dates that fall within our tick data range (Sep 2025 – May 2026) will have tick data — earlier dates in news_events can be excluded with an INNER JOIN or date filter
- The aggregate-then-JOIN pattern applies here: find MIN/MAX ts_event first, then point-lookup for prices
- For spike_direction: JOIN back to ticks to find the timestamp of the reaction_high and reaction_low, compare which came first


WITH ticks_fomc_days AS (
    SELECT
        *,
        (ts_event AT TIME ZONE 'America/New_York')::date AS date
    FROM nq_data.ticks t
    JOIN nq_data.news_events ne ON (ts_event AT TIME ZONE 'America/New_York')::date = ne.event_date
    WHERE ne.event_type = 'FOMC'
    AND side != 'N'
),
pre_event_prices AS (
    SELECT
        date,
        MAX(ts_event) AS last_event_ts
    FROM ticks_fomc_days
    WHERE (ts_event AT TIME ZONE 'America/New_York')::time < '14:00'
    GROUP BY date
),
pre_event AS (
    SELECT p.date, t.price AS pre_event_price
    FROM pre_event_prices p
    JOIN ticks_fomc_days t ON t.ts_event = p.last_event_ts
),
reaction_prices AS (
    SELECT
        date,
        MIN(price) AS reaction_low,
        MAX(price) AS reaction_high
    FROM ticks_fomc_days
    WHERE (ts_event AT TIME ZONE 'America/New_York')::time >= '14:00'
      AND (ts_event AT TIME ZONE 'America/New_York')::time <= '15:00'
    GROUP BY date
),
reaction_low_times AS (
    SELECT DISTINCT ON (t.date)
        t.date,
        t.ts_event AS reaction_low_time
    FROM ticks_fomc_days t
    JOIN reaction_prices r ON t.date = r.date
    WHERE (t.ts_event AT TIME ZONE 'America/New_York')::time >= '14:00'
      AND (t.ts_event AT TIME ZONE 'America/New_York')::time <= '15:00'
      AND t.price = r.reaction_low
    ORDER BY t.date, t.ts_event
),
reaction_high_times AS (
    SELECT DISTINCT ON (t.date)
        t.date,
        t.ts_event AS reaction_high_time
    FROM ticks_fomc_days t
    JOIN reaction_prices r ON t.date = r.date
    WHERE (t.ts_event AT TIME ZONE 'America/New_York')::time >= '14:00'
      AND (t.ts_event AT TIME ZONE 'America/New_York')::time <= '15:00'
      AND t.price = r.reaction_high
    ORDER BY t.date, t.ts_event
),
event_agg AS (
SELECT
    r.date AS event_date,
    p.pre_event_price,
    r.reaction_low,
    rl.reaction_low_time AT TIME ZONE 'America/New_York' AS reaction_low_time_et,
    r.reaction_high,
    rh.reaction_high_time AT TIME ZONE 'America/New_York' AS reaction_high_time_et,
    r.reaction_high - r.reaction_low AS spike_magnitude,
    CASE WHEN rh.reaction_high_time < rl.reaction_low_time THEN 'up' ELSE 'down' END AS spike_direction
FROM reaction_prices r
JOIN pre_event p ON r.date = p.date
JOIN reaction_low_times rl ON r.date = rl.date
JOIN reaction_high_times rh ON r.date = rh.date
ORDER BY r.date
)
SELECT * FROM event_agg


Hold up.
This IS ALREADY A VERY LONG QUERY AND WE'VE GOT VEERY VALUABLE INSIGHTS ABOUT HOW THE HIGH/LOW during the first hour are formed and it's very precise!
I don't want to go further than the first hour after news agg - it already could tell us something, BUT THE current approach is a bit poor.

We're exposed to some errors:
- we could've already came back to the pre-event price and we wouldn't even notice that.
It would make much more sense to direct us into the first spike high/low or do something similar, and then do a check based on that from that point onwards. I'm not entirely sure how to approach it, but we need a logical approach that will prevent some heuristics


Anyway, let's stop at this point as this is a very complex QUERY and we've achieved a lot. We could as well easily add a similar aggregation logic for other news types :)).

We'd also could add streamlined logic based on event time instead of hardcoded values, so that it would apply to more news days.

Anyway, it's not a job for today.


---

## Task 2: 15-Minute Bucket Magnitude by Prior Bucket Delta

**Scenario:**
RTH-VOL-007 showed that the 09:45–10:00 bucket closes up 66% of the time when the prior bucket (09:30–09:45) has positive delta. But direction alone doesn't tell us if the edge is tradeable — we need magnitude. If the winning buckets move +2 pts and the losing buckets move -20 pts, the 66% win rate is worthless. **(ID: RTH-VOL-008)**

Using `nq_data.rth_15min_buckets_agg`, extend the RTH-VOL-007 query to add magnitude columns:

- `avg_up_move` — avg (bucket_close - bucket_open) on days where bucket_direction = 'up', rounded to 1
- `avg_down_move` — avg (bucket_close - bucket_open) on days where bucket_direction = 'down' (will be negative), rounded to 1
- `avg_move_all` — avg (bucket_close - bucket_open) across all days in that group, rounded to 1

Focus on the windows identified in RTH-VOL-007 as having the strongest directional edge: 09:45, 11:00, 14:45, 15:00. Filter `hour_min IN ('09:45:00', '11:00:00', '14:45:00', '15:00:00')` to keep output readable.

**Output:**
- `hour_min`
- `prior_bucket_delta_direction` — 'positive' or 'negative'
- `current_bucket_up_pct` — from RTH-VOL-007
- `avg_up_move`
- `avg_down_move`
- `avg_move_all` — the net expected value per trade (positive = edge has real magnitude)
- `total`

Order by `hour_min`, `prior_bucket_delta_direction`.

**Tables:** `nq_data.rth_15min_buckets_agg`

**Difficulty Rating:** 3/5



WITH buckets_hr_min AS (
SELECT 
	*,
	TO_CHAR(bucket_start, 'YYYY-MM-DD')::DATE AS trade_day,
	TO_CHAR(bucket_start, 'HH24:MI')::TIME AS hour_min,
	bucket_close - bucket_open AS move_magnitude
FROM nq_data.rth_15min_buckets_agg
),
buckets_prev_agg AS (
SELECT 
	*,
	trade_day,
	lag(bucket_delta) OVER (PARTITION BY trade_day ORDER BY trade_day, bucket_start) AS prev_delta_direction,
	lag(bucket_direction) OVER (PARTITION BY trade_day ORDER BY trade_day, bucket_start) AS prev_bucket_direction
FROM buckets_hr_min
),
buckets_prev2_agg AS (
SELECT 
	*,
	CASE WHEN prev_delta_direction > 0 THEN 'positive' ELSE 'negative' END AS prev_delta_direction_indicator
FROM buckets_prev_agg
WHERE prev_delta_direction IS NOT NULL
)
SELECT 
	hour_min,
	prev_delta_direction_indicator,
	COUNT(*) FILTER (WHERE bucket_direction = 'up') AS next_bucket_up,
	COUNT(*) AS total,
	AVG(move_magnitude) FILTER (WHERE bucket_direction = 'up') AS avg_move_up,
	AVG(move_magnitude) FILTER (WHERE bucket_direction = 'down') AS avg_move_down,
	AVG(move_magnitude) AS avg_move_all,
	ROUND(COUNT(*) FILTER (WHERE bucket_direction = 'up') / COUNT(*)::NUMERIC * 100, 2) AS next_up_pct
FROM buckets_prev2_agg
WHERE hour_min IN ('09:45:00', '11:00:00', '14:45:00', '15:00:00')
GROUP BY hour_min, prev_delta_direction_indicator
ORDER BY hour_min, prev_delta_direction_indicator


hour_min	prev_delta_direction_indicator	next_bucket_up	total	avg_move_up	avg_move_down	avg_move_all	next_up_pct
09:45:00	negative	40	88	59.1875	-50.53125	-0.6590909091	45.45
09:45:00	positive	47	71	58.1010638298	-54.5416666667	20.0246478873	66.2
11:00:00	negative	32	67	29.7734375	-37.1571428571	-5.1902985075	47.76
11:00:00	positive	57	92	32.1973684211	-38.5214285714	5.2934782609	61.96
14:45:00	negative	44	82	23.9261363636	-26.7565789474	0.4390243902	53.66
14:45:00	positive	47	72	18.7074468085	-19.5	5.4409722222	65.28
15:00:00	negative	42	70	21.0178571429	-18.5357142857	5.1964285714	60
15:00:00	positive	41	84	25.1646341463	-25.5174418605	-0.7797619048	48.81


Then I've done the same aggregation without the filter for the best windows and this is what I've got:


hour_min	prev_delta_direction_indicator	next_bucket_up	total	avg_move_up	avg_move_down	avg_move_all	next_up_pct
09:45:00	negative	40	88	59.1875	-50.53125	-0.6590909091	45.45
09:45:00	positive	47	71	58.1010638298	-54.5416666667	20.0246478873	66.2
10:00:00	negative	41	82	34.237804878	-47.9817073171	-6.8719512195	50
10:00:00	positive	39	77	39.4743589744	-35.1644736842	2.6396103896	50.65
10:15:00	negative	42	77	30.4226190476	-50.0357142857	-6.1493506494	54.55
10:15:00	positive	47	82	39.9361702128	-41.7	5.0914634146	57.32
10:30:00	negative	42	78	49.6904761905	-39.3125	8.6121794872	53.85
10:30:00	positive	41	81	33.1524390244	-37.575	-1.774691358	50.62
10:45:00	negative	37	83	35.722972973	-46.8043478261	-10.015060241	44.58
10:45:00	positive	38	76	32.5394736842	-38.1513157895	-2.8059210526	50
11:00:00	negative	32	67	29.7734375	-37.1571428571	-5.1902985075	47.76
11:00:00	positive	57	92	32.1973684211	-38.5214285714	5.2934782609	61.96
11:15:00	negative	39	77	40.8717948718	-44.6418918919	-0.75	50.65
11:15:00	positive	42	82	26.0535714286	-24.7625	1.2652439024	51.22
11:30:00	negative	46	77	31.9347826087	-40.7833333333	3.1883116883	59.74
11:30:00	positive	35	82	27.6571428571	-27.8031914894	-4.131097561	42.68
11:45:00	negative	45	83	31.5944444444	-35.6315789474	0.8162650602	54.22
11:45:00	positive	42	76	24.5833333333	-19.5294117647	4.8486842105	55.26
12:00:00	negative	38	81	22.6184210526	-27.6845238095	-3.7438271605	46.91
12:00:00	positive	43	78	26.6860465116	-23	4.391025641	55.13
12:15:00	negative	48	84	20.7291666667	-25.4722222222	0.9285714286	57.14
12:15:00	positive	32	75	29.78125	-22.375	0.1766666667	42.67
12:30:00	negative	30	72	23.3166666667	-25.7202380952	-5.2881944444	41.67
12:30:00	positive	45	87	31.2444444444	-22.0548780488	5.7672413793	51.72
12:45:00	negative	44	82	23.9375	-26.625	0.506097561	53.66
12:45:00	positive	28	77	20.4464285714	-26.0957446809	-8.4935064935	36.36
13:00:00	negative	50	86	26.635	-37.5071428571	0.2209302326	58.14
13:00:00	positive	26	70	26.4230769231	-19.8465909091	-2.6607142857	37.14
13:15:00	negative	39	82	23.1474358974	-26.5406976744	-2.9085365854	47.56
13:15:00	positive	38	72	20.7565789474	-27.946969697	-1.8541666667	52.78
13:30:00	negative	53	88	27.7971698113	-29.3897058824	5.3863636364	60.23
13:30:00	positive	28	66	27.1428571429	-20.2763157895	-0.1590909091	42.42
13:45:00	negative	33	70	33.4318181818	-18.2837837838	6.0964285714	47.14
13:45:00	positive	39	84	22.5064102564	-22.9	-1.818452381	46.43
14:00:00	negative	44	85	22.9488636364	-21.512195122	1.5029411765	51.76
14:00:00	positive	32	69	28.890625	-29.1351351351	-2.2246376812	46.38
14:15:00	negative	34	74	24.2058823529	-19.025	0.8378378378	45.95
14:15:00	positive	41	80	24.9268292683	-22.3552631579	2.15625	51.25
14:30:00	negative	40	85	21.2	-17.1333333333	0.9058823529	47.06
14:30:00	positive	33	69	22.9772727273	-31.3055555556	-5.3442028986	47.83
14:45:00	negative	44	82	23.9261363636	-26.7565789474	0.4390243902	53.66
14:45:00	positive	47	72	18.7074468085	-19.5	5.4409722222	65.28
15:00:00	negative	42	70	21.0178571429	-18.5357142857	5.1964285714	60
15:00:00	positive	41	84	25.1646341463	-25.5174418605	-0.7797619048	48.81
15:15:00	negative	31	78	18.5564516129	-22.7712765957	-6.3461538462	39.74
15:15:00	positive	36	76	23.7083333333	-21.2625	0.0394736842	47.37
15:30:00	negative	48	83	21.28125	-18.8071428571	4.3765060241	57.83
15:30:00	positive	31	71	20.6774193548	-25.1375	-5.1338028169	43.66
15:45:00	negative	44	96	30.4545454545	-30.5048076923	-2.5651041667	45.83
15:45:00	positive	33	58	32.6287878788	-26.1	7.3146551724	56.9


---

## Submission Instructions

Paste your query and results. Log query IDs: RTH-NEWS-003a, RTH-VOL-008.
