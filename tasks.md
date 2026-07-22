# NQ Project — Week 31 Day 3

**Generated:** 2026-07-22
**Focus:** Snapshot predictability × OR direction (RTH-INTRA-003) + FH extreme location × close direction (RTH-FH-008)

---

## Task A: Snapshot Predictability × OR Direction
**(ID: RTH-INTRA-003)**

**Business question:**
RTH-INTRA-002 split snapshot predictability by gap direction (overnight context). This task replaces gap_direction with OR direction — does the first 30-minute price action (09:30–10:00) override the overnight context as a predictor? OR direction is known at 10:00 and is the earliest intraday signal available. The question: does bullish OR (close > open of the OR period) sharpen or contradict the snapshot signal for the rest of the session?

**Architecture:**
Same CTE structure as RTH-INTRA-002. Replace `gap_direction` with `or_direction`:

```sql
CASE WHEN or_close > or_open THEN 'or_bullish' ELSE 'or_bearish' END AS or_direction
```

Where `or_open` = first bucket_close at 09:30 (or use `d.open` as RTH open proxy) and `or_close` = bucket_close at bucket_window = '10:00'. You'll need to pull the 10:00 bucket_close per trade_date into the CTE — do this with a subquery or second JOIN to `rth_15min_buckets_agg` filtered to bucket_start::time = '09:30' for or_open and '10:00' for or_close.

Keep `bucket_window >= '10:00'` filter (OR window is closed by then). Keep `HAVING COUNT(*) >= 5`. Keep `above_open` as the snapshot state. Group by `weekday, or_direction, bucket_window, above_open`.

**Key questions to answer from results:**
- Does or_bullish + above_open give stronger confirmation than gap_up + above_open did?
- Does or_bearish + above_open flip to a fade (like Tuesday gap_up did)?
- Which weekdays show the biggest split between or_bullish and or_bearish cells?

**Expected output columns:**
`weekday, or_direction, bucket_window, above_open, count, close_same_dir_pct, close_above_snapshot_pct`

**Difficulty Rating:** 4/5


WITH pre_final_agg AS (
SELECT 
	rf.trade_date,
	TO_CHAR(r.bucket_start, 'HH24:MI') AS bucket_window,
	TO_CHAR(rf.trade_date, 'Day') AS weekday,
	r.bucket_start,
	r.bucket_close,
	rf.or_open AS or_open,
	rf.or_close AS or_close,
	rf.r_close AS rth_close,
	CASE WHEN r.bucket_close > rf.or_open THEN TRUE ELSE FALSE END AS above_open,
	CASE WHEN rf.r_close > r.bucket_close THEN TRUE ELSE FALSE END AS close_above_snapshot,
	CASE WHEN rf.r_close > rf.or_open THEN 'bullish' ELSE 'bearish' END AS day_direction,
	CASE WHEN rf.or_close > rf.or_open THEN 'bullish' ELSE 'bearish' END AS or_direction,
	CASE WHEN rf.or_open > LAG(rf.r_close) OVER (ORDER BY rf.trade_date) THEN 'gap_up' ELSE 'gap_down' END AS gap_direction
FROM nq_data.or_rest_ohlc_ranges rf
JOIN nq_data.rth_15min_buckets_agg r ON rf.trade_date = r.bucket_start::date
)
SELECT 
	weekday,
	or_direction,
	bucket_window,
	above_open,
	COUNT(*) AS days,
	ROUND(COUNT(*) FILTER (WHERE close_above_snapshot IS TRUE) / COUNT(*)::NUMERIC * 100, 2) AS close_above_snapshot_pct
FROM pre_final_agg
GROUP BY weekday, or_direction, bucket_window, above_open
HAVING COUNT(*) > 5


I've found an issue here and in RTH-INTRA 2.
Aren't we really checking CLOSES ABOVE, instead of both directions truly?
That could give biased info that's simply wrong and we need to be VERY CAREFUL WITH THIS APPROACH.

We could obviously still get useful info if we know that X close above, so the rest close below, it's obvious, BUT IT NEEDS TO BE PROPERLY NOTED AND CONVEYED, so there are no misunderstandings

It only makes sense to keep one aggregation in this sense - close_above_snapshot_pct, as it give us an obvious info about the rest that are closed BELOW the snapshot, which is obvious.


