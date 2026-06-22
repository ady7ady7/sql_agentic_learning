# NQ Project — Week 27 Day 1

**Generated:** 2026-06-22
**Focus:** NFP/CPI reaction spike analysis + delta vs candle direction alignment (Sunday extra)

---

## Extra Task (Sunday Trading Brief): Delta Bias vs Candle Direction Alignment

**Query ID: RTH-VOL-009** — Done during Sunday trading brief, logged here for record.

```sql
WITH bucket_agg AS (
SELECT 
    *,
    CASE 
        WHEN bucket_delta = 0 THEN 'flat'
        WHEN bucket_delta > 0 THEN 'bullish' ELSE 'bearish' 
    END AS delta_bias
FROM nq_data.rth_15min_buckets_agg
)
SELECT 
    delta_bias,
    COUNT(*) AS total_days,
    COUNT(*) FILTER (WHERE bucket_direction = 'up') AS bullish_bucket,
    ROUND(COUNT(*) FILTER (WHERE bucket_direction = 'up') / COUNT(*)::numeric * 100, 2) AS bullish_buckets_pct
FROM bucket_agg
GROUP BY delta_bias
```

| Delta Bias | Total Buckets | Bullish Candle | Bullish % |
|---|---|---|---|
| Bullish | 1,973 | 1,530 | 77.6% |
| Bearish | 2,098 | 532 | 25.4% |
| Flat | 5 | 3 | 60% |

Delta and candle direction agree ~75-77% of the time. ~23% divergence = absorption/exhaustion setups.

---

## Task 1: NFP/CPI Reaction — Spike Range and Direction

**Scenario:**
NFP and CPI release at 08:30 ET — pre-market, during Globex. Unlike FOMC (mid-session at 14:00), the reaction happens before RTH opens. We want to quantify: what is the spike range in the 60 minutes after 08:30, which direction does price spike first, and where does RTH open and close relative to the pre-announcement price? **(ID: RTH-NEWS-003b)**

Using `nq_data.ticks` and `nq_data.news_events` (WHERE event_type IN ('NFP', 'CPI')), for each event date:

1. **Pre-event reference price** — last traded price strictly before 08:30 ET
2. **Reaction range** — high and low in the 08:30–09:30 ET window
3. **Spike direction** — which extreme (high or low) was reached first
4. **RTH open** — first traded price at or after 09:30 ET on that date
5. **RTH close** — from `nq_data.daily_ohlcv_rth`
6. **rth_open_vs_pre_event** — RTH open minus pre_event_price (gap created by the news)
7. **rth_close_vs_pre_event** — RTH close minus pre_event_price (did session close above/below pre-announcement?)

Reuse the same CTE structure from RTH-NEWS-003a (FOMC query) — just change the event_type filter and the time windows.

**Output — one row per event date:**
- `event_date`
- `event_type` — NFP or CPI
- `pre_event_price`
- `reaction_low`, `reaction_low_time_et`
- `reaction_high`, `reaction_high_time_et`
- `spike_magnitude` — reaction_high - reaction_low
- `spike_direction` — 'up' or 'down'
- `rth_open`
- `rth_open_vs_pre_event`
- `rth_close_vs_pre_event`

Order by `event_date`.

**Tables:** `nq_data.ticks`, `nq_data.news_events`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 4/5

**Note:** RTH open requires a point-lookup for the first tick at or after 09:30 ET — same aggregate-then-JOIN pattern used throughout. Some NFP dates fall outside our tick data window (pre Sep 2025) — INNER JOIN will exclude them automatically.



