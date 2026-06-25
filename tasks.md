# NQ Project — Week 27 Day 4

**Generated:** 2026-06-25
**Focus:** OR × FH weekday breakdown + agree-bearish bounce targets + fomc_agg fix

---

## Task 1: OR × FH Signal Combinations by Weekday

**Scenario:**
RTH-ORB-005 found that agree-bearish (both OR and FH bearish) resolves bullish 59.7% of the time — stronger than agree-bullish (58.4%). But is Thursday driving that result? Thursday gaps up today, has a 59% bearish FH tendency, and the 3-way stack (RTH-SESS-003) already shows Thursday gap_up + bullish FH = 0% rest bullish. Breaking RTH-ORB-005 down by weekday will tell us whether agree-bearish is a genuine cross-weekday pattern or a Thursday artefact. **(ID: RTH-ORB-006)**

Extend the RTH-ORB-005 query to add `weekday` as a grouping dimension.

Use:
- `nq_data.or_rest_ohlc_ranges` for OR open/close and rest open/close
- `nq_data.rth_firsthour_rest_ohlc_ranges` for FH open/close

Define directions as before (flat excluded from OR and FH direction; rest direction flat falls into bearish via ELSE).

**Output:**
- `weekday`
- `or_direction` — bullish / bearish
- `fh_direction` — bullish / bearish
- `days`
- `rest_bullish_days`
- `rest_bullish_pct`

Filter out rows where `or_direction = 'flat'` or `fh_direction = 'flat'`. Order by `weekday`, `or_direction`, `fh_direction`.