weekday	or_direction	bucket_window	above_open	days	close_above_snapshot_pct
Thursday 	bearish	11:30	false	16	62.5
Thursday 	bullish	15:45	true	6	0
Thursday 	bearish	10:00	false	19	68.42
Tuesday  	bullish	13:00	true	13	61.54
Wednesday	bullish	13:30	true	12	58.33
Monday   	bullish	10:45	true	18	61.11
Monday   	bearish	11:30	false	9	66.67
Wednesday	bearish	15:00	false	11	63.64
Wednesday	bullish	11:30	true	13	53.85
Tuesday  	bearish	12:45	true	7	42.86
Tuesday  	bullish	10:00	true	18	50
Wednesday	bearish	11:00	false	12	50
Monday   	bearish	15:00	false	8	75
Tuesday  	bullish	15:30	false	6	50
Tuesday  	bearish	14:15	false	6	66.67
Thursday 	bullish	14:30	false	6	50
Monday   	bearish	15:15	false	8	62.5
Friday   	bullish	15:00	true	13	53.85
Thursday 	bearish	15:15	false	14	64.29
Tuesday  	bearish	13:15	true	6	50
Monday   	bearish	10:30	false	10	50
Tuesday  	bullish	15:30	true	13	23.08
Tuesday  	bearish	15:15	false	7	57.14
Tuesday  	bearish	11:30	false	7	57.14
Wednesday	bullish	10:15	true	15	60
Friday   	bullish	10:15	true	16	56.25
Friday   	bullish	13:30	true	13	61.54
Thursday 	bullish	10:00	true	7	42.86
Wednesday	bearish	11:15	false	11	54.55
Thursday 	bearish	11:15	false	17	58.82
Tuesday  	bullish	13:45	true	13	53.85
Tuesday  	bearish	14:15	true	8	25
Tuesday  	bearish	14:00	true	7	14.29
Thursday 	bearish	11:45	false	16	56.25
Thursday 	bullish	11:00	true	7	42.86
Tuesday  	bearish	12:45	false	7	85.71
Tuesday  	bullish	12:00	false	7	57.14
Wednesday	bullish	12:15	false	6	66.67
Wednesday	bearish	12:45	false	11	63.64
Wednesday	bearish	14:30	false	11	54.55
Tuesday  	bearish	13:45	true	6	16.67
Wednesday	bullish	12:30	true	11	45.45
Wednesday	bullish	12:00	true	12	50
Thursday 	bearish	11:00	false	18	66.67
Tuesday  	bullish	14:45	true	13	53.85
Thursday 	bullish	12:15	true	6	66.67
Friday   	bearish	13:45	false	8	37.5
Tuesday  	bearish	13:00	false	7	57.14
Friday   	bullish	11:30	true	13	53.85
Monday   	bearish	12:00	false	9	44.44
Tuesday  	bearish	12:30	true	7	28.57
Monday   	bullish	10:30	true	20	75
Thursday 	bearish	14:15	false	16	43.75
Tuesday  	bullish	11:15	false	7	57.14
Monday   	bearish	14:45	false	9	66.67
Friday   	bearish	13:30	false	8	37.5
Monday   	bullish	15:00	true	18	33.33
Wednesday	bullish	15:45	true	13	0
Wednesday	bullish	10:00	true	16	62.5
Wednesday	bearish	10:00	false	12	41.67
Tuesday  	bearish	11:30	true	7	42.86
Friday   	bearish	10:00	false	8	37.5
Friday   	bearish	09:45	false	11	36.36
Thursday 	bearish	15:00	false	14	50
Wednesday	bullish	12:45	true	12	50
Tuesday  	bullish	14:30	true	13	53.85
Thursday 	bearish	14:00	false	15	60
Friday   	bullish	15:15	true	12	50
Thursday 	bullish	09:30	true	7	42.86
Wednesday	bullish	14:45	true	12	58.33
Tuesday  	bullish	12:45	false	6	66.67
Monday   	bearish	11:00	false	9	66.67
Monday   	bearish	10:45	false	9	77.78
Tuesday  	bullish	13:15	true	13	61.54
Wednesday	bullish	12:15	true	12	50
Thursday 	bullish	13:30	false	6	50
Wednesday	bearish	12:30	false	11	63.64
Tuesday  	bearish	13:00	true	7	42.86
Monday   	bearish	15:30	false	9	55.56
Monday   	bearish	13:30	false	9	22.22
Thursday 	bearish	12:45	false	16	50
Friday   	bullish	14:15	true	13	46.15
Tuesday  	bearish	13:30	false	8	87.5
Friday   	bearish	15:15	false	8	25
Tuesday  	bearish	12:15	false	6	83.33
Wednesday	bullish	15:30	true	12	58.33
Thursday 	bullish	15:30	false	6	66.67
Wednesday	bullish	14:30	false	7	71.43
Monday   	bearish	14:15	false	8	62.5
Tuesday  	bearish	13:45	false	8	50
Tuesday  	bearish	11:00	true	6	66.67
Thursday 	bearish	10:45	false	19	63.16
Tuesday  	bullish	14:00	true	13	53.85
Thursday 	bullish	11:15	true	6	50
Monday   	bearish	14:00	false	8	50
Wednesday	bullish	13:00	true	12	50
Thursday 	bullish	09:45	true	10	40
Wednesday	bullish	13:15	false	6	50
Wednesday	bullish	11:15	true	14	64.29
Monday   	bearish	09:30	false	10	30
Tuesday  	bearish	10:45	false	11	54.55
Wednesday	bearish	10:15	false	12	58.33
Thursday 	bearish	09:45	false	21	57.14
Thursday 	bullish	12:00	true	7	71.43
Monday   	bullish	10:00	true	20	75
Tuesday  	bullish	10:45	true	13	53.85
Friday   	bullish	09:45	true	18	61.11
Tuesday  	bullish	09:45	true	19	52.63
Friday   	bearish	11:30	false	7	28.57
Tuesday  	bearish	11:00	false	8	62.5
Wednesday	bullish	13:15	true	11	54.55
Tuesday  	bullish	12:15	false	6	50
Friday   	bearish	10:15	false	10	40
Wednesday	bullish	15:00	true	12	58.33
Tuesday  	bearish	11:15	false	9	55.56
Tuesday  	bearish	12:00	false	6	66.67
Tuesday  	bearish	12:30	false	7	71.43
Wednesday	bullish	10:45	true	15	53.33
Thursday 	bearish	15:30	false	14	57.14
Monday   	bullish	11:15	true	19	57.89
Thursday 	bearish	13:00	false	15	60
Wednesday	bearish	14:15	false	11	63.64
Tuesday  	bullish	14:30	false	6	33.33
Monday   	bearish	12:30	false	10	40
Tuesday  	bearish	15:15	true	7	42.86
Thursday 	bullish	14:15	false	6	50
Thursday 	bearish	14:45	false	16	56.25
Friday   	bearish	14:00	false	8	25
Wednesday	bullish	11:00	true	14	71.43
Monday   	bullish	12:00	true	18	44.44
Tuesday  	bullish	15:00	true	13	61.54
Wednesday	bullish	15:15	true	12	41.67
Monday   	bullish	12:15	true	17	47.06
Tuesday  	bearish	10:30	false	12	66.67
Thursday 	bearish	15:00	true	6	66.67
Thursday 	bullish	11:30	true	6	66.67
Friday   	bearish	14:45	false	7	14.29
Friday   	bearish	14:30	false	8	12.5
Monday   	bearish	12:15	false	10	50
Tuesday  	bullish	12:30	false	6	33.33
Wednesday	bearish	15:30	false	11	54.55
Monday   	bullish	15:45	true	17	0
Wednesday	bullish	12:45	false	6	66.67
Monday   	bullish	12:30	true	18	44.44
Wednesday	bullish	10:30	true	16	62.5
Thursday 	bullish	10:15	true	10	50
Friday   	bearish	11:15	false	8	25
Friday   	bearish	13:00	false	8	37.5
Wednesday	bearish	09:30	false	13	30.77
Wednesday	bullish	14:30	true	10	50
Wednesday	bullish	14:15	true	11	54.55
Tuesday  	bullish	13:45	false	6	33.33
Tuesday  	bearish	12:00	true	8	37.5
Monday   	bullish	14:00	true	17	29.41
Friday   	bearish	13:15	false	8	37.5
Tuesday  	bearish	13:15	false	8	75
Tuesday  	bullish	12:15	true	13	46.15
Tuesday  	bearish	09:30	false	11	63.64
Friday   	bearish	10:30	false	8	37.5
Friday   	bearish	10:45	false	7	42.86
Monday   	bearish	13:45	false	8	37.5
Thursday 	bearish	12:30	false	16	56.25
Tuesday  	bullish	11:00	false	6	50
Thursday 	bearish	15:15	true	6	66.67
Tuesday  	bullish	12:00	true	12	66.67
Monday   	bearish	10:00	false	11	36.36
Monday   	bullish	11:45	true	17	47.06
Monday   	bearish	14:30	false	8	62.5
Monday   	bullish	14:15	true	18	44.44
Friday   	bullish	13:15	true	13	53.85
Wednesday	bullish	13:45	true	12	58.33
Tuesday  	bullish	10:45	false	6	50
Monday   	bearish	13:00	false	9	11.11
Friday   	bullish	14:45	true	13	38.46
Tuesday  	bearish	12:15	true	8	37.5
Monday   	bullish	14:30	true	18	44.44
Friday   	bullish	12:00	true	13	53.85
Tuesday  	bearish	14:30	true	8	25
Thursday 	bullish	15:00	false	6	66.67
Tuesday  	bullish	15:45	false	6	0
Friday   	bullish	15:30	true	12	50
Friday   	bullish	11:45	true	13	46.15
Monday   	bearish	12:45	false	10	30
Wednesday	bearish	13:30	false	11	72.73
Wednesday	bearish	15:15	false	10	60
Monday   	bullish	12:45	true	18	38.89
Monday   	bearish	15:45	false	9	0
Wednesday	bearish	09:45	false	15	46.67
Thursday 	bullish	14:45	false	6	66.67
Tuesday  	bullish	10:15	true	16	62.5
Friday   	bearish	15:00	false	8	25
Tuesday  	bullish	11:00	true	13	53.85
Wednesday	bearish	10:45	false	11	54.55
Monday   	bullish	10:15	true	20	60
Friday   	bearish	14:15	false	8	25
Wednesday	bearish	11:45	false	11	54.55
Monday   	bullish	15:30	true	18	44.44
Wednesday	bearish	14:45	false	11	63.64
Monday   	bullish	15:15	true	17	47.06
Thursday 	bearish	15:30	true	6	66.67
Thursday 	bullish	11:45	true	7	71.43
Friday   	bearish	15:30	false	7	28.57
Tuesday  	bullish	13:30	true	13	61.54
Thursday 	bullish	15:15	false	6	66.67
Wednesday	bullish	09:30	false	6	83.33
Friday   	bullish	13:45	true	13	46.15
Thursday 	bearish	10:30	false	19	52.63
Thursday 	bearish	14:30	false	16	56.25
Wednesday	bearish	13:00	false	11	63.64
Monday   	bullish	13:30	true	18	55.56
Friday   	bearish	12:30	false	8	37.5
Tuesday  	bearish	15:45	false	8	0
Wednesday	bullish	11:45	true	13	53.85
Friday   	bearish	11:00	false	9	44.44
Friday   	bearish	12:45	false	8	37.5
Thursday 	bearish	13:30	false	15	53.33
Friday   	bullish	12:15	true	14	64.29
Wednesday	bullish	09:45	true	18	61.11
Friday   	bullish	14:30	true	13	53.85
Thursday 	bearish	10:15	false	18	61.11
Tuesday  	bearish	15:30	false	7	57.14
Tuesday  	bullish	11:30	true	11	63.64
Friday   	bearish	12:00	false	7	28.57
Tuesday  	bearish	14:00	false	7	71.43
Friday   	bearish	12:15	false	7	14.29
Wednesday	bearish	13:45	false	12	58.33
Tuesday  	bullish	13:00	false	6	33.33
Tuesday  	bearish	11:45	false	6	50
Tuesday  	bearish	15:00	false	6	66.67
Tuesday  	bullish	15:45	true	13	0
Friday   	bullish	10:00	true	16	56.25
Thursday 	bullish	13:00	false	7	42.86
Wednesday	bearish	13:15	false	12	66.67
Tuesday  	bullish	11:45	true	13	61.54
Friday   	bullish	11:00	true	14	64.29
Tuesday  	bullish	09:30	true	14	64.29
Monday   	bearish	13:15	false	8	25
Tuesday  	bearish	13:30	true	6	16.67
Tuesday  	bullish	12:30	true	13	61.54
Friday   	bullish	10:45	true	14	50
Wednesday	bullish	12:30	false	7	71.43
Tuesday  	bullish	11:15	true	12	58.33
Tuesday  	bearish	15:30	true	7	42.86
Wednesday	bullish	12:00	false	6	50
Monday   	bearish	11:45	false	9	44.44
Wednesday	bullish	13:00	false	6	50
Tuesday  	bullish	11:45	false	6	66.67
Monday   	bullish	09:30	true	17	70.59
Thursday 	bearish	13:15	false	16	56.25
Tuesday  	bearish	10:15	false	12	75
Friday   	bullish	13:00	true	14	57.14
Friday   	bullish	14:00	true	13	46.15
Tuesday  	bullish	15:15	true	14	50
Thursday 	bullish	13:15	false	6	50
Thursday 	bullish	12:30	true	6	66.67
Friday   	bearish	09:30	false	8	37.5
Friday   	bullish	12:30	true	14	71.43
Friday   	bullish	11:15	true	13	46.15
Tuesday  	bearish	15:45	true	6	0
Wednesday	bearish	15:45	false	11	0
Tuesday  	bullish	14:45	false	6	16.67
Tuesday  	bullish	13:15	false	6	50
Monday   	bearish	11:15	false	9	55.56
Thursday 	bullish	10:30	true	9	44.44
Friday   	bearish	15:45	false	7	0
Wednesday	bearish	14:00	false	11	54.55
Thursday 	bearish	15:45	false	15	0
Wednesday	bullish	14:15	false	6	83.33
Friday   	bullish	15:45	true	12	0
Monday   	bearish	09:45	false	11	27.27
Tuesday  	bullish	11:30	false	8	75
Thursday 	bearish	12:15	false	17	64.71
Monday   	bullish	09:45	true	22	72.73
Thursday 	bullish	14:00	false	6	66.67
Tuesday  	bullish	14:15	true	14	57.14
Tuesday  	bullish	13:30	false	6	33.33
Monday   	bullish	11:30	true	19	52.63
Monday   	bullish	11:00	true	19	78.95
Monday   	bearish	10:15	false	10	70
Monday   	bullish	13:15	true	16	50
Tuesday  	bullish	15:00	false	6	33.33
Wednesday	bearish	11:30	false	12	66.67
Thursday 	bearish	09:30	false	18	66.67
Tuesday  	bearish	09:45	false	14	71.43
Tuesday  	bearish	14:30	false	6	66.67
Wednesday	bullish	14:00	false	6	83.33
Tuesday  	bearish	15:00	true	8	25
Monday   	bullish	13:00	true	15	40
Friday   	bearish	11:45	false	7	28.57
Monday   	bullish	13:45	true	15	46.67
Friday   	bullish	10:30	true	15	46.67
Friday   	bullish	09:30	true	16	68.75
Thursday 	bullish	10:45	true	8	37.5
Monday   	bullish	14:45	true	18	50
Thursday 	bearish	12:00	false	17	58.82
Wednesday	bullish	14:00	true	11	45.45
Tuesday  	bullish	12:45	true	13	61.54
Tuesday  	bearish	11:45	true	8	37.5
Wednesday	bullish	09:30	true	12	58.33
Tuesday  	bullish	14:00	false	6	33.33
Tuesday  	bearish	10:00	false	14	64.29
Wednesday	bearish	12:15	false	12	58.33
Friday   	bullish	12:45	true	14	57.14
Wednesday	bearish	12:00	false	12	58.33
Wednesday	bearish	10:30	false	11	54.55
Tuesday  	bullish	10:30	true	14	50
Tuesday  	bearish	14:45	true	9	22.22
Thursday 	bearish	13:45	false	16	56.25



