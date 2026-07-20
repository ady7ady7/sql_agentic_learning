# NQ Project — Week 30 Day 6

**Generated:** 2026-07-20
**Focus:** Intraday snapshot price → close predictability (RTH-INTRA-001 + RTH-INTRA-001b)

---

## Task A: Intraday Snapshot Predictability — All Days

**Scenario:**
We know gap direction and FH direction predict the afternoon. But what about a live price check mid-session — if NQ is above its RTH open at 12:00 ET, how often does it close above the open? And does it close above the 12:00 price itself (continuation) or below it (reversal from that point)? This builds a predictability curve through the day: at what time does intraday direction "lock in" and become a reliable close predictor? **(ID: RTH-INTRA-001)**

**Architecture:**
1. CTE 1 — join `rth_15min_buckets_agg` to `daily_ohlcv_rth` on `trade_date`:
   - `rth_open` from `daily_ohlcv_rth.open`
   - `rth_close` from `daily_ohlcv_rth.close`
   - `bucket_close` from `rth_15min_buckets_agg` (price at snapshot time T)
   - `above_open` = bucket_close > rth_open (direction at T relative to open)
   - `close_above_open` = rth_close > rth_open (day direction at close)
   - `close_above_snapshot` = rth_close > bucket_close (continuation from T to close)
2. Final SELECT — GROUP BY `bucket_start`, `above_open`:
   - `day_count`
   - `close_same_dir_pct` = % where close_above_open = above_open (close confirms direction seen at T)
   - `close_above_snapshot_pct` = % where rth_close > bucket_close (price continued higher from snapshot)

Filter to bucket_start times from 10:00 onwards (FH is already covered by RTH-FH-001 / RTH-ORB series).

**Expected output columns:**
`bucket_start, above_open, day_count, close_same_dir_pct, close_above_snapshot_pct`

**Finding to answer:** Does predictability increase monotonically as T approaches close? Is there a specific time (e.g. 13:00, 14:00) where direction "locks in" — meaning same-dir % jumps significantly? Is the signal stronger when price is above open vs below open (asymmetry)?

**Tables:** `nq_data.rth_15min_buckets_agg`, `nq_data.daily_ohlcv_rth`

**Difficulty Rating:** 3/5

**(ID: RTH-INTRA-001)**

WITH pre_final_agg AS (
SELECT 
	d.trade_date,
	TO_CHAR(r.bucket_start, 'HH:MI') AS bucket_window,
	d.weekday,
	r.bucket_start,
	r.bucket_close,
	d.OPEN AS rth_open,
	D.CLOSE AS rth_close,
	CASE WHEN r.bucket_close > d.OPEN THEN TRUE ELSE FALSE END AS above_open,
	CASE WHEN d.close > r.bucket_close THEN TRUE ELSE FALSE END AS close_above_snapshot,
	CASE WHEN d.CLOSE > d.OPEN THEN 'bullish' ELSE 'bearish' END AS day_direction
FROM nq_data.daily_ohlcv_rth d
JOIN nq_data.rth_15min_buckets_agg r ON d.trade_date = r.bucket_start::date
)
SELECT 
	bucket_window,
	above_open,
	COUNT(*),
	ROUND(COUNT(*) FILTER (WHERE above_open = close_above_snapshot) / COUNT(*)::NUMERIC * 100, 2) AS close_same_dir_pct,
	ROUND(COUNT(*) FILTER (WHERE close_above_snapshot IS True) / COUNT(*)::NUMERIC * 100, 2) AS close_above_snapshot_pct
FROM pre_final_agg
WHERE bucket_window >= '10:00'
GROUP BY bucket_window, above_open
ORDER BY bucket_window


bucket_window	above_open	count	close_same_dir_pct	close_above_snapshot_pct
10:00	false	84	48.81	51.19
10:00	true	102	60.78	60.78
10:15	false	82	41.46	58.54
10:15	true	104	57.69	57.69
10:30	false	85	47.06	52.94
10:30	true	101	61.39	61.39
10:45	false	88	42.05	57.95
10:45	true	98	52.04	52.04
11:00	false	90	44.44	55.56
11:00	true	96	64.58	64.58
11:15	false	89	44.94	55.06
11:15	true	97	56.7	56.7
11:30	false	89	39.33	60.67
11:30	true	97	54.64	54.64
11:45	false	84	45.24	54.76
11:45	true	102	50	50
12:00	false	89	46.07	53.93
12:00	true	97	50.52	50.52
12:15	false	92	42.39	57.61
12:15	true	94	48.94	48.94
12:30	false	92	45.65	54.35
12:30	true	94	52.13	52.13
12:45	false	90	46.67	53.33
12:45	true	96	52.08	52.08

