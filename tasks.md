# NQ Project — Layer 1, Task 1

**Generated:** 2026-06-02
**Week 24, Day 1 Focus:** Daily OHLCV materialized view — foundation for all future analysis

---

## Task 1: Daily OHLCV Bars — Globex Session

**Scenario:**
Build the foundational daily bar table for the NQ project. Every future analysis — ranges, session patterns, news day comparisons — will sit on top of this.

A "trading day" follows CME Globex convention:
- Opens: 18:00 ET of the **previous calendar day**
- Closes: 17:00 ET of the **label date**
- So Monday's bar = Sunday 18:00 ET → Monday 17:00 ET
- Label each bar by the **close date**

**Difficulty Rating:** 5/5

**Solution (aggregate-then-JOIN approach):**

```sql
CREATE MATERIALIZED VIEW nq_data.daily_ohlcv_globex AS
WITH trade_dates AS (
    SELECT
        ((ts_event AT TIME ZONE 'America/New_York') - INTERVAL '18 hours')::date + 1 AS trade_date,
        price,
        size,
        side,
        ts_event
    FROM nq_data.ticks
    WHERE side != 'N'
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
    FROM trade_dates
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
```

**Notes:**
- First attempt with FIRST_VALUE window functions ran 10 minutes on 56M rows
- Aggregate-then-JOIN: one GROUP BY pass to find open_time/close_time, then two point lookups back to ticks for the price — far faster
- Trade date shift logic: subtract 18h from ET timestamp, take date, add 1 day — maps overnight ticks to the correct next-day label
