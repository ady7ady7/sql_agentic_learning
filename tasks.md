# PySpark Exercises — 2026-10-01 (Week 40, Day 4)

**Focus:** `.agg()` with multiple aggregate functions in one `groupBy`

---

## Exercise 1 — Min/Max per Day of Week

**Goal:** translate this SQL shape into PySpark DataFrame API:
```sql
SELECT day_of_week, MIN(low), MAX(high)
FROM candles
GROUP BY day_of_week
```

**Steps:**
1. Add `from pyspark.sql import functions as F` at the top of `practice_spark.py`.
2. Group by `day_of_week`, and in one `.agg(...)` call compute `F.min("low")` and `F.max("high")`.
3. Save the result to a variable, print it with `.show()`.

Don't worry about renaming the output columns yet — `.agg()` will give them default names like `min(low)`. That's expected for now.


min_max_by_dow = df.groupBy('day_of_week').agg(
    F.min('low'),
    F.max('high')
    )
min_max_by_dow.show()


---

---

## Exercise 2 — Naming the Output Columns

**Goal:** `.agg()` gave you ugly default names like `min(low)`. Fix that.

In PySpark, you can rename the result of any function call with `.alias("new_name")` — e.g. `F.min("low").alias("day_low")`.

**Steps:**
1. Repeat Exercise 1, but alias `F.min("low")` as `day_low` and `F.max("high")` as `day_high`.
2. Add a third aggregate to the same `.agg()` call: `F.avg("volume")`, aliased as `avg_volume`.
3. `.show()` the result — column names should now read cleanly.


min_max_by_dow = df.groupBy('day_of_week').agg(
    F.min('low').alias('lowest_price'),
    F.max('high').alias('highest_price'),
    F.avg('volume').alias('avg_volume')
)
min_max_by_dow.show()


---

## Exercise 3 — Filter Before Grouping

**Goal:** combine what you already know (`.filter()` from Day 3) with `.groupBy().agg()` from today — the PySpark equivalent of `WHERE ... GROUP BY ...` (filter happens before aggregation, same as SQL).

**Steps:**
1. Start from `df`, filter to only `timeframe == 'm1'` rows (check first whether this filters anything — the file may already be all `m1`).
2. On the filtered result, group by `day_of_week` and compute `day_low`, `day_high`, `avg_volume` same as Exercise 2.
3. Order the result by `avg_volume` descending.
4. `.show()` it.

**Think about order of operations:** does it matter whether you `.filter()` before or after `.groupBy()`? What would happen if you tried to filter on `volume` AFTER grouping — would `df.filter(df.volume > ...)` even work on a grouped/aggregated result? (You don't need to test this — just think about it, we'll cover `HAVING`-equivalent filtering on aggregated results another day.)



filtered_df = df.filter(df.timeframe == 'm1')
filtered_df = filtered_df.groupBy('day_of_week').agg(
    F.min('low').alias('day_low'),
    F.max('high').alias('day_high'),
    F.avg('volume').alias('avg_vol')
).orderBy('avg_vol', ascending = False)
filtered_df.show()

I think it wouldn't work properly, as I guess filter is the equivalent of WHERE, which works on unaggregated data.




---

## Submission Instructions

Paste your code and output below, for each exercise.