---

## Task B: Add Weekday Dimension

**Scenario:**
Extension of Task A — does Monday "lock in" direction earlier than Tuesday? Does Wednesday's direction at 12:00 predict close better than Friday's? Some weekdays have structural stories that should show up here (Monday drifts higher all day; Thursday fades the open; Tuesday is mean-reverting). **(ID: RTH-INTRA-001b)**

**Architecture:**
Same as Task A, add `weekday` (from `daily_ohlcv_rth`) to GROUP BY. Focus findings commentary on:
- At what time does each weekday reach ~75%+ close_same_dir_pct?
- Any weekday where the curve is flat or non-monotonic (direction never locks in)?

Filter findings commentary to cells with N ≥ 5 per bucket_start × weekday × above_open combination — many cells will be small.

**Expected output columns:**
`bucket_start, weekday, above_open, day_count, close_same_dir_pct, close_above_snapshot_pct`

**Difficulty Rating:** 3/5

**(ID: RTH-INTRA-001b)**

WITH pre_final_agg AS (
SELECT 
	d.trade_date,
	TO_CHAR(r.bucket_start, 'HH:MI') AS bucket_window,
	d.weekday,
	r.bucket_start,
	r.bucket_close,
	d.OPEN AS rth_open,
	D.CLOSE AS rth_close,
	CASE WHEN r.bucket_close > d.OPEN THEN TRUE ELSE FALSE END AS above_open,
	CASE WHEN d.close > r.bucket_close THEN TRUE ELSE FALSE END AS close_above_snapshot,
	CASE WHEN d.CLOSE > d.OPEN THEN 'bullish' ELSE 'bearish' END AS day_direction
FROM nq_data.daily_ohlcv_rth d
JOIN nq_data.rth_15min_buckets_agg r ON d.trade_date = r.bucket_start::date
)
SELECT
	weekday,
	bucket_window,
	above_open,
	COUNT(*),
	ROUND(COUNT(*) FILTER (WHERE above_open = close_above_snapshot) / COUNT(*)::NUMERIC * 100, 2) AS close_same_dir_pct,
	ROUND(COUNT(*) FILTER (WHERE close_above_snapshot IS True) / COUNT(*)::NUMERIC * 100, 2) AS close_above_snapshot_pct
FROM pre_final_agg
WHERE bucket_window >= '10:00'
GROUP BY weekday, bucket_window, above_open
ORDER BY weekday, bucket_window


