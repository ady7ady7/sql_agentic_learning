# SQL Tasks — 2026-09-29 (Week 40, Day 2)

**Dataset:** crappy_data_db  
**Focus:** Gaps-and-islands (monthly granularity)

---

## Task 1 — Monthly Order Streaks per User

**Difficulty: 4/5**

**Business question:**  
For each user, identify consecutive-month streaks of order activity — a "streak" is a run of consecutive calendar months where the user placed at least one order. If a user is active in Jan, Feb, Mar, then skips Apr, then is active again in May, Jun — that's two separate streaks (Jan-Mar, May-Jun).

For each streak, show its length in months.

Only include users with at least one streak of length >= 2.

**Expected output columns:**  
`user_id, streak_start_month, streak_end_month, streak_length`

Order by `user_id`, `streak_start_month`.

WITH users_transactions AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_,
	LAG(DATE_TRUNC('Month', created_at)) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_transaction_month
FROM crappy_data_db.orders o
),
users_streak_keys AS (
SELECT 
	*,
	month_ - prev_transaction_month AS diff,
	CASE WHEN prev_transaction_month IS NULL OR (month_ - prev_transaction_month > INTERVAL '31 Day') THEN 1 ELSE 0 END AS is_new_streak
FROM users_transactions
),
users_streak_ids AS (
SELECT 
	*,
	SUM(is_new_streak) OVER (PARTITION BY user_id ORDER BY created_at) AS streak_id
FROM users_streak_keys
)
SELECT 
	user_id,
	MIN(month_) AS streak_start_month,
	MAX(month_) AS streak_end_month,
	COUNT(DISTINCT(month_)) AS streak_length
FROM users_streak_ids
GROUP BY user_id, streak_id
HAVING COUNT(DISTINCT(month_)) >= 2


---

## Submission Instructions

Paste your query below.

---

# PySpark Exercise — Daily OHLC Rollup from M1 Candles

**Goal:** translate a familiar SQL aggregation pattern into the DataFrame API, in `spark_intro/practice.py`.

**Task:** using one of the M1 Parquet files (e.g. `usatechidxusd_m1_tradfi_ohlcv.parquet`), compute daily OHLC bars:
- `open` = the price at the first minute of the day
- `high` = max price of the day
- `low` = min price of the day
- `close` = the price at the last minute of the day
- `volume` = sum of the day's volume

Think about which DataFrame API pieces you need — grouping, aggregation functions, and something equivalent to `FIRST_VALUE`/`LAST_VALUE` with ordering (hint: PySpark's `first()` and `last()` aggregate functions take an `ignorenulls` flag but need the data pre-sorted, or you may want a window function like in SQL — PySpark has `Window` and `F.first()`/`F.last()` over it, very close to what you already know).

Compare the result against what you'd expect from the equivalent SQL query on the same data, as a sanity check.



from pyspark.sql import SparkSession
import pandas as pd

spark = (
    SparkSession.builder
    .appName("adrian_spark")
    .master("local[*]")
    .getOrCreate()
)


df = spark.read.parquet('./data/xauusd_m1_tradfi_ohlcv.parquet')
df.show(5)

df['trade_date'] = pd.to_datetime(df['timestamp'])

grouped_days = df.groupby('trade_date').agg(
    open = ('open', 'first')
)

grouped_days.show(5)



spark.stop()