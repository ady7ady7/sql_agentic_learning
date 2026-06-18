# NQ Project — Week 26 Day 4

**Generated:** 2026-06-18
**Focus:** News events table + clean vs event-day analysis + 15-min delta by hour_min

---

## Task 1: Create the News Events Table

**Scenario:**
Several of our findings (RTH-CLOSE-001, RTH-VOL-005) may be distorted by FOMC, NFP, CPI and other macro events — especially Wednesday. Before we can isolate clean baseline behavior, we need a reference table of known event dates. Create it and populate it with all relevant events within our dataset window (Sep 2025 – May 2026). **(ID: RTH-NEWS-001)**

Design the table to support future extension to multiple instruments and event types:

```sql
CREATE TABLE nq_data.news_events (
    event_date    DATE NOT NULL,
    event_time_et TIME,
    event_type    TEXT NOT NULL,
    instrument    TEXT NOT NULL DEFAULT 'ALL',
    notes         TEXT
);
```

Then populate it with known macro events from Sep 2025 – May 2026:
- **FOMC** meetings (rate decision days only, not minutes)
- **NFP** (Non-Farm Payrolls — first Friday of each month)
- **CPI** (Consumer Price Index releases)

Use `instrument = 'ALL'` for macro events that affect all futures (FOMC, NFP, CPI).

**Expected output after INSERT:** A SELECT showing all rows ordered by event_date. Paste the result so we can verify coverage.

**Difficulty Rating:** 3/5

**Note:** You'll need to look up the actual dates — these are known in advance from the Fed/BLS calendar. Focus on getting FOMC and NFP right; CPI is a bonus.


Done:


INSERT INTO nq_data.news_events (event_date, event_time_et, event_type, instrument, notes) VALUES
('2025-06-06', '08:30:00', 'NFP', 'ALL', 'Employment Situation for May 2025'),
('2025-06-11', '08:30:00', 'CPI', 'ALL', 'Consumer Price Index for May 2025'),
('2025-06-18', '14:00:00', 'FOMC', 'ALL', 'Interest Rate Decision & Press Conference (w/ SEP Projections)'),
('2025-07-03', '08:30:00', 'NFP', 'ALL', 'Employment Situation for June 2025 (Shifted for July 4th holiday)'),
('2025-07-15', '08:30:00', 'CPI', 'ALL', 'Consumer Price Index for June 2025'),
('2025-07-30', '14:00:00', 'FOMC', 'ALL', 'Interest Rate Decision & Press Conference'),
('2025-08-01', '08:30:00', 'NFP', 'ALL', 'Employment Situation for July 2025'),
('2025-08-12', '08:30:00', 'CPI', 'ALL', 'Consumer Price Index for July 2025'),
('2025-09-05', '08:30:00', 'NFP', 'ALL', 'Employment Situation for August 2025'),
('2025-09-11', '08:30:00', 'CPI', 'ALL', 'Consumer Price Index for August 2025'),
('2025-09-17', '14:00:00', 'FOMC', 'ALL', 'Interest Rate Decision & Press Conference (w/ SEP Projections)'),
('2025-10-15', '08:30:00', 'CPI', 'ALL', 'Consumer Price Index for September 2025'),
('2025-10-29', '14:00:00', 'FOMC', 'ALL', 'Interest Rate Decision & Press Conference'),
('2025-11-20', '08:30:00', 'NFP', 'ALL', 'Employment Situation for September 2025 (Delayed from Oct 3 due to govt funding lapse)'),
('2025-11-13', '08:30:00', 'CPI', 'ALL', 'Consumer Price Index for October 2025'),
('2025-12-10', '14:00:00', 'FOMC', 'ALL', 'Interest Rate Decision & Press Conference (w/ SEP Projections)'),
('2025-12-16', '08:30:00', 'NFP', 'ALL', 'Employment Situation for November 2025'),
('2025-12-18', '08:30:00', 'CPI', 'ALL', 'Consumer Price Index for November 2025'),
('2026-01-09', '08:30:00', 'NFP', 'ALL', 'Employment Situation for December 2025'),
('2026-01-13', '08:30:00', 'CPI', 'ALL', 'Consumer Price Index for December 2025'),
('2026-01-28', '14:00:00', 'FOMC', 'ALL', 'Interest Rate Decision & Press Conference'),
('2026-02-11', '08:30:00', 'NFP', 'ALL', 'Employment Situation for January 2026 (Delayed due to BLS adjustments)'),
('2026-02-13', '08:30:00', 'CPI', 'ALL', 'Consumer Price Index for January 2026 (Delayed due to BLS adjustments)'),
('2026-03-06', '08:30:00', 'NFP', 'ALL', 'Employment Situation for February 2026'),
('2026-03-11', '08:30:00', 'CPI', 'ALL', 'Consumer Price Index for February 2026'),
('2026-03-18', '14:00:00', 'FOMC', 'ALL', 'Interest Rate Decision & Press Conference (w/ SEP Projections)'),
('2026-04-03', '08:30:00', 'NFP', 'ALL', 'Employment Situation for March 2026'),
('2026-04-10', '08:30:00', 'CPI', 'ALL', 'Consumer Price Index for March 2026'),
('2026-04-29', '14:00:00', 'FOMC', 'ALL', 'Interest Rate Decision & Press Conference'),
('2026-05-08', '08:30:00', 'NFP', 'ALL', 'Employment Situation for April 2026'),
('2026-05-12', '08:30:00', 'CPI', 'ALL', 'Consumer Price Index for April 2026'),
('2026-06-05', '08:30:00', 'NFP', 'ALL', 'Employment Situation for May 2026'),
('2026-06-10', '08:30:00', 'CPI', 'ALL', 'Consumer Price Index for May 2026'),
('2026-06-17', '14:00:00', 'FOMC', 'ALL', 'Interest Rate Decision & Press Conference (w/ SEP Projections)');