weekday	bucket_window	above_open	count	close_same_dir_pct	close_above_snapshot_pct
Friday	10:00	false	12	58.33	41.67
Friday	10:00	true	23	65.22	65.22
Friday	10:15	false	14	64.29	35.71
Friday	10:15	true	21	57.14	57.14
Friday	10:30	false	13	69.23	30.77
Friday	10:30	true	22	54.55	54.55
Friday	10:45	false	12	58.33	41.67
Friday	10:45	true	23	39.13	39.13
Friday	11:00	false	15	53.33	46.67
Friday	11:00	true	20	55	55
Friday	11:15	false	14	64.29	35.71
Friday	11:15	true	21	38.1	38.1
Friday	11:30	false	14	71.43	28.57
Friday	11:30	true	21	47.62	47.62
Friday	11:45	false	13	61.54	38.46
Friday	11:45	true	22	36.36	36.36
Friday	12:00	false	13	69.23	30.77
Friday	12:00	true	22	40.91	40.91
Friday	12:15	false	12	83.33	16.67
Friday	12:15	true	23	47.83	47.83
Friday	12:30	false	14	57.14	42.86
Friday	12:30	true	21	61.9	61.9
Friday	12:45	false	14	57.14	42.86
Friday	12:45	true	21	47.62	47.62
Monday	10:00	false	15	66.67	33.33
Monday	10:00	true	24	75	75
Monday	10:15	false	14	28.57	71.43
Monday	10:15	true	25	60	60
Monday	10:30	false	15	40	60
Monday	10:30	true	24	79.17	79.17
Monday	10:45	false	15	13.33	86.67
Monday	10:45	true	24	62.5	62.5
Monday	11:00	false	14	42.86	57.14
Monday	11:00	true	25	76	76
Monday	11:15	false	14	28.57	71.43
Monday	11:15	true	25	60	60
Monday	11:30	false	14	21.43	78.57
Monday	11:30	true	25	56	56
Monday	11:45	false	16	37.5	62.5
Monday	11:45	true	23	47.83	47.83
Monday	12:00	false	15	40	60
Monday	12:00	true	24	45.83	45.83
Monday	12:15	false	18	33.33	66.67
Monday	12:15	true	21	47.62	47.62
Monday	12:30	false	16	56.25	43.75
Monday	12:30	true	23	43.48	43.48
Monday	12:45	false	16	62.5	37.5
Monday	12:45	true	23	43.48	43.48
Thursday	10:00	false	27	37.04	62.96
Thursday	10:00	true	12	41.67	41.67
Thursday	10:15	false	23	39.13	60.87
Thursday	10:15	true	16	50	50
Thursday	10:30	false	26	46.15	53.85
Thursday	10:30	true	13	53.85	53.85
Thursday	10:45	false	27	40.74	59.26
Thursday	10:45	true	12	33.33	33.33
Thursday	11:00	false	28	32.14	67.86
Thursday	11:00	true	11	45.45	45.45
Thursday	11:15	false	28	39.29	60.71
Thursday	11:15	true	11	54.55	54.55
Thursday	11:30	false	26	34.62	65.38
Thursday	11:30	true	13	53.85	53.85
Thursday	11:45	false	25	40	60
Thursday	11:45	true	14	57.14	57.14
Thursday	12:00	false	26	38.46	61.54
Thursday	12:00	true	13	61.54	61.54
Thursday	12:15	false	28	32.14	67.86
Thursday	12:15	true	11	54.55	54.55
Thursday	12:30	false	27	37.04	62.96
Thursday	12:30	true	12	58.33	58.33
Thursday	12:45	false	27	44.44	55.56
Thursday	12:45	true	12	58.33	58.33
Tuesday	10:00	false	15	33.33	66.67
Tuesday	10:00	true	24	50	50
Tuesday	10:15	false	15	26.67	73.33
Tuesday	10:15	true	24	58.33	58.33
Tuesday	10:30	false	17	35.29	64.71
Tuesday	10:30	true	22	50	50
Tuesday	10:45	false	19	52.63	47.37
Tuesday	10:45	true	20	60	60
Tuesday	11:00	false	16	50	50
Tuesday	11:00	true	23	60.87	60.87
Tuesday	11:15	false	18	50	50
Tuesday	11:15	true	21	61.9	61.9
Tuesday	11:30	false	17	41.18	58.82
Tuesday	11:30	true	22	59.09	59.09
Tuesday	11:45	false	14	50	50
Tuesday	11:45	true	25	52	52
Tuesday	12:00	false	16	50	50
Tuesday	12:00	true	23	56.52	56.52
Tuesday	12:15	false	15	46.67	53.33
Tuesday	12:15	true	24	45.83	45.83
Tuesday	12:30	false	16	56.25	43.75
Tuesday	12:30	true	23	52.17	52.17
Tuesday	12:45	false	16	37.5	62.5
Tuesday	12:45	true	23	56.52	56.52
Wednesday	10:00	false	15	60	40
Wednesday	10:00	true	19	63.16	63.16
Wednesday	10:15	false	16	50	50
Wednesday	10:15	true	18	61.11	61.11
Wednesday	10:30	false	14	50	50
Wednesday	10:30	true	20	65	65
Wednesday	10:45	false	15	46.67	53.33
Wednesday	10:45	true	19	57.89	57.89
Wednesday	11:00	false	17	52.94	47.06
Wednesday	11:00	true	17	76.47	76.47
Wednesday	11:15	false	15	46.67	53.33
Wednesday	11:15	true	19	68.42	68.42
Wednesday	11:30	false	18	33.33	66.67
Wednesday	11:30	true	16	56.25	56.25
Wednesday	11:45	false	16	43.75	56.25
Wednesday	11:45	true	18	61.11	61.11
Wednesday	12:00	false	19	42.11	57.89
Wednesday	12:00	true	15	53.33	53.33
Wednesday	12:15	false	19	36.84	63.16
Wednesday	12:15	true	15	53.33	53.33
Wednesday	12:30	false	19	31.58	68.42
Wednesday	12:30	true	15	46.67	46.67
Wednesday	12:45	false	17	35.29	64.71
Wednesday	12:45	true	17	58.82	58.82

---

## Submission Instructions

Paste your queries and results. Log query IDs: RTH-INTRA-001, RTH-INTRA-001b.
