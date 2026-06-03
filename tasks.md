# NQ Project — Layer 1 completion + first analytical queries

**Generated:** 2026-06-03
**Week 24, Day 2 Focus:** RTH materialized view + daily range analysis + buy/sell pressure by weekday

---

## Task 1: RTH Materialized View

**Scenario:**
Build the Regular Trading Hours (RTH) daily bar view. Unlike Globex, RTH is simple — same calendar day, fixed window.

RTH session: **09:30:00 ET → 15:59:59.999 ET**, same calendar date.

Create materialized view `nq_data.daily_ohlcv_rth` with identical columns to `daily_ohlcv_globex`:
- `trade_date` — ET calendar date (same as tick date for RTH, no overnight shift needed)
- `weekday`
- `open`, `high`, `low`, `close`
- `total_volume`, `buy_volume`, `sell_volume`, `tick_count`

**Key differences from Globex:**
- Filter ticks: `EXTRACT(HOUR FROM ts_event AT TIME ZONE 'America/New_York') >= 9` and time >= 09:30 ET
- No date shift — `(ts_event AT TIME ZONE 'America/New_York')::date` is the trade_date directly
- Use the same aggregate-then-JOIN pattern for open/close (not window functions)

**Tables:** `nq_data.ticks`

**Difficulty Rating:** 4/5


CREATE MATERIALIZED VIEW nq_data.daily_ohlcv_rth AS
WITH rth_ticks AS (
    SELECT
        (ts_event AT TIME ZONE 'America/New_York')::date AS trade_date,
        price,
        size,
        side,
        ts_event
    FROM nq_data.ticks
    WHERE (ts_event AT TIME ZONE 'America/New_York')::time >= '09:30'
      AND (ts_event AT TIME ZONE 'America/New_York')::time < '16:00'
      AND side != 'N'
),
agg AS (
    SELECT
        trade_date,
        MIN(ts_event) AS open_time,
        MAX(ts_event) AS close_time,
        MAX(price)    AS high,
        MIN(price)    AS low,
        SUM(size)     AS total_volume,
        SUM(size) FILTER (WHERE side = 'B') AS buy_volume,
        SUM(size) FILTER (WHERE side = 'A') AS sell_volume,
        COUNT(*)      AS tick_count
    FROM rth_ticks
    GROUP BY trade_date
)
SELECT
    a.trade_date,
    TRIM(TO_CHAR(a.trade_date, 'Day')) AS weekday,
    t_open.price  AS open,
    a.high,
    a.low,
    t_close.price AS close,
    a.total_volume,
    a.buy_volume,
    a.sell_volume,
    a.tick_count
FROM agg a
JOIN nq_data.ticks t_open  ON t_open.ts_event  = a.open_time
JOIN nq_data.ticks t_close ON t_close.ts_event = a.close_time
ORDER BY trade_date;

---

## Task 2: Daily Range Analysis by Weekday

**Scenario:**
What is the average daily range for each day of the week? Are some days consistently bigger movers than others?

Using `nq_data.daily_ohlcv_globex`, show for each weekday:
- `weekday`
- `avg_range` — average of (high - low), rounded to 2
- `max_range` — largest single-day range
- `min_range` — smallest single-day range
- `day_count` — how many trading days in the sample

Order by `avg_range DESC`.

**Tables:** `nq_data.daily_ohlcv_globex`

**Difficulty Rating:** 3/5

WITH nq_days_daily_ranges AS (
SELECT 
	*,
	high - low AS daily_range
FROM nq_data.daily_ohlcv_rth
)
SELECT 
	weekday,
	ROUND(AVG(daily_range), 2) AS avg_range,
	MAX(daily_range) AS max_range,
	MIN(daily_range) AS min_range,
	COUNT(*) AS day_count
FROM nq_days_daily_ranges
GROUP BY weekday
ORDER BY avg_range DESC



---

## Task 3: Buy/Sell Pressure by Weekday

**Scenario:**
Which weekdays tend to be buyer-dominated vs seller-dominated? Use buy/sell volume ratio as the pressure metric.

Using `nq_data.daily_ohlcv_globex`, show for each weekday:
- `weekday`
- `avg_buy_volume` — average daily buy volume, rounded to 0
- `avg_sell_volume` — average daily sell volume, rounded to 0
- `avg_ratio` — `avg_buy_volume / NULLIF(avg_sell_volume, 0)`, rounded to 4 — values above 1.0 = buyer dominated, below 1.0 = seller dominated

Order by `avg_ratio DESC`.

**Tables:** `nq_data.daily_ohlcv_globex`

**Difficulty Rating:** 3/5

WITH weekdays_avg_volumes AS (
SELECT 
	weekday,
	ROUND(AVG(buy_volume), 2) AS avg_buy_volume,
	ROUND(AVG(sell_volume), 2) AS avg_sell_volume
FROM nq_data.daily_ohlcv_rth
GROUP BY weekday
)
SELECT 
	*,
	avg_buy_volume / NULLIF(avg_sell_volume, 0) AS avg_volume_ratio
FROM weekdays_avg_volumes
ORDER BY avg_volume_ratio DESC


It's interesting, as it's in a range of 0.0995 - 1.001 at most


---

## Submission Instructions

1. Task 1 — daily_ohlcv_rth materialized view (4/5)
2. Task 2 — daily range by weekday (3/5)
3. Task 3 — buy/sell pressure by weekday (3/5)