---

## Task 2: Clean Weekday Close-to-Close vs Event Days

**Scenario:**
RTH-CLOSE-001 showed Wednesday averaging +80 pts close-to-close (67.6% up days) — but FOMC Wednesdays likely skew this heavily. Now that you have `nq_data.news_events`, split every weekday's close-to-close stats into **event days** vs **clean days**. **(ID: RTH-NEWS-002)**

Using `nq_data.daily_ohlcv_rth` LEFT JOIN `nq_data.news_events`:

- `weekday`
- `is_event_day` — TRUE if any news_events row matches that trade_date, FALSE otherwise
- `days`
- `avg_close_change` — LAG(close) based, rounded to 1
- `up_pct` — % of days where close > prev_close, rounded to 1

Order by `weekday`, `is_event_day`.

**Tables:** `nq_data.daily_ohlcv_rth`, `nq_data.news_events`

**Difficulty Rating:** 4/5


WITH fh_r_stats_rth_w_news AS (
SELECT 
	*,
	TRIM(TO_CHAR(trade_date, 'Day')) AS weekday,
	LAG(r_close) OVER (ORDER BY trade_date) AS prev_close,
	r_close - LAG(r_close) OVER (ORDER BY trade_date) AS close_gap,
	CASE WHEN event_date IS NULL THEN FALSE ELSE TRUE END AS is_event_day
FROM nq_data.rth_firsthour_rest_ohlc_ranges rfror
LEFT JOIN nq_data.news_events ne ON rfror.trade_date = ne.event_date
)
SELECT 
	weekday,
	is_event_day,
	COUNT(*) AS days,
	ROUND(AVG(close_gap), 2) AS avg_close_change,
	ROUND(COUNT(*) FILTER (WHERE r_close > prev_close) / COUNT(*)::NUMERIC * 100, 2)  AS up_pct
FROM fh_r_stats_rth_w_news
GROUP BY weekday, is_event_day
ORDER BY weekday, is_event_day

weekday	is_event_day	days	avg_close_change	up_pct
Friday	false	25	10.01	60
Friday	true	4	-13.88	75
Monday	false	33	135.7	66.67
Thursday	false	28	-99.03	35.71
Thursday	true	3	-251.67	33.33
Tuesday	false	31	-6.79	45.16
Tuesday	true	2	-0.38	50
Wednesday	false	24	102.3	70.83
Wednesday	true	9	30.22	66.67