---

## Task B: First Hour Extreme Location × Close Direction
**(ID: RTH-FH-008)**

**Business question:**
Where within the full day range does the first hour set its extreme — and does that predict close direction? A FH high in the top 20% of the day range is a very different situation than a FH high in the middle. This extends the FH series into a new dimension: not just whether FH sets HOD/LOD (RTH-FH-001/004), but *where* in the day range the FH extreme sits and whether close location correlates.

**Architecture:**
Join `rth_firsthour_rest_ohlc_ranges` (for fh_high, fh_low, fh_open, fh_close) to `daily_ohlcv_rth` (for full day high, low, open, close) on trade_date.

Compute per day:
- `fh_high_location` = (fh_high - daily_low) / NULLIF(daily_high - daily_low, 0) — where in the full day range is the FH high (0 = bottom, 1 = top)
- `fh_low_location` = (fh_low - daily_low) / NULLIF(daily_high - daily_low, 0)
- `fh_direction` = CASE WHEN fh_close > fh_open THEN 'bullish' ELSE 'bearish' END
- `close_location` = (daily_close - daily_low) / NULLIF(daily_high - daily_low, 0)
- `close_above_open` = daily_close > daily_open

Bucket `fh_high_location` into thirds (NTILE(3)) — low third (FH high in bottom of day range), mid third, top third.