WITH ticks_fomc_days AS (
    SELECT
        *,
        (ts_event AT TIME ZONE 'America/New_York')::date AS date
    FROM nq_data.ticks t
    JOIN nq_data.news_events ne ON (ts_event AT TIME ZONE 'America/New_York')::date = ne.event_date
    WHERE ne.event_type IN ('CPI', 'NFP') AND side != 'N'
),
pre_event_prices AS (
    SELECT
        date,
        MAX(ts_event) AS last_event_ts
    FROM ticks_fomc_days
    WHERE (ts_event AT TIME ZONE 'America/New_York')::time < '8:30'
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
        event_type,
        MIN(price) AS reaction_low,
        MAX(price) AS reaction_high
    FROM ticks_fomc_days
    WHERE (ts_event AT TIME ZONE 'America/New_York')::time >= '8:30'
      AND (ts_event AT TIME ZONE 'America/New_York')::time <= '9:30'
    GROUP BY date, event_type
),
reaction_low_times AS (
    SELECT DISTINCT ON (t.date)
        t.date,
        t.ts_event AS reaction_low_time
    FROM ticks_fomc_days t
    JOIN reaction_prices r ON t.date = r.date
    WHERE (ts_event AT TIME ZONE 'America/New_York')::time >= '8:30'
      AND (ts_event AT TIME ZONE 'America/New_York')::time <= '9:30'
      AND t.price = r.reaction_low
    ORDER BY t.date, t.ts_event
),
reaction_high_times AS (
    SELECT DISTINCT ON (t.date)
        t.date,
        t.ts_event AS reaction_high_time
    FROM ticks_fomc_days t
    JOIN reaction_prices r ON t.date = r.date
    WHERE (ts_event AT TIME ZONE 'America/New_York')::time >= '8:30'
      AND (ts_event AT TIME ZONE 'America/New_York')::time <= '9:30'
      AND t.price = r.reaction_high
    ORDER BY t.date, t.ts_event
),
event_agg AS (
SELECT
    r.date AS event_date,
    r.event_type,
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
SELECT DISTINCT ON (event_date, event_type)
	e.event_date,
	e.event_type,
	e.pre_event_price,
	e.reaction_low,
	e.reaction_low_time_et,
	e.reaction_high_time_et,
	e.spike_magnitude,
	e.spike_direction,
	r.fh_open,
	r.fh_open - e.pre_event_price AS rth_open_vs_pre_event,
	r.fh_close - e.pre_event_price AS rth_close_vs_pre_event
FROM event_agg e
JOIN nq_data.rth_firsthour_rest_ohlc_ranges r ON e.event_date = r.trade_date
ORDER BY e.event_date


event_date	event_type	pre_event_price	reaction_low	reaction_low_time_et	reaction_high_time_et	spike_magnitude	spike_direction	fh_open	rth_open_vs_pre_event	rth_close_vs_pre_event
2025-10-15	CPI	24,995.5	24,984	2025-10-15 08:31:35.980	2025-10-15 08:59:25.251	41.5	down	25,002.5	7	105.5
2025-11-13	CPI	25,572.75	25,435	2025-11-13 09:10:15.625	2025-11-13 08:30:15.062	141.5	up	25,477.25	-95.5	-188.75
2025-11-20	NFP	25,140.25	25,130	2025-11-20 08:30:01.071	2025-11-20 08:52:08.686	134.75	down	25,211.5	71.25	118
2025-12-16	NFP	25,065	24,980	2025-12-16 09:01:24.493	2025-12-16 08:32:21.975	155.25	up	24,989.25	-75.75	-65.5
2025-12-18	CPI	24,876.25	24,876.5	2025-12-18 08:30:01.579	2025-12-18 09:27:05.868	176	down	25,048	171.75	211.5
2026-01-09	NFP	25,741.75	25,700	2026-01-09 09:22:32.305	2026-01-09 09:03:13.629	124.75	up	25,714	-27.75	112.25
2026-01-13	CPI	25,911.75	25,905.5	2026-01-13 08:30:00.509	2026-01-13 08:32:06.338	140	down	25,950.5	38.75	-69
2026-02-11	NFP	25,238	25,238.75	2026-02-11 08:30:01.004	2026-02-11 08:56:01.231	213.25	down	25,418.5	180.5	-160.75
2026-02-13	CPI	24,706.25	24,695.5	2026-02-13 08:30:00.415	2026-02-13 08:30:30.185	127.75	down	24,743	36.75	142.25
2026-03-06	NFP	24,863.5	24,612.5	2026-03-06 09:09:31.593	2026-03-06 08:30:01.032	296.25	up	24,685.25	-178.25	-41.5
2026-03-11	CPI	25,000	24,909.25	2026-03-11 08:47:21.505	2026-03-11 08:30:01.029	148.75	up	25,034.75	34.75	27.75
2026-04-10	CPI	25,321	25,291.75	2026-04-10 08:37:54.651	2026-04-10 08:30:03.536	80	up	25,315.25	-5.75	19.5



---

## Task 2: NFP/CPI Spike Summary — Avg Magnitude and Close Location

**Scenario:**
Task 1 gives one row per event. Now aggregate across all NFP days and all CPI days separately to find the average spike behavior. **(ID: RTH-NEWS-004)**

Using the output of Task 1 as a base CTE:

- `event_type` — NFP or CPI
- `days` — number of events in our window
- `avg_spike_magnitude` — avg reaction_high - reaction_low, rounded to 1
- `spike_up_pct` — % of events where spike_direction = 'up', rounded to 1
- `avg_rth_open_vs_pre_event` — avg gap created at RTH open, rounded to 1
- `avg_rth_close_vs_pre_event` — avg where session closed vs pre-announcement, rounded to 1
- `closed_above_pre_event_pct` — % of days RTH closed above pre_event_price, rounded to 1

Order by `event_type`.

**Tables:** `nq_data.ticks`, `nq_data.news_events`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 3/5

WITH ticks_fomc_days AS (
    SELECT
        *,
        (ts_event AT TIME ZONE 'America/New_York')::date AS date
    FROM nq_data.ticks t
    JOIN nq_data.news_events ne ON (ts_event AT TIME ZONE 'America/New_York')::date = ne.event_date
    WHERE ne.event_type IN ('CPI', 'NFP') AND side != 'N'
),
pre_event_prices AS (
    SELECT
        date,
        MAX(ts_event) AS last_event_ts
    FROM ticks_fomc_days
    WHERE (ts_event AT TIME ZONE 'America/New_York')::time < '8:30'
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
        event_type,
        MIN(price) AS reaction_low,
        MAX(price) AS reaction_high
    FROM ticks_fomc_days
    WHERE (ts_event AT TIME ZONE 'America/New_York')::time >= '8:30'
      AND (ts_event AT TIME ZONE 'America/New_York')::time <= '9:30'
    GROUP BY date, event_type
),
reaction_low_times AS (
    SELECT DISTINCT ON (t.date)
        t.date,
        t.ts_event AS reaction_low_time
    FROM ticks_fomc_days t
    JOIN reaction_prices r ON t.date = r.date
    WHERE (ts_event AT TIME ZONE 'America/New_York')::time >= '8:30'
      AND (ts_event AT TIME ZONE 'America/New_York')::time <= '9:30'
      AND t.price = r.reaction_low
    ORDER BY t.date, t.ts_event
),
reaction_high_times AS (
    SELECT DISTINCT ON (t.date)
        t.date,
        t.ts_event AS reaction_high_time
    FROM ticks_fomc_days t
    JOIN reaction_prices r ON t.date = r.date
    WHERE (ts_event AT TIME ZONE 'America/New_York')::time >= '8:30'
      AND (ts_event AT TIME ZONE 'America/New_York')::time <= '9:30'
      AND t.price = r.reaction_high
    ORDER BY t.date, t.ts_event
),
event_agg AS (
SELECT
    r.date AS event_date,
    r.event_type,
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
),
final_agg AS (
SELECT DISTINCT ON (event_date, event_type)
	e.event_date,
	e.event_type,
	e.pre_event_price,
	e.reaction_low,
	e.reaction_low_time_et,
	e.reaction_high_time_et,
	e.spike_magnitude,
	e.spike_direction,
	r.fh_open,
	r.fh_open - e.pre_event_price AS rth_open_vs_pre_event,
	r.fh_close - e.pre_event_price AS rth_close_vs_pre_event
FROM event_agg e
JOIN nq_data.rth_firsthour_rest_ohlc_ranges r ON e.event_date = r.trade_date
ORDER BY e.event_date
)
SELECT
	event_type,
	COUNT(*) AS days,
	ROUND(AVG(spike_magnitude), 2) AS avg_spike_magnitude,
	ROUND(COUNT(*) FILTER (WHERE spike_direction = 'up') / COUNT(*)::NUMERIC * 100, 2) AS spike_up_pct,
	ROUND(AVG(rth_open_vs_pre_event), 2) AS avg_rth_open_vs_pre_event,
	ROUND(AVG(rth_close_vs_pre_event), 2) AS avg_rth_close_vs_pre_event,
	ROUND(COUNT(*) FILTER (WHERE rth_close_vs_pre_event > 0) / COUNT(*)::NUMERIC * 100, 2) AS closed_Above_pre_event_pct
FROM final_agg
GROUP BY event_type



event_type	days	avg_spike_magnitude	spike_up_pct	avg_rth_open_vs_pre_event	avg_rth_close_vs_pre_event	closed_above_pre_event_pct
CPI	7	122.21	42.86	26.82	35.54	71.43
NFP	5	184.85	60	-6	-7.5	40


---

## Submission Instructions

Paste your query and results. Log query IDs: RTH-VOL-009 (Sunday extra), RTH-NEWS-003b, RTH-NEWS-004.