We've got some findings - it would also be nice to see how the price behaves first hour after news events - are there any particular patterns, breakouts, backs to the start of the news range etc.

---

## Task 3: 15-Minute Delta by Hour_Min Window

**Scenario:**
RTH-VOL-006 showed no aggregate edge (51% vs 50.4%) for prev 15-min delta predicting next bucket direction. But specific windows may behave differently — the open (09:30–09:45) and close (15:30–15:45) are structurally different from the dead midday. Break the same analysis down by `hour_min`. **(ID: RTH-VOL-007)**

Take your existing RTH-VOL-006 final query and modify only the SELECT and GROUP BY to add `hour_min`. Everything else stays the same.

**Output:**
- `hour_min` — the bucket being predicted (e.g. '09:45', '10:00' ... '16:00')
- `prev_delta_direction` — 'positive' or 'negative'
- `next_bucket_up` — count where bucket_direction = 'up'
- `total`
- `next_up_pct` — rounded to 1

Order by `hour_min`, `prev_delta_direction`.

Look for windows where next_up_pct diverges meaningfully from 50% in either direction — those are the time-specific edges.

**Tables:** `nq_data.rth_15min_buckets_agg`

**Difficulty Rating:** 3/5


WITH buckets_hr_min AS (
SELECT 
	*,
	TO_CHAR(bucket_start, 'YYYY-MM-DD')::DATE AS trade_day,
	TO_CHAR(bucket_start, 'HH24:MI')::TIME AS hour_min 
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
	ROUND(COUNT(*) FILTER (WHERE bucket_direction = 'up') / COUNT(*)::NUMERIC * 100, 2) AS next_up_pct
FROM buckets_prev2_agg
GROUP BY hour_min, prev_delta_direction_indicator
ORDER BY hour_min, prev_delta_direction_indicator

09:45:00	negative	40	88	45.45
09:45:00	positive	47	71	66.20
10:00:00	negative	41	82	50.00
10:00:00	positive	39	77	50.65
10:15:00	negative	42	77	54.55
10:15:00	positive	47	82	57.32
10:30:00	negative	42	78	53.85
10:30:00	positive	41	81	50.62
10:45:00	negative	37	83	44.58
10:45:00	positive	38	76	50.00
11:00:00	negative	32	67	47.76
11:00:00	positive	57	92	61.96
11:15:00	negative	39	77	50.65
11:15:00	positive	42	82	51.22
11:30:00	negative	46	77	59.74
11:30:00	positive	35	82	42.68
11:45:00	negative	45	83	54.22
11:45:00	positive	42	76	55.26
12:00:00	negative	38	81	46.91
12:00:00	positive	43	78	55.13
12:15:00	negative	48	84	57.14
12:15:00	positive	32	75	42.67
12:30:00	negative	30	72	41.67
12:30:00	positive	45	87	51.72
12:45:00	negative	44	82	53.66
12:45:00	positive	28	77	36.36
13:00:00	negative	50	86	58.14
13:00:00	positive	26	70	37.14
13:15:00	negative	39	82	47.56
13:15:00	positive	38	72	52.78
13:30:00	negative	53	88	60.23
13:30:00	positive	28	66	42.42
13:45:00	negative	33	70	47.14
13:45:00	positive	39	84	46.43
14:00:00	negative	44	85	51.76
14:00:00	positive	32	69	46.38
14:15:00	negative	34	74	45.95
14:15:00	positive	41	80	51.25
14:30:00	negative	40	85	47.06
14:30:00	positive	33	69	47.83
14:45:00	negative	44	82	53.66
14:45:00	positive	47	72	65.28
15:00:00	negative	42	70	60.00
15:00:00	positive	41	84	48.81
15:15:00	negative	31	78	39.74
15:15:00	positive	36	76	47.37
15:30:00	negative	48	83	57.83
15:30:00	positive	31	71	43.66
15:45:00	negative	44	96	45.83
15:45:00	positive	33	58	56.90

Here are all the findings



---

## Submission Instructions

Paste your query and results. Log query IDs: RTH-NEWS-001, RTH-NEWS-002, RTH-VOL-007.
