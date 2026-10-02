# PySpark Exercises — 2026-10-02 (Week 40, Day 5)

**Focus:** `F.first()` / `F.last()` with ordering · closing the original daily-OHLC goal · writing output back to Parquet

---

## Exercise 1 — First and Last Price of the Day

**Goal:** get `open` (price at the first minute of the day) and `close` (price at the last minute of the day) inside a `groupBy().agg()`.

**The problem:** `F.first("open")` / `F.last("close")` only give you a deterministic answer if the rows within each group are sorted first — otherwise Spark may pick an arbitrary row. You need to sort the DataFrame by `timestamp` BEFORE grouping, so that "first" and "last" actually mean "earliest" and "latest" in time.

**Steps:**
1. Add a `trade_date` column to `df` using `F.to_date("timestamp")` (new function — look up its signature, it's a one-argument call like `F.min`/`F.max`), via `.withColumn("trade_date", F.to_date("timestamp"))`. `.withColumn()` is how you add/replace a single column in PySpark — think of it as the DataFrame-API equivalent of adding an expression to a SELECT list. Remember: like everything else, it returns a NEW DataFrame — save it.
2. Sort that DataFrame by `timestamp` ascending (`.orderBy("timestamp")`).
3. Group by `trade_date`, and in `.agg()` compute:
   - `F.first("open").alias("day_open")`
   - `F.last("close").alias("day_close")`
   - `F.min("low").alias("day_low")`
   - `F.max("high").alias("day_high")`
   - `F.sum("volume").alias("day_volume")`
4. Order the final result by `trade_date`.
5. `.show(10)` it.

**Sanity check:** pick one date from the output and sanity-check `day_open`/`day_close` against what you'd expect — does `day_open` look like a 00:00-ish price and `day_close` like a 23:59-ish price for that date? (Remember Sunday is thin — maybe pick a Tuesday.)




df = df.withColumn('trade_date', F.to_date('timestamp')).orderBy('timestamp')
aggregated_df = df.groupBy('trade_date').agg(
    F.first('open').alias('daily_open'),
    F.last('close').alias('daily_close'),
    F.min('low').alias('daily_low'),
    F.max('high').alias('daily_high'),
    F.sum('volume').alias('daily_volume')
).orderBy('trade_date')
aggregated_df.show(10)


+----------+----------+-----------+---------+----------+------------------+     
|trade_date|daily_open|daily_close|daily_low|daily_high|      daily_volume|
+----------+----------+-----------+---------+----------+------------------+
|2024-01-10|  2030.674|   2026.695| 2020.395|  2040.135|25.370000000000093|
|2024-01-11|  2026.735|   2035.125| 2013.225|  2043.864| 38.21999999999991|
|2024-01-12|  2035.055|   2048.675| 2029.975|  2062.145| 31.62000000000005|
|2024-01-14|  2048.498|   2047.505| 2046.235|  2048.605|0.5300000000000002|
|2024-01-15|  2047.525|   2053.895| 2045.675|  2058.505| 17.60999999999989|
|2024-01-16|  2054.025|   2028.045| 2024.165|  2054.435| 36.02999999999978|
|2024-01-17|  2028.055|   2009.465| 2001.755|  2032.815| 34.69999999999991|
|2024-01-18|  2009.505|   2023.675| 2005.705|  2024.555| 23.63000000000021|
|2024-01-19|  2023.675|   2029.375| 2020.325|  2039.278|25.110000000000205|
|2024-01-21|  2029.715|   2027.535| 2026.848|  2029.715|0.3800000000000001|
+----------+----------+-----------+---------+----------+------------------+


Yeah, it looks alright

---

## Exercise 2 — Write the Result to Parquet

**Goal:** close the full cycle — read → transform → write — which is the shape of every real PySpark job.

**Steps:**
1. Take the daily OHLC DataFrame from Exercise 1.
2. Write it to a new file with `.write.mode("overwrite").parquet("./data/xauusd_daily_ohlc.parquet")`.
3. In a separate block (or just after), read it back with `spark.read.parquet(...)` into a new variable, and `.show(5)` it — to confirm the round-trip actually worked.

**Note on `.mode("overwrite")`:** by default, `.write.parquet()` refuses to run if the target already exists (safety net against accidentally destroying data). `"overwrite"` disables that — useful while iterating, risky on anything you care about keeping. Worth noticing, not worth overthinking today.



aggregated_df.write.mode('overwrite').parquet('./data/xauusd_daily_ohlc.parquet')
read_df = spark.read_parquet('./data/xauusd_daily_ohlc.parquet')
read_df.show(5)
print('Reading successful')


spark.stop()



---

## Submission Instructions

Paste your code and output below, for each exercise.
