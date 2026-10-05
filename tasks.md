# PySpark Exercises — 2026-10-05 (Week 41, Day 1)

**Focus:** Window functions in the DataFrame API (`Window.partitionBy().orderBy()`)

---

## Window — Introduction

In SQL you write:
```sql
LAG(close) OVER (PARTITION BY trade_date ORDER BY timestamp)
```

In PySpark, the window spec is built as its own object first, then passed into a function call:
```python
from pyspark.sql.window import Window
from pyspark.sql import functions as F

w = Window.partitionBy("trade_date").orderBy("timestamp")

df.withColumn("prev_close", F.lag("close").over(w))
```

Breaking that down:
- `Window.partitionBy(...).orderBy(...)` — builds the window specification object (`w`). This is just a spec, not data — nothing is computed yet (same laziness as everything else).
- `F.lag("close")` — the window function itself, same concept as SQL's `LAG(close)`.
- `.over(w)` — this is the part with no SQL equivalent syntax-wise: you attach the window spec to the function call explicitly, rather than writing `OVER (...)` inline.
- `.withColumn("prev_close", ...)` — same as always, adds a new column, returns a new DataFrame.

Available window functions under `F`: `F.lag(col, offset=1)`, `F.lead(col, offset=1)`, `F.rank()`, `F.row_number()`, `F.first(col)`, `F.last(col)` — all familiar names from SQL.

---

## Exercise 1 — Previous Day's Close (LAG)

**Goal:** for the daily XAUUSD rollup you already built on Friday, add a column with the previous day's close — the direct PySpark equivalent of `LAG(close) OVER (ORDER BY trade_date)`.

**Steps:**
1. Start from your `aggregated_df` (daily OHLC) from Friday.
2. Build a window spec ordered by `trade_date` — no `partitionBy` needed here, since you want to look across ALL days, not within groups. (Can you build a `Window` with no `.partitionBy()` at all? Try it and see.)
3. Add a column `prev_close` using `F.lag("daily_close").over(w)`.
4. Add a column `close_change` = `daily_close - prev_close`.
5. `.show(10)`, ordered by `trade_date`.

**Think about it:** the very first row will have `prev_close` as `null` — same reason as in SQL (nothing precedes the first row). No fix needed, just notice it.


df = df.withColumn('trade_date', F.to_date('timestamp')).orderBy('timestamp')
aggregated_df = df.groupBy('trade_date').agg(
    F.first('open').alias('daily_open'),
    F.last('close').alias('daily_close'),
    F.min('low').alias('daily_low'),
    F.max('high').alias('daily_high'),
    F.sum('volume').alias('daily_volume')
).orderBy('trade_date')

w = Window.orderBy('trade_date')

aggregated_df = aggregated_df.withColumn('prev_close', F.lag('daily_close').over(w))
aggregated_df = aggregated_df.withColumn('close_change', F.round(aggregated_df['daily_close'] - aggregated_df['prev_close'], 2))
aggregated_df.show(10)

+----------+----------+-----------+---------+----------+------------------+----------+------------+
|trade_date|daily_open|daily_close|daily_low|daily_high|      daily_volume|prev_close|close_change|
+----------+----------+-----------+---------+----------+------------------+----------+------------+
|2024-01-10|  2030.674|   2026.695| 2020.395|  2040.135|25.370000000000093|      NULL|        NULL|
|2024-01-11|  2026.735|   2035.125| 2013.225|  2043.864| 38.21999999999991|  2026.695|        8.43|
|2024-01-12|  2035.055|   2048.675| 2029.975|  2062.145| 31.62000000000005|  2035.125|       13.55|
|2024-01-14|  2048.498|   2047.505| 2046.235|  2048.605|0.5300000000000002|  2048.675|       -1.17|
|2024-01-15|  2047.525|   2053.895| 2045.675|  2058.505| 17.60999999999989|  2047.505|        6.39|
|2024-01-16|  2054.025|   2028.045| 2024.165|  2054.435| 36.02999999999978|  2053.895|      -25.85|
|2024-01-17|  2028.055|   2009.465| 2001.755|  2032.815| 34.69999999999991|  2028.045|      -18.58|
|2024-01-18|  2009.505|   2023.675| 2005.705|  2024.555| 23.63000000000021|  2009.465|       14.21|
|2024-01-19|  2023.675|   2029.375| 2020.325|  2039.278|25.110000000000205|  2023.675|         5.7|
|2024-01-21|  2029.715|   2027.535| 2026.848|  2029.715|0.3800000000000001|  2029.375|       -1.84|
+----------+----------+-----------+---------+----------+------------------+----------+------------+


---

## Exercise 2 — Rank Days by Volume (RANK)

**Goal:** direct equivalent of `RANK() OVER (ORDER BY daily_volume DESC)`.

**Steps:**
1. Build a window spec ordered by `daily_volume` descending (`.orderBy(F.col("daily_volume").desc())`).
2. Add a column `volume_rank` using `F.rank().over(w)`.
3. Filter to `volume_rank <= 5` and `.show()` — the 5 highest-volume trading days.



w = Window.orderBy(F.col('daily_volume').desc())
ordered_vol_daily_rank = aggregated_df.withColumn('rank', F.rank().over(w))
ordered_vol_daily_rank = ordered_vol_daily_rank.filter(ordered_vol_daily_rank.rank <= 5)
ordered_vol_daily_rank.show()

spark.stop()


+----------+----------+-----------+---------+----------+------------------+----------+------------+----+
|trade_date|daily_open|daily_close|daily_low|daily_high|      daily_volume|prev_close|close_change|rank|
+----------+----------+-----------+---------+----------+------------------+----------+------------+----+
|2025-08-08|  3400.119|   3398.579| 3373.288|  3404.213|167.36000000000044|  3400.109|       -1.53|   1|
|2024-08-05|  2440.865|   2404.675| 2364.164|  2458.655|121.34000000000005|  2440.865|      -36.19|   2|
|2024-11-06|  2745.485|   2663.875| 2652.295|  2749.605|113.61000000000024|  2745.495|      -81.62|   3|
|2024-12-18|  2647.165|   2587.948| 2582.264|  2651.525|113.30000000000021|  2647.198|      -59.25|   4|
|2024-12-11|  2695.835|   2719.835| 2674.345|  2721.005|109.25000000000044|  2695.798|       24.04|   5|
+----------+----------+-----------+---------+----------+------------------+----------+------------+----+

---

## Submission Instructions

Paste your code and output below, for each exercise.