Aggregate by `fh_direction × fh_high_location_bucket × weekday`: avg close_location, pct close_above_open, count.

**Key questions:**
- When FH sets its high in the top third of the day range (FH high ≈ HOD), how often does close end up bullish vs bearish?
- When FH high is in the bottom third (early low then big rally), what does close look like?
- Does weekday interact with this pattern?

**Expected output columns:**
`weekday, fh_direction, fh_high_location_bucket, count, avg_fh_high_location, avg_close_location, pct_close_above_open`

**Difficulty Rating:** 4/5


WITH daily_fh_rest_agg AS (
SELECT 
	*,
	(fh_high - low) / NULLIF(high - low, 0) AS fh_high_location,
	(fh_low - low) / NULLIF(high - low, 0) AS fh_low_location,
	CASE WHEN fh_close > fh_open THEN 'bullish' ELSE 'bearish' END AS fh_direction,
	CLOSE - low / NULLIF(high - low, 0) AS close_location,
	CLOSE > OPEN AS close_above_open
FROM nq_data.rth_firsthour_rest_ohlc_ranges r
JOIN nq_data.daily_ohlcv_rth d ON r.trade_date = d.trade_date
),
fh_agg_buckets AS (
SELECT 
	*,
	ntile(3) OVER (ORDER BY fh_high_location) AS fh_high_location_bucket
FROM daily_fh_rest_agg
)
SELECT 
	fh_direction,
	fh_high_location_bucket,
	weekday,
	COUNT(*),
	ROUND(AVG(close_location), 2) AS avg_close_location,
	ROUND(COUNT(*) FILTER (WHERE close_above_open IS TRUE) / COUNT(*)::NUMERIC * 100, 2) AS close_above_open_pct
FROM fh_agg_buckets
GROUP BY fh_direction, fh_high_location_bucket, weekday


We could also check situations where FH sets high/low as it's 0/1 in the high/low location if I'm not mistaken. That could be done tomorrow.

fh_direction	fh_high_location_bucket	weekday	count	avg_close_location	close_above_open_pct
bullish	2	Friday	8	25,543.46	87.5
bearish	1	Monday	1	24,959.62	100
bullish	1	Wednesday	10	25,557.88	100
bullish	1	Monday	15	25,189.07	100
bullish	2	Thursday	6	24,900.4	50
bearish	3	Thursday	14	25,244.05	0
bullish	1	Friday	10	25,289.58	90
bearish	2	Friday	6	25,315.1	33.33
bearish	1	Thursday	2	25,780.98	100
bearish	2	Thursday	7	25,929.44	42.86
bearish	3	Monday	8	25,147.11	0
bullish	2	Monday	7	25,141.62	71.43
bearish	2	Tuesday	5	24,820.16	60
bearish	3	Wednesday	8	24,953.98	0
bearish	2	Monday	5	25,437.07	40
bullish	2	Tuesday	8	25,351.42	62.5
bearish	1	Friday	2	25,815.16	100
bullish	3	Monday	3	24,605.94	33.33
bullish	1	Tuesday	9	25,340.58	88.89
bearish	3	Friday	6	24,951.22	0
bullish	3	Tuesday	7	26,056.78	28.57
bearish	1	Wednesday	3	25,547.74	100
bullish	3	Friday	3	24,776.6	33.33
bearish	2	Wednesday	5	25,138.94	0
bullish	2	Wednesday	5	25,461.9	80
bullish	3	Wednesday	3	25,043.9	66.67
bullish	3	Thursday	5	24,548.24	60
bearish	1	Tuesday	5	25,134.75	80
bearish	3	Tuesday	5	24,808.83	0
bullish	1	Thursday	5	24,547.75	80

---

## Submission Instructions

Paste your queries and results. Log query IDs: RTH-INTRA-003, RTH-FH-008.