**Finding to answer:** On which weekday(s) is agree-bearish → rest bullish the strongest? Does Thursday agree-bearish stand out? Is agree-bullish consistently weak on Thursdays (aligning with RTH-SESS-003's 0% finding)?

**Tables:** `nq_data.or_rest_ohlc_ranges`, `nq_data.rth_firsthour_rest_ohlc_ranges`

**Difficulty Rating:** 4/5

**(ID: RTH-ORB-006)**

WITH directions_agg AS (
SELECT 
	r.trade_date,
	TRIM(TO_CHAR(o.trade_date, 'Day')) AS weekday,
	CASE WHEN o.or_close = o.or_open THEN 'flat' WHEN o.or_close > o.or_open THEN 'bullish' ELSE 'bearish' END AS or_direction,
	CASE WHEN r.fh_close = r.fh_open THEN 'flat' WHEN r.fh_close > r.fh_open THEN 'bullish' ELSE 'bearish' END AS fh_direction,
	CASE WHEN r.r_close = r.r_open THEN 'flat' WHEN r.r_close > r.r_open THEN 'bullish' ELSE 'bearish' END AS r_direction
FROM nq_data.rth_firsthour_rest_ohlc_ranges r
JOIN nq_data.or_rest_ohlc_ranges o ON r.trade_date = o.trade_date
),
pre_agg AS (
SELECT 
	*,
	CASE WHEN or_direction = fh_direction THEN TRUE ELSE FALSE END AS signals_agree
FROM directions_agg
),
agg AS (
SELECT
	weekday,
	or_direction,
	fh_direction,
	COUNT(*) AS days,
	COUNT(*) FILTER (WHERE r_direction = 'bullish') AS bullish_days,
	ROUND(COUNT(*) FILTER (WHERE r_direction = 'bullish') / COUNT(*)::NUMERIC * 100, 2) AS bullish_days_pct
FROM pre_agg
GROUP BY weekday, or_direction, fh_direction
)
SELECT
	*
FROM agg


weekday	or_direction	fh_direction	days	bullish_days	bullish_days_pct
Wednesday	bullish	bullish	15	9	60
Tuesday	bullish	bearish	3	2	66.67
Wednesday	bearish	bullish	3	2	66.67
Monday	bearish	bearish	10	6	60
Friday	bullish	bullish	16	9	56.25
Tuesday	bearish	bearish	12	9	75
Friday	bearish	bearish	10	4	40
Thursday	bearish	bullish	3	1	33.33
Monday	bearish	bullish	1	0	0
Thursday	bullish	bullish	10	5	50
Friday	bearish	bullish	1	0	0
Tuesday	bullish	bullish	16	10	62.5
Wednesday	bearish	bearish	12	7	58.33
Wednesday	bullish	bearish	3	0	0
Monday	bullish	bearish	2	2	100
Thursday	bearish	bearish	18	11	61.11
Monday	bullish	bullish	20	12	60
Tuesday	bearish	bullish	2	1	50
Friday	bullish	bearish	2	1	50


---

## Task 2: Agree-Bearish Bounce Targets

**Scenario:**
On the 62 agree-bearish days (both OR and FH bearish → rest bullish 59.7%), how far does the afternoon bounce typically reach? This is directly actionable: if today sets up as agree-bearish, you need to know whether to target OR close, RTH open, or OR high as a realistic exit level. **(ID: RTH-ORB-007)**

Using `nq_data.or_rest_ohlc_ranges` joined to `nq_data.rth_firsthour_rest_ohlc_ranges`:

Filter to agree-bearish days only: `or_close < or_open AND fh_close < fh_open`.

For each day compute whether the rest-of-session high (`r_high`) reached each of these reference levels:
- `reached_or_close` — `r_high >= or_close` (OR close = the 10:00 price, after the 30-min selloff)
- `reached_rth_open` — `r_high >= or_open` (RTH open = OR open = 09:30 price, i.e. full OR reversal)
- `reached_or_high` — `r_high >= or_high` (OR high = the intraday peak during the OR window)

Then aggregate:
- `total_agree_bearish_days`
- `reached_or_close_pct`
- `reached_rth_open_pct`
- `reached_or_high_pct`
- `avg_r_high_vs_or_close` — AVG(r_high - or_close) — how many points above OR close does the bounce typically reach on days it bounces at all? Filter to rest_bullish days only (r_close > r_open) for this metric.

**Tables:** `nq_data.or_rest_ohlc_ranges`, `nq_data.rth_firsthour_rest_ohlc_ranges`

**Difficulty Rating:** 4/5

**(ID: RTH-ORB-007)**


WITH dates_agg AS (
SELECT 
	o.trade_date,
	r.r_high - o.or_close AS r_high_vs_or_close,
	CASE 
		WHEN r.r_high >= or_close THEN TRUE ELSE FALSE 
	END AS reached_or_close,
	CASE 
		WHEN r.r_high >= or_open THEN TRUE ELSE FALSE 
	END AS reached_rth_open,
	CASE 
		WHEN r.r_high >= or_high THEN TRUE ELSE FALSE 
	END AS reached_or_high
FROM nq_data.or_rest_ohlc_ranges o
JOIN nq_data.rth_firsthour_rest_ohlc_ranges r ON o.trade_date = r.trade_date
WHERE o.or_close < o.or_open AND r.fh_close < r.fh_open 
)
SELECT
	COUNT(*) AS total_bearish_days,
	ROUND(COUNT(*) FILTER (WHERE reached_or_close IS TRUE) / COUNT(*)::NUMERIC * 100, 2) AS reached_or_close,
	ROUND(COUNT(*) FILTER (WHERE reached_rth_open IS TRUE) / COUNT(*)::NUMERIC * 100, 2) AS reached_rth_open,
	ROUND(COUNT(*) FILTER (WHERE reached_or_high IS TRUE) / COUNT(*)::NUMERIC * 100, 2) AS reached_or_high,
	ROUND(AVG(r_high_vs_or_close), 2) AS avg_r_high_vs_or_close
FROM dates_agg

total_bearish_days	reached_or_close	reached_rth_open	reached_or_high	avg_r_high_vs_or_close
62	83.87	43.55	32.26	113.15
---

## Bonus: Fix fomc_agg VIEW — Correct spike_magnitude

**Scenario:**
The existing `nq_data.fomc_agg` VIEW has `spike_magnitude = reaction_high - reaction_low` (full reaction window range). The correct definition is `ABS(spike_extreme - pre_event_price)` — the distance from the reference price to the actual spike extreme. **(No new ID — fix to existing view)**

Recreate the view with the corrected spike_magnitude. The spike_extreme depends on spike_direction:
- If `spike_direction = 'up'`: `spike_extreme = reaction_high`, so `spike_magnitude = reaction_high - pre_event_price`
- If `spike_direction = 'down'`: `spike_extreme = reaction_low`, so `spike_magnitude = pre_event_price - reaction_low`

Use `CREATE OR REPLACE VIEW` (not DROP + CREATE) to avoid losing the view definition if you make a typo.

The rest of the view definition stays identical to what was built in RTH-NEWS-003a.

**Difficulty Rating:** 1/5 (syntax fix only)


I've dropped it instead, as it was easier and quicker honestly as it prompted an error.

CREATE MATERIALIZED VIEW nq_data.fomc_agg AS
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
    CASE WHEN rh.reaction_high_time < rl.reaction_low_time THEN 'up' ELSE 'down' END AS spike_direction,
    CASE WHEN rh.reaction_high_time < rl.reaction_low_time THEN r.reaction_high - p.pre_event_price ELSE p.pre_event_price - r.reaction_low END AS spike_magnitude
FROM reaction_prices r
JOIN pre_event p ON r.date = p.date
JOIN reaction_low_times rl ON r.date = rl.date
JOIN reaction_high_times rh ON r.date = rh.date
ORDER BY r.date
)
SELECT * FROM event_agg


---

## Submission Instructions

Paste your queries and results. Log query IDs: RTH-ORB-006, RTH-ORB-007.
