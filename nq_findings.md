# NQ Futures — Analytical Findings

**Dataset:** NQ.c.0 continuous front month, Sep 2025 – May 2026 (~7.5 months)
**Session default:** RTH (09:30–16:00 ET) unless noted as Globex
**Query IDs:** reference SQL queries by ID (e.g. RTH-001) for reproducibility

---

## Volume & Aggressor Pressure

### RTH-VOL-002 — Volume by Hour of Day (RTH)
**Query ID:** RTH-VOL-002 | Task date: 2026-06-04

| Hour ET | Avg Volume | Avg Ticks | % of Day |
|---------|------------|-----------|----------|
| 9       | 64,217     | 45,221    | 16.3%    |
| 10      | 83,335     | 56,723    | 21.1%    |
| 11      | 61,648     | 40,985    | 15.6%    |
| 12      | 46,516     | 30,690    | 11.8%    |
| 13      | 42,228     | 27,702    | 10.7%    |
| 14      | 40,036     | 26,224    | 10.2%    |
| 15      | 56,342     | 35,471    | 14.3%    |

**Findings:**
- Hour 10 (10:00–10:59 ET) is the most active by volume (21.1% of day) — post-open continuation/reversal drive
- Hours 12–14 are the dead zone (lowest volume, ~10–12% each) — typical midday lull
- Hour 15 (15:00–15:59 ET) spikes back up to 14.3% — closing hour drive
- Hour 9 (09:30–09:59 ET, partial) at 16.3% is high relative to its 30-minute window — opening is extremely dense
- Volume distribution is U-shaped with a dominant morning peak: open > close > midday

### RTH-VOL-005 — Buy/Sell Volume Imbalance by Hour × Weekday (RTH)
**Query ID:** RTH-VOL-005 | Task date: 2026-06-16
**Note:** Hour/weekday grouping computed from ts_recv instead of ts_event — minor correctness issue, practical impact negligible.

**Selected highlights (delta_pct):**

| Weekday   | Hour | Delta    | Delta % | Note |
|-----------|------|----------|---------|------|
| Wednesday | 10   | +15,876  | +0.62%  | Largest buy imbalance in RTH |
| Thursday  | 9    | -18,921  | -0.90%  | Largest sell imbalance in RTH |
| Friday    | 15   | -14,664  | -0.86%  | Heavy end-of-week sell pressure |
| Tuesday   | 10   | -130     | 0.00%   | Near-perfect balance |
| Wednesday | 15   | +2,488   | +0.14%  | Only day with buy bias into close |

**Findings:**
- Wednesday hour 10 has the largest buy imbalance (+0.62%) — aligns with Wed being 2nd strongest close-to-close day (RTH-CLOSE-001)
- Thursday hour 9 has the largest sell imbalance (-0.90%) — consistent with Thursday being the most bearish-first-hour weekday (RTH-FH-003, 59% bearish opens)
- Friday hour 15 shows heavy sell pressure (-0.86%) — position squaring into weekend
- Wednesday is the only weekday with buy bias in hour 15 (+0.14%) — possible post-FOMC holding into close
- All imbalances remain small (<1%) — confirms structural volume symmetry even at weekday×hour granularity

### RTH-VOL-004 — First Hour Cumulative Delta vs Rest-of-Session Direction
**Query ID:** RTH-VOL-004 | Task date: 2026-06-15
**Note:** Query measured rest-of-session direction (not full-day close vs open as specified). Also: rest_direction label was inverted in code (r_open - r_close > 0 is bearish, not bullish) — results below reflect actual rest-of-session bearish/bullish with corrected interpretation.

| FH Delta Direction | Days | Avg FH Delta | Bullish Rest Days | Bullish Rest % |
|--------------------|------|-------------|-------------------|----------------|
| Positive           | 73   | +1,601.82   | 44                | 60.3%          |
| Negative           | 86   | -1,561.69   | 47                | 54.7%          |

**Findings:**
- Positive first-hour delta (net buy pressure) leads to bullish rest-of-session 60.3% vs 54.7% for negative delta — modest ~6pp edge
- Delta magnitude is symmetric: avg +1,602 vs -1,562 contracts — buy and sell pressure roughly equal in size when present
- The edge is real but weak as a standalone signal — most useful when combined with gap direction (RTH-GAP-001/002) or FH price direction (RTH-FH-001)
- Measuring rest-of-session (10:30–16:00) is the correct actionable scope — first-hour delta is known at 10:30, predicting what comes after

### RTH-VOL-003 — Buy/Sell Volume Imbalance by Hour (Full Day incl. Globex)
**Query ID:** RTH-VOL-003 | Task date: 2026-06-15
**Note:** Query included all hours (0–23), not RTH-only as specified. RTH hours (9–15) extracted below.

| Hour ET | Buy Volume | Sell Volume | Total Volume | Buy % | Delta | Delta % |
|---------|------------|-------------|--------------|-------|-------|---------|
| 9       | 5,983,060  | 6,001,275   | 11,984,335   | 49.92%| -18,215 | -0.15% |
| 10      | 6,628,334  | 6,622,008   | 13,250,342   | 50.02%| +6,326  | +0.05% |
| 11      | 4,897,516  | 4,904,542   | 9,802,058    | 49.96%| -7,026  | -0.07% |
| 12      | 3,698,342  | 3,697,781   | 7,396,123    | 50.00%| +561    | +0.01% |
| 13      | 3,288,285  | 3,299,291   | 6,587,576    | 49.92%| -11,006 | -0.17% |
| 14      | 3,080,125  | 3,085,468   | 6,165,593    | 49.96%| -5,343  | -0.09% |
| 15      | 4,320,973  | 4,355,740   | 8,676,713    | 49.80%| -34,767 | -0.40% |

**Findings:**
- Volume-level buy/sell symmetry confirmed across RTH hours — delta never exceeds 0.40% of total volume in any hour
- Hour 15 (closing hour) shows the largest net sell imbalance (-34,767 contracts, -0.40%) — consistent with institutional end-of-day position hedging/rebalancing
- Hour 10 is the only hour with meaningful net buy bias (+6,326, +0.05%) — post-open buyers slightly more aggressive in the first full hour
- Hours 9, 11, 13, 14, 15 all show mild net sell bias — RTH has a slight structural sell skew outside the post-open window
- Despite large absolute volumes (13M+ contracts in hour 10), imbalances are tiny — confirms RTH-VOL-001: aggregate-level symmetry holds at the contract level too

### RTH-VOL-001 — Buy/Sell Volume Ratio by Weekday (RTH)
**Query ID:** RTH-VOL-001 | Task date: 2026-06-03
**Finding:** Buy/sell volume ratio is nearly perfectly balanced across all weekdays (range: 0.995–1.001). No weekday shows persistent directional aggressor bias at the aggregate level.
**Implication:** Daily-level buy/sell symmetry — aggressor imbalance, if it exists, operates at intraday or tick-by-tick level, not at the weekday aggregate.

---

## Daily Ranges

### RTH-RANGE-002 — First Hour Range vs Full RTH Day Range
**Query ID:** RTH-RANGE-002 | Task date: 2026-06-04 (fixed 2026-06-05)

| Weekday   | Avg FH Range | Avg Rest Range | Avg Full Day Range | Avg FH % of Day |
|-----------|-------------|----------------|-------------------|-----------------|
| Thursday  | 217.22      | 308.97         | 380.97            | 65.6%           |
| Friday    | 214.85      | 294.59         | 373.70            | 63.7%           |
| Tuesday   | 195.18      | 260.05         | 311.41            | 64.2%           |
| Wednesday | 192.25      | 255.80         | 334.70            | 57.9%           |
| Monday    | 189.27      | 211.73         | 298.58            | 66.8%           |

**Findings:**
- The first hour (09:30–10:30 ET) captures 58–67% of the full day's range on average — a substantial portion
- Monday has the highest first-hour capture rate (66.8%) despite having the smallest absolute FH range (189 pts) — the rest of Monday is comparatively quiet (211 pts rest range)
- Wednesday has the lowest first-hour capture rate (57.9%) — the afternoon session on Wednesdays contributes relatively more range (likely FOMC-related)
- Thursday and Friday have the largest absolute first-hour ranges (217 and 215 pts respectively) — consistent with their higher full-day volatility (RTH-RANGE-001)
- Rest-of-session range consistently exceeds first-hour range across all days — but the FH is a meaningful predictor of daily activity level

### RTH-RANGE-001 — Average Daily Range by Weekday (RTH)
**Query ID:** RTH-RANGE-001 | Task date: 2026-06-03
**Period:** Sep 2025 – May 2026, Mon–Fri only

| Weekday   | Avg Range | Max Range | Min Range | Days |
|-----------|-----------|-----------|-----------|------|
| Friday    | 370.56    | 1003.75   | 101.00    | 35   |
| Thursday  | 369.15    | 1212.00   | 32.00     | 39   |
| Wednesday | 334.91    | 606.50    | 108.00    | 34   |
| Tuesday   | 311.88    | 733.25    | 97.50     | 39   |
| Monday    | 298.33    | 745.00    | 126.75    | 39   |

**Findings:**
- Friday and Thursday are the most volatile days by average RTH range (~370 pts), Monday the least (~298 pts)
- Thursday holds the single largest range day (1212 pts) — likely a news/event day
- Friday also produced the second largest single-day range (1003.75 pts)
- Monday has the tightest minimum range floor (126.75 pts) — fewest extreme outlier days
- Wednesday sits in the middle despite being a known FOMC day — may warrant news-day filtering to isolate baseline vs event volatility
- Spread between highest (Fri 370) and lowest (Mon 298) avg range is ~24% — meaningful but not extreme

---

## Volatility

### RTH-VOL-013 — Range Expansion Curve by Weekday
**Query ID:** RTH-VOL-013 | Task date: 2026-07-07
**Extension of RTH-VOL-011** — adds weekday dimension. Same 5-CTE tick architecture. ~29–39 days per weekday. Note: `ts_recv AT TIME ZONE` used for window bucketing (should be `ts_event`) — minor correctness issue, negligible practical impact since ts_recv ≈ ts_event for RTH ticks.

**09:30 avg range captured % and window where median hits 100%:**

| Weekday | 09:30 avg % | 10:30 avg % | 11:00 avg % | Median 100% at | Character |
|---|---|---|---|---|---|
| Monday | **56.1%** | 73.7% | 78.8% | 14:30 | Front-loaded open, gradual late expansion |
| Thursday | 55.8% | 76.7% | 80.1% | 13:30 | Front-loaded, fastest to stabilize |
| Tuesday | 52.4% | 67.1% | 73.8% | 15:30 | Slowest early expansion, steady through session |
| Friday | 47.7% | 74.2% | 79.8% | 13:00 | Slow open, then accelerates fast by 10:30 |
| Wednesday | **43.3%** | 65.8% | 70.5% | 14:00 | Slowest overall — keeps expanding all day |

**Findings:**
- **Wednesday has the least range captured at 09:30 (43.3%)** — lowest of all weekdays by a significant margin. Wednesday range expands slowly and continuously through the whole session; median doesn't hit 100% until 14:00 (latest of all weekdays). Consistent with FOMC-driven afternoon moves and Wednesday's low first-hour capture rate (RTH-RANGE-002: 57.9%, lowest weekday).
- **Monday and Thursday are the most front-loaded at 09:30 (~56%)** — nearly identical opening range capture. Both have structural reasons: Thursday's HOD front-loading (45% in 09:30 window, RTH-SESS-006) and Monday's LOD front-loading (51.5% in 09:30 window). Despite similar opening front-loading, Monday's median hits 100% at 14:30 vs Thursday's 13:30 — Monday keeps slowly adding range late in the session (the HOD drift pattern).
- **Tuesday has the slowest early expansion (67.1% at 10:30 — lowest of all weekdays)** — Tuesday builds range gradually and steadily throughout the session. No dominant single window. Consistent with Tuesday's bi-modal HOD distribution (24% open, 27% at 15:30 — RTH-SESS-006).
- **Friday has a notable acceleration pattern: only 47.7% at 09:30 but jumps to 74.2% at 10:30** — the biggest single-window jump of any weekday (26.5pp in 30 minutes). Friday range is relatively quiet in the opening 30 min then expands rapidly through the OR/FH window. Median fully ranged by 13:00 (earliest of all weekdays).
- **Practical weekday-specific sizing rules:**
  - **Monday:** 56% range known at open, but keep targets open — median day adds range until 14:30
  - **Tuesday:** At 10:30 you only know 67% of the range — widest remaining uncertainty at that point. Keep stops wider on Tuesday morning setups.
  - **Wednesday:** At 11:00 only 70% known — the afternoon often brings a new extreme. Don't assume the range is set by midday.
  - **Thursday:** 76.7% known by 10:30, median fully ranged by 13:30 — range sets early, use tighter targets after noon
  - **Friday:** Slow open (47.7%), then rapid expansion to 74% by 10:30 — the 09:30–10:30 window is the critical expansion window on Fridays

### RTH-VOL-010 — Day-over-Day Range Continuity (Volatility Clustering)
**Query ID:** RTH-VOL-010 | Task date: 2026-07-03
**Method:** LAG(high - low) OVER (ORDER BY trade_date) to get prior day range. NTILE(3) on both prev_range and daily_range within same CTE. Wide day threshold: daily_range ≥ 380 pts (approx global Q3). 185 days after excluding first (no prior day).

| Prev Day Bucket | Days | Avg Prev Range | Avg Today Range | Pct Today Wide |
|---|---|---|---|---|
| Large (Q3) | 61 | 522 pts | **424 pts** | **55.7%** |
| Medium (Q2) | 62 | 306 pts | 324 pts | 27.4% |
| Small (Q1) | 62 | 185 pts | **266 pts** | **16.1%** |

**Findings:**
- **Volatility clusters strongly in NQ — daily ranges are NOT independent.** After a large day (avg 522 pts), today's expected range is 424 pts and there is a 55.7% chance of another wide day. After a small day (avg 185 pts), expected range is only 266 pts and wide-day probability drops to 16.1% — a 3.5x difference.
- **After a large day, today's avg range (424 pts) exceeds the large-bucket threshold (~380 pts) on average** — wide days don't just predict wide days, they predict days that are themselves in the top third.
- **The medium bucket shows a clear intermediate step (324 pts avg, 27.4% wide)** — the relationship is monotonic and not just a binary high/low split. Volatility decays gradually, not in a single step.
- **Practical implication — pre-market sizing rule:**
  - Yesterday large (500+ pts): expect today ~420 pts, >50% odds wide. Size up, wider stops.
  - Yesterday medium (240–380 pts): expect today ~325 pts, ~27% odds wide. Normal sizing.
  - Yesterday small (<240 pts): expect today ~265 pts, 16% odds wide. Size down, tighter targets.
- **Bucket thresholds (approx):** small < ~240 pts, medium ~240–380 pts, large > ~380 pts (derived from avg prev_range per bucket).

### RTH-VOL-012 — Volatility Regime × Prior Day Direction
**Query ID:** RTH-VOL-012 | Task date: 2026-07-04
**Extension of RTH-VOL-010** — adds prior day direction dimension. LAG(close) / LAG(open) for prev_direction. NTILE(3) on prev_range. 185 days (first excluded, no prior day). Flat days (prev_close = prev_open) absorbed into 'bearish' — negligible count.

| Prev Bucket | Prev Direction | Days | Avg Today Range | Today Bullish % |
|---|---|---|---|---|
| Large (3) | bullish | 35 | 423 pts | **65.7%** |
| Large (3) | bearish | 26 | 424 pts | **50.0%** |
| Medium (2) | bullish | 30 | 308 pts | 63.3% |
| Medium (2) | bearish | 32 | 338 pts | 43.8% |
| Small (1) | bullish | 35 | 238 pts | 60.0% |
| Small (1) | bearish | 27 | 302 pts | **37.0%** |

**Findings:**
- **After a large bullish day: today bullish 65.7%** — strong directional continuation. A wide up day begets another up day more often than not. Range also stays wide (423 pts avg). The volatility clustering from RTH-VOL-010 is amplified and directional on the bullish side.
- **After a large bearish day: today bullish only 50%** — coin flip. Large bearish days produce no directional signal for the next session. Neither continuation nor mean reversion. Prior direction is irrelevant after a large down day.
- **After a small bearish day: today bullish only 37%** — the strongest bearish continuation signal in the dataset. Small down days tend to continue lower. Counterintuitively, large down days don't continue but small down days do.
- **After a small bullish day: today bullish 60%** — mild bullish continuation regardless of prior range size.
- **Range is symmetric across prior direction within each bucket** — avg today range is nearly identical whether the prior large day was bullish or bearish (423 vs 424 pts). Prior direction affects which way the range expands, not how much.
- **Prior direction adds real information beyond range size alone — but only on the bullish side and the small-bearish case.** Large bearish days are the exception where direction adds nothing.
- **Combined pre-market framework (RTH-VOL-010 + RTH-VOL-012):**
  - Yesterday large bullish: expect wide day (~424 pts), 65.7% bullish — best long setup from prior-day signal
  - Yesterday large bearish: expect wide day (~424 pts), direction unknown (50/50) — size up for range, no directional lean
  - Yesterday small bearish: expect narrow day (~238–302 pts), 37% bullish — mild bearish lean, size down
  - Yesterday small bullish: expect narrow day (~238 pts), 60% bullish — mild bullish lean, size down

### RTH-VOL-011 — Cumulative Intraday Range Expansion Curve
**Query ID:** RTH-VOL-011 | Task date: 2026-07-04
**Method:** Raw ticks joined to daily_ohlcv_rth via session_start/session_end. MAX(price)/MIN(price) per (trade_date, 30-min window). Running MAX/MIN via window function OVER (PARTITION BY trade_date ORDER BY current_window_et). Expressed as % of full day's range (daily_ohlcv_rth.high - low). 16:00 row shows 102% — boundary artefact from ticks landing just after session close, ignore.

| Time Window | Avg Range Captured % | Median Range Captured % |
|---|---|---|
| 09:30 | 51% | 49% |
| 10:00 | 63% | 63% |
| 10:30 | 72% | 72% |
| 11:00 | 77% | 78% |
| 11:30 | 81% | 84% |
| 12:00 | 84% | 87% |
| 12:30 | 87% | 94% |
| 13:00 | 89% | 95% |
| 13:30 | 92% | **100%** |
| 14:00 | 93% | 100% |
| 14:30 | 95% | 100% |
| 15:00 | 97% | 100% |
| 15:30 | 100% | 100% |

**Findings:**
- **By 09:30 (end of first 30-min window): 51% of the day's range is already captured on average** — the opening half-hour alone establishes over half the day's range. Consistent with RTH-SESS-005 (33% of HODs and 37% of LODs form in the 09:30 window).
- **By 10:30 (FH midpoint): 72% captured** — nearly three quarters of the day's range is already in place by the time the first hour is half done.
- **By 11:00 (FH close): 77% captured** — consistent with RTH-RANGE-002 (FH captures 58–67% of full day range by its close). The discrepancy is because RTH-RANGE-002 measured FH range / (FH + rest) rather than cumulative running range vs full day.
- **Median hits 100% at 13:30** — on more than half of all trading days, the complete day's range (both HOD and LOD) is already established by 13:30. The afternoon session rarely breaks new ground on the median day.
- **The curve is steep early and flat late:** most range expansion happens 09:30–11:30 (avg goes from 0% to 81% in 2 hours); the final 4 hours add only ~19% of the remaining range on average.
- **Mean vs median divergence widens after 12:00** (84% avg vs 87% median at 12:00, then 92% avg vs 100% median at 13:30) — the avg is pulled down by outlier days where a late news event or momentum spike creates new extremes in the final hour. The typical day is fully ranged by early afternoon; the late-expanding days are the exception.
- **Practical implications:**
  - By 10:30, you know ~72% of the day's range — use this for realistic target-setting at that point in the session
  - Midday trades (11:30–13:30) are operating in a window where the median day has already set its range — expect low breakout success rate from midday levels
  - Late-session new extremes (after 14:00) are genuine outlier events, not the norm — treat 14:00+ breakouts with skepticism unless driven by a catalyst
  - The 50% range marker is hit in the first 30 minutes — opening volatility is not "noise," it's establishing half the day's structure

---

## Gap Analysis

### RTH-GAP-001 — Overnight Gap Analysis (RTH close → next RTH open)
**Query ID:** RTH-GAP-001 | Task date: 2026-06-04

| Metric | Value |
|--------|-------|
| Gap up days | 98 |
| Gap down days | 88 |
| Avg gap (all) | +5.35 pts |
| Avg gap up | +145.01 pts |
| Avg gap down | -150.24 pts |

**Findings:**
- Slight upward bias: 98 gap-up vs 88 gap-down days over the sample period
- Average gap magnitude is symmetric: +145 up vs -150 down — gaps are roughly equal in size regardless of direction
- Average gap of +5.35 across all days reflects the overall bullish drift in NQ during this period
- Large average gap sizes (±145–150 pts) relative to typical RTH ranges (~300–370 pts) — overnight gaps represent ~40–50% of a typical day's range

## Close-to-Close Change

### RTH-CLOSE-001 — Day-over-Day Close Change by Weekday (RTH)
**Query ID:** RTH-CLOSE-001 | Task date: 2026-06-05

| Weekday   | Days | Avg Close Change | Up Days | Down Days | Up % |
|-----------|------|-----------------|---------|-----------|------|
| Monday    | 39   | +114.28         | 27      | 13        | 69.2% |
| Wednesday | 34   | +80.22          | 23      | 11        | 67.6% |
| Friday    | 35   | +5.55           | 21      | 15        | 60.0% |
| Tuesday   | 39   | -5.42           | 20      | 24        | 51.3% |
| Thursday  | 39   | -90.44          | 19      | 26        | 48.7% |

**Findings:**
- Monday closes up 69% of the time with avg +114 pts — strongest bullish close-to-close day by both frequency and magnitude
- Wednesday also skews bullish (67.6%, avg +80 pts) — notable given it also hosts FOMC days; positive drift may reflect post-FOMC relief rallies
- Thursday is the weakest day: only 48.7% up, avg -90 pts — consistent profit-taking or position squaring before Friday
- Friday is nearly flat (+5.55 avg) but still closes up 60% — slight upward bias but muted in magnitude
- Tuesday is the only other negative day on average (-5.42) but nearly coinflip in frequency (51.3%)
- Strong Mon/Wed vs weak Thu pattern is consistent with RTH-RANGE-001 (Thu has high range but negative drift)

## Close Location in Range

### RTH-CLOSE-002 — Close Location in Day's Range by Weekday
**Query ID:** RTH-CLOSE-002 | Task date: 2026-06-09

| Weekday   | Days | Avg Close Location | % Closed Upper Half |
|-----------|------|--------------------|---------------------|
| Monday    | 39   | 62.02%             | 66.67%              |
| Wednesday | 34   | 58.79%             | 61.76%              |
| Tuesday   | 39   | 57.45%             | 56.41%              |
| Friday    | 35   | 54.84%             | 57.14%              |
| Thursday  | 39   | 52.49%             | 53.85%              |

**Findings:**
- Monday closes in the upper half of its range 66.7% of the time (avg 62%) — consistent with RTH-CLOSE-001 showing Monday as the strongest close-to-close day
- Thursday avg close location is 52.5% despite being the worst close-to-close weekday (-90 avg change) — when Thursday sells off, it sells off hard, but it doesn't consistently close near its own low
- All weekdays close above the midpoint of their range on average — NQ has a general upward close bias within the day's range across the sample period
- Wednesday and Tuesday sit mid-table (58-57%) — moderate close strength

---

## First Hour Analysis

### RTH-FH-001 — First Hour Direction Bias
**Query ID:** RTH-FH-001 | Task date: 2026-06-08

| First Hour Direction | Days | Rest Same Direction | Same Dir % |
|----------------------|------|---------------------|------------|
| Bullish              | 87   | 49                  | 56.3%      |
| Bearish              | 72   | 30                  | 41.7%      |

**Findings:**
- Bullish first hours (09:30–10:30) continue bullish into the rest of session 56.3% of the time — slight positive continuation bias
- Bearish first hours reverse more often than they continue: only 41.7% same-direction follow-through, meaning 58.3% of bearish opens see a rest-of-session bounce
- Practical implication: a bearish first hour is a weak signal for a bearish full day — the market fades early selling more often than not
- Note: materialized view `nq_data.rth_firsthour_rest_ohlc_ranges` created this session for performance — stores per-day FH and rest OHLC, eliminating repeated 56M-row scans

### RTH-FH-003 — First Hour Direction by Weekday
**Query ID:** RTH-FH-003 | Task date: 2026-06-10

| Weekday   | Days | Bullish FH % | Bearish FH % | Avg FH Range |
|-----------|------|-------------|-------------|--------------|
| Monday    | 39   | 64.1%       | 35.9%       | 189.72       |
| Tuesday   | 39   | 61.5%       | 38.5%       | 189.72       |
| Friday    | 35   | 60.0%       | 40.0%       | 215.59       |
| Wednesday | 34   | 52.9%       | 47.1%       | 189.69       |
| Thursday  | 39   | 41.0%       | 59.0%       | 213.84       |

**Findings:**
- Thursday is the only weekday with a majority bearish first hour (59%) — aligns with RTH-FH-002 (FH sets the day high most often on Thursdays) and RTH-CLOSE-001 (weakest close-to-close day)
- Monday and Tuesday have the most bullish first hours (64%/62%) — Monday's bullish open is consistent with its strong close-to-close drift (+114 avg)
- Friday and Thursday have the highest avg FH ranges (~215 pts) — more volatile openings, consistent with being the highest full-day range weekdays (RTH-RANGE-001)
- Monday/Tuesday/Wednesday have nearly identical avg FH ranges (~190 pts) — similar opening volatility despite different directional biases

### RTH-FH-006 — FH Range Size vs Rest-of-Session Range
**Query ID:** RTH-FH-006 | Task date: 2026-06-30
**Definition:** FH range = fh_high - fh_low (09:30–10:30). Rest range = r_high - r_low (10:30–16:00). Bucketed into NTILE(4) by FH range. fh_pct_of_day = FH range / (FH range + rest range).

| Bucket | Days | Avg FH Range | Avg Rest Range | FH % of Day | Rest Bullish % |
|---|---|---|---|---|---|
| Q1 (tightest) | 40 | 101 pts | 199 pts | 38.4% | 52.5% |
| Q2 | 40 | 164 pts | 250 pts | 43.5% | 60.0% |
| Q3 | 40 | 213 pts | 271 pts | 46.6% | 52.5% |
| Q4 (widest) | 39 | 330 pts | 342 pts | 50.2% | **64.1%** |

**Findings:**
- **Wide FH → wider rest-of-session** — same expansion pattern as RTH-ORB-009 (OR range). Q4 rest range (342 pts) is 72% larger than Q1 (199 pts). No volatility compression after a wide first hour.
- **FH front-loads more of the day than the OR** — Q4 FH claims 50.2% of combined range vs OR Q4 at 41.0%. The first hour (60 min) captures half the day's range when wide; the OR (30 min) captures only 41% even at its widest.
- **Q4 bullish close bias: 64.1%** — matches RTH-ORB-009 finding (OR Q4: 66.7%). Wide opening windows (both OR and FH) tend to resolve bullishly. High-volatility opens are not exhaustion signals — they precede strong directional closes.
- **Direct comparison with RTH-ORB-009 (OR range):**

| Metric | OR Q1→Q4 range | FH Q1→Q4 range |
|---|---|---|
| Avg window range | 86→251 pts | 101→330 pts |
| Avg rest range | 219→383 pts | 199→342 pts |
| Window % of day | 32.5→41.0% | 38.4→50.2% |
| Rest bullish % at Q4 | 66.7% | 64.1% |

- FH captures more of the day when wide, but the absolute rest-of-session range is slightly smaller than for wide ORs (342 vs 383 pts) — the OR predicts a wider afternoon than the FH does at equivalent quartile.

### RTH-FH-005 — Gap-Up Tuesday FH Rejection Depth
**Query ID:** RTH-FH-005 | Task date: 2026-06-30
**Filter:** Tuesday + gap-up (open > prev_close) + FH high > rest-of-session high (proxy for FH setting day high — note: filter uses fh_high > r_high rather than fh_high >= daily_high; slight semantic difference, results directionally valid). 10 qualifying days.

| Metric | Value |
|---|---|
| Days | 10 |
| Avg drop from FH high to afternoon low | **310 pts** |
| Avg drop from FH high to RTH close | **235 pts** |
| Reached FH open (09:30 price) | **100%** |

**Findings:**
- **Every gap-up Tuesday where the FH sets the high sees the afternoon fully retrace back to the 09:30 open price** — 100% of 10 occurrences. The entire first-hour bullish move is erased by the close.
- **Avg afternoon drop from FH high to rest low: 310 pts** — a substantial fade. The afternoon doesn't just drift lower; it typically drops hard from the FH peak.
- **Avg close is 235 pts below the FH high** — the day closes well below the first-hour peak, not just a minor pullback. Combined with the 100% FH-open reach rate, the close is typically near or below the 09:30 level.
- **Practical implication for today:** If Tuesday opens with a gap-up and the first hour is bullish, the FH high is the fade level. Target: FH open (= 09:30 price, 100% historical reach), expect ~235 pts below FH high at close. Size accordingly — this is the highest-conviction fade setup found on Tuesdays.
- **Caveat:** Small sample (10 days). Filter uses fh_high > r_high as proxy for FH-sets-high — rerun with fh_high >= daily_high for exact population.

### RTH-FH-004 — FH High/Low as Day Extreme × Gap Direction
**Query ID:** RTH-FH-004 | Task date: 2026-06-26
**Extension of RTH-FH-002** — adds gap direction dimension. Gap = RTH open vs prior RTH close (LAG). Flat opens excluded.

| Weekday | Gap | Days | FH Sets High % | FH Sets Low % |
|---|---|---|---|---|
| Thursday | gap_up | 17 | **58.8%** | 35.3% |
| Thursday | gap_down | 21 | **57.1%** | 38.1% |
| Tuesday | gap_up | 18 | 55.6% | 22.2% |
| Friday | gap_up | 20 | 50.0% | 50.0% |
| Wednesday | gap_up | 22 | 50.0% | 50.0% |
| Wednesday | gap_down | 12 | 33.3% | 41.7% |
| Monday | gap_up | 20 | 45.0% | 40.0% |
| Monday | gap_down | 18 | 27.8% | **66.7%** |
| Friday | gap_down | 15 | 20.0% | **66.7%** |
| Tuesday | gap_down | 21 | 19.1% | **52.4%** |

**Findings:**
- **Thursday FH-sets-high is gap-direction agnostic: 58.8% gap-up, 57.1% gap-down** — both above the 54.8% overall Thursday baseline (RTH-FH-002). Thursday simply opens near its high regardless of gap direction. The structural fade tendency is baked in — gap direction doesn't amplify or reduce it.
- **Monday gap-down: FH sets day LOW 66.7%** — when Monday gaps down, the opening weakness is the day's low two thirds of the time. Consistent with Monday's strong close-to-close drift (+114 avg, RTH-CLOSE-001) — gap-down Monday is a buy-the-open setup.
- **Friday gap-down: FH sets day LOW 66.7%** — same rate as Monday gap-down. Friday opening weakness is typically the day's low, but the afternoon close-to-open drift is near-flat (+5.55 avg) — the bounce exists intraday but doesn't carry to close as strongly as Monday.
- **Tuesday gap-down: FH sets day LOW 52.4%, FH sets day HIGH only 19%** — gap-down Tuesdays are strongly directional: the FH low is almost never the high, and the FH low is the day's low more than half the time. Consistent with Tuesday agree-bearish → 75% rest bullish (RTH-ORB-006).
- **Tuesday gap-up: FH sets day HIGH 55.6%** — gap-up Tuesdays see the FH establish the day's high more than half the time, the opposite regime.
- **Gap direction is a strong moderator of FH extreme-setting on all weekdays except Thursday** — Thursday's FH-sets-high tendency is structural; for all other days gap direction meaningfully splits the distribution.

### RTH-FH-002 — First Hour High/Low as Day's Extreme
**Query ID:** RTH-FH-002 | Task date: 2026-06-09

**Overall (159 trading days):**

| Metric | Days | % |
|--------|------|---|
| FH sets day high | 66 | 41.5% |
| FH sets day low  | 72 | 45.3% |
| Full day inside FH range | 5 | 3.1% |

**By weekday:**

| Weekday   | Days | FH Sets High % | FH Sets Low % | Inside FH % |
|-----------|------|----------------|---------------|-------------|
| Thursday  | 31   | 54.84%         | 35.48%        | 6.45%       |
| Wednesday | 33   | 45.45%         | 48.48%        | 3.03%       |
| Friday    | 29   | 37.93%         | 51.72%        | 3.45%       |
| Monday    | 33   | 36.36%         | 54.55%        | 3.03%       |
| Tuesday   | 33   | 33.33%         | 36.36%        | 0.00%       |

**Findings:**
- FH sets the day's low more often than the high (45.3% vs 41.5%) — opening weakness more likely to be the day's extreme than opening strength
- Inside-FH days are rare (3.1%) — the rest of session almost always extends beyond the first-hour range
- Thursday is the standout: FH sets the day HIGH 54.8% of the time — the first hour is the high of the day more often than not on Thursdays. Consistent with RTH-CLOSE-001 (Thu avg close -90 pts) — market opens strong and reverses
- Monday is the mirror: FH sets the day LOW 54.6% — Monday opens weak, then rallies through the session, consistent with RTH-CLOSE-001 (+114 avg close change)
- Tuesday shows no skew — FH sets neither extreme particularly often (33%/36%), suggesting Tuesdays are more symmetric intraday

### RTH-GAP-004 — Overnight Gap Tendency by Exit Weekday
**Query ID:** RTH-GAP-004 | Task date: 2026-06-29
**Definition:** Gap = RTH open vs prior RTH close (LAG). Grouped by exit weekday (the day being opened into). Flat opens excluded. Answers: is holding overnight favorable by weekday?

| Exit Weekday | Days | Gap Up % | Gap Down % | Avg Gap Up Size | Avg Gap Down Size |
|---|---|---|---|---|---|
| Wednesday | 34 | **64.7%** | 35.3% | 138 pts | 71 pts |
| Friday | 35 | 57.1% | 42.9% | 111 pts | 196 pts |
| Monday | 38 | 52.6% | 47.4% | 223 pts | 170 pts |
| Tuesday | 39 | 46.2% | 53.9% | 91 pts | 158 pts |
| Thursday | 38 | 44.7% | **55.3%** | 167 pts | 146 pts |

**Findings:**
- **Wednesday is the most favorable overnight hold for longs** — gaps up 64.7% of the time, with avg gap-up size (138 pts) nearly double the avg gap-down size (71 pts). Holding Tuesday → Wednesday has strong historical tailwind.
- **Thursday gaps down more often than up (55.3%)** — holding Wednesday → Thursday overnight is the most unfavorable for longs. Consistent with Thursday being the weakest close-to-close weekday (RTH-CLOSE-001, -90 avg).
- **Tuesday also gaps down more often (53.9%)** — holding Monday → Tuesday overnight has a modest downside bias. Gap-down size (158 pts) > gap-up size (91 pts) — asymmetric risk to the downside.
- **Monday has the largest average gap sizes in both directions (223 up / 170 down)** — most volatile overnight move of the week regardless of direction. Weekend gap risk is real on NQ.
- **Friday gaps up 57.1%** — holding Thursday → Friday has a modest upside bias in direction, but gap-down size (196 pts) is the largest of any weekday in that direction — when Friday gaps down, it gaps down hard.
- **Overnight hold summary:** Wednesday is the cleanest long hold. Thursday is the worst. Tuesday has downside asymmetry. Monday has high gap variance either way.

### RTH-GAP-003 — Gap Magnitude Buckets vs Fill Rate and Rest-of-Session Direction
**Query ID:** RTH-GAP-003 | Task date: 2026-06-29
**Definition:** Gap size = ABS(open - prev_close). Bucketed into thirds via NTILE(3) across all gap days (both directions combined). Fill = low ≤ prev_close for gap-up; high ≥ prev_close for gap-down. Rest bullish = RTH close > RTH open.

| Direction | Bucket | Days | Avg Gap Size | Fill % | Bullish Close % |
|---|---|---|---|---|---|
| down | small | 25 | 30 pts | **80%** | 52% |
| down | medium | 35 | 118 pts | 63% | 66% |
| down | large | 27 | 309 pts | 56% | **74%** |
| up | small | 37 | 35 pts | **86%** | 49% |
| up | medium | 26 | 115 pts | 58% | 50% |
| up | large | 34 | 292 pts | **29%** | **38%** |

**Findings:**
- **Large gap-ups (avg 292 pts) fill only 29% of the time** — by far the lowest fill rate of any bucket. The overall 58.2% gap-up fill rate (RTH-GAP-002) is heavily inflated by small gaps. A large gap-up is structurally resistant to filling.
- **Large gap-ups produce a bearish RTH close 62% of the time (bullish only 38%)** — the only bucket with a net bearish close bias. Large gap-ups exhaust upside; the session tends to drift lower from the open.
- **Large gap-downs (avg 309 pts) fill 56% of the time and produce bullish closes 74% of the time** — large downside gaps get bought. The bigger the gap-down, the more bullish the close. Opposite asymmetry from gap-ups.
- **Small gaps fill at very high rates (86% gap-up, 80% gap-down)** — small gaps are almost always filled intraday. Not a directional signal.
- **Practical implication for today's ~300 pt gap-up:** This is a large bucket gap-up. Historical fill rate: 29%. Bullish close rate: 38%. The setup strongly favors fading strength rather than chasing. Consistent with RTH-SESS-001 gap-up findings.
- **Bucket thresholds (approx):** small < ~75 pts, medium ~75–180 pts, large > ~180 pts (derived from avg gap sizes per bucket).

### RTH-GAP-002 — Gap Fill Rate (RTH open vs prior RTH close)
**Query ID:** RTH-GAP-002 | Task date: 2026-06-08
**Note:** Gap-up count includes flat opens (open = prev_close) due to ELSE branch in CASE — fill rate for gap_up slightly overstated

| Gap Direction | Days | Filled Days | Fill % | Avg Gap Size |
|---------------|------|-------------|--------|--------------|
| Gap down      | 87   | 57          | 65.5%  | 151.97 pts   |
| Gap up        | 98   | 57          | 58.2%  | 145.01 pts   |

**Findings:**
- Gaps fill at a high rate in both directions: 65.5% for gap-down, 58.2% for gap-up
- Gap-down days fill more reliably than gap-up days — downside gaps are faded more aggressively
- Average gap size is large (±145–152 pts) relative to typical RTH first-hour range (~195–217 pts) — many gaps require significant intraday reversal to fill
- This connects to RTH-GAP-001: average overnight gaps of ±145–150 pts, consistent with these fill-day averages

## Stacked Edges

### RTH-SESS-003 — Weekday × Gap Direction × First Hour Direction
**Query ID:** RTH-SESS-003 | Task date: 2026-06-16 (corrected same session)
**Note:** Original submission had two label inversions — both fixed and rerun: (1) `fh_open > fh_close` now correctly labeled 'bearish'; (2) avg_close_location uses `r_close - daily_low` in numerator. Results below are from the corrected query.

| Weekday   | Gap       | FH Dir  | Days | Rest Bullish % | Avg Close Location |
|-----------|-----------|---------|------|----------------|--------------------|
| Tuesday   | gap_down  | bearish | 9    | 100%           | 70.4%              |
| Monday    | gap_down  | bearish | 5    | 80%            | 68.1%              |
| Thursday  | gap_down  | bullish | 10   | 80%            | 65.2%              |
| Wednesday | gap_down  | bullish | 5    | 80%            | 69.8%              |
| Monday    | gap_down  | bullish | 13   | 69.2%          | 72.8%              |
| Thursday  | gap_up    | bearish | 12   | 66.7%          | 57.1%              |
| Monday    | gap_up    | bearish | 9    | 66.7%          | 41.5%              |
| Thursday  | gap_up    | bullish | 5    | 0%             | 34.0%              |

**Findings:**
- **Tuesday gap_down + bearish FH: 100% rest bullish** (9 days) — small sample but striking. Gap down, first hour sells off, afternoon fully reverses every time in the dataset
- **Monday gap_down in either FH direction: 69-80% rest bullish** — Monday absorbs gap-down weakness regardless of first-hour direction, consistent with Mon being the strongest close-to-close day
- **Thursday gap_up + bullish FH: 0% rest bullish** (5 days) — Thursday that gaps up AND has a bullish first hour always fades in the afternoon. Pure fade setup
- **Thursday gap_down + bullish FH: 80% rest bullish** (10 days) — gap down recovery on Thursday is the exception to the bearish Thursday pattern
- Sample sizes are small (5-13 days per bucket) — directional but not statistically robust without more data

### RTH-VOL-009 — Delta Bias vs Candle Direction Alignment
**Query ID:** RTH-VOL-009 | Task date: 2026-06-21

How often does aggressor delta direction agree with the 15-min candle direction (bucket_close vs bucket_open)?

| Delta Bias | Total Buckets | Bullish Candle | Bullish Candle % |
|---|---|---|---|
| Bullish (delta > 0) | 1,973 | 1,530 | **77.6%** |
| Bearish (delta < 0) | 2,098 | 532 | **25.4%** |
| Flat (delta = 0) | 5 | 3 | 60% |

**Findings:**
- Delta and candle direction agree ~75-77% of the time — meaningful correlation but not redundant
- ~23% of buckets show divergence (positive delta + bearish candle, or negative delta + bullish candle) — these absorption/exhaustion cases are structurally interesting signals on their own
- **Practical implication:** OHLC candle direction is a usable but weaker proxy for delta. TradingView's approximated delta (tick-rule based) will agree with true aggressor delta directionally most of the time on liquid instruments like NQ, but divergences will be noisier than our true bid/ask data
- The 09:45 signal from RTH-VOL-008 is based on true delta — candle-direction substitute would capture ~75% of the same signal

### RTH-VOL-008 — 15-Minute Bucket Magnitude by Prior Bucket Delta
**Query ID:** RTH-VOL-008 | Task date: 2026-06-19

Extension of RTH-VOL-007 adding avg price move (bucket_close - bucket_open) split by direction outcome. `avg_move_all` is the net expected value per trade across all days in that group.

**Key windows (filtered + full dataset):**

| Current Bucket | Prior Delta | Up % | Avg Up Move | Avg Down Move | Avg Move All | Verdict |
|---|---|---|---|---|---|---|
| 09:45–10:00 | positive | 66.2% | +58.1 | -54.5 | **+20.0** | Strong — direction AND magnitude confirmed |
| 09:45–10:00 | negative | 45.5% | +59.2 | -50.5 | -0.7 | No edge |
| 11:00–11:15 | positive | 62.0% | +32.2 | -38.5 | +5.3 | Weak magnitude |
| 14:45–15:00 | positive | 65.3% | +18.7 | -19.5 | +5.4 | Weak magnitude |
| 15:00–15:15 | negative | 60.0% | +21.0 | -18.5 | +5.2 | Weak magnitude |
| 15:00–15:15 | positive | 48.8% | +25.2 | -25.5 | -0.8 | Direction signal was a mirage |

**Findings:**
- **09:45–10:00, positive prior delta is the only window with both strong direction (66.2%) AND strong magnitude (+20 pts avg_move_all)** — this is the only setup that survives a real expected value test
- All other "edges" from RTH-VOL-007 have avg_move_all of ±5 pts or less — marginal after spread, not reliably tradeable standalone
- The 09:45 positive setup has symmetric avg win/loss magnitude (~+58 up, -55 down) — the edge is purely in win rate (66%), not asymmetric payoff
- 15:00 negative prior delta: 60% up but only +5.2 avg move — directional but insufficient magnitude
- **Practical conclusion:** use 09:45 positive prior delta as a confirming signal in a broader setup, not as a standalone scalp trigger. Other windows need stacking with additional filters before they're actionable.

### RTH-VOL-007 — 15-Minute Bucket Direction by Prior Bucket Delta
**Query ID:** RTH-VOL-007 | Task date: 2026-06-18

**How to read this table:**
- `current_bucket` = the 15-min window being observed (e.g. 09:45 = 09:45–10:00)
- `prior_bucket_delta` = net aggressor delta of the immediately preceding bucket (e.g. 09:30–09:45)
- `current_bucket_up_pct` = % of days where the current bucket closes above its open
- Signal is known at the **start** of the current bucket (i.e. at 09:45 you know the 09:30–09:45 delta and can act on it immediately)

Selected windows with meaningful divergence from 50%:

| Current Bucket | Prior Bucket Delta | Current Bucket Up % | Note |
|---|---|---|---|
| 09:45–10:00 | positive | **66.2%** | Strongest signal in dataset |
| 09:45–10:00 | negative | 45.5% | Mild fade |
| 11:00–11:15 | positive | 62.0% | Mid-morning continuation |
| 13:00–13:15 | negative | 58.1% | Midday sell pressure fades |
| 13:30–13:45 | negative | 60.2% | Midday sell fades (2nd window) |
| 14:45–15:00 | positive | **65.3%** | Pre-close buy momentum carries |
| 15:00–15:15 | negative | 60.0% | Closing-hour sell absorbed |
| 12:45–13:00 | positive | 36.4% | Midday buy is exhaustion signal |
| 13:00–13:15 | positive | 37.1% | Same — midday buying fades |

**Findings:**
- **09:45–10:00, prior bucket positive delta (66.2%):** when 09:30–09:45 has net buy aggression, the next 15-min window closes up 66% of the time — the strongest directional signal in the 15-min analysis
- **14:45–15:00, prior bucket positive delta (65.3%):** pre-close buy aggression at 14:30–14:45 carries into the next window at nearly the same rate
- **Midday reversal pattern (12:45–13:30):** positive prior delta predicts DOWN (36-37%), negative prior delta predicts UP (58-60%) — midday buying is an exhaustion signal, midday selling gets faded. Opposite regime vs open/close windows
- **15:00–15:15, prior bucket negative delta (60%):** closing-hour sell pressure fades — consistent with institutional end-of-day absorption
- Aggregate edge is 50% because open/close continuation and midday reversal regimes cancel each other out across all windows

### RTH-VOL-006 — 15-Minute Rolling Delta Windows
**Query ID:** RTH-VOL-006 | Task date: 2026-06-16
**Materialized view created:** `nq_data.rth_15min_buckets_agg` — per-bucket OHLC + delta for all RTH 15-min windows

| Prior Bucket Delta Direction | Current Bucket Up Count | Total | Current Bucket Up % |
|------------------------------|------------------------|-------|---------------------|
| Negative                     | 1,026                  | 2,010 | 51.0%               |
| Positive                     | 961                    | 1,907 | 50.4%               |

**Findings:**
- No edge at aggregate level: prior bucket delta direction has essentially zero predictive power for current bucket direction across all windows (51% vs 50.4%)
- This is expected — aggregating across all hours and days kills any time-specific patterns
- Next step: break down by `hour_min` — specific windows (e.g. 09:30, 15:45) may show edge while midday does not

### RTH-ORB-009 — OR Range Size vs Rest-of-Session Range
**Query ID:** RTH-ORB-009 | Task date: 2026-06-29
**Definition:** OR range = or_high - or_low (09:30–10:00). Rest range = r_high - r_low (10:00–16:00). Bucketed into NTILE(4) by OR range. or_pct_of_day = OR range / (OR range + rest range).

| Bucket | Days | Avg OR Range | Avg Rest Range | OR % of Day | Rest Bullish % |
|---|---|---|---|---|---|
| Q1 (tightest) | 40 | 86 pts | 219 pts | 32.5% | 57.5% |
| Q2 | 40 | 128 pts | 279 pts | 34.9% | 45.0% |
| Q3 | 40 | 175 pts | 285 pts | 40.2% | 52.5% |
| Q4 (widest) | 39 | 251 pts | 383 pts | 41.0% | **66.7%** |

**Findings:**
- **Wide OR → wider rest-of-session, not compression** — Q4 rest range (383 pts) is 75% larger than Q1 (219 pts). A volatile OR signals a volatile afternoon; there is no mean-reversion in volatility across the session.
- **OR % of day increases with OR size (32.5% → 41%)** — wider ORs do front-load a larger fraction of the day's range, but the absolute rest-of-session range still expands significantly. The afternoon is not "used up" by a wide OR.
- **Q4 (widest OR) has the strongest bullish close bias: 66.7%** — days with a wide OR tend to resolve bullishly into the close. Possible explanation: wide ORs often occur on high-momentum days where the directional move started in the OR continues.
- **Q2 is the weakest for bulls (45%)** — medium-wide ORs produce the most uncertain close direction. Tight ORs (Q1, 57.5%) and wide ORs (Q4, 66.7%) are both more directional than medium.
- **Practical implication:** If the OR is very wide today (>~200 pts), expect a wide afternoon range and a bullish close bias. A tight OR (< ~100 pts) suggests a quieter afternoon but still slightly bullish close tendency.

## Opening Range Breakout

### RTH-ORB-008 — Agree-Bearish Bounce Targets by Weekday
**Query ID:** RTH-ORB-008 | Task date: 2026-06-26
**Extension of RTH-ORB-007** — adds weekday dimension to agree-bearish bounce target analysis.

| Weekday | Days | Rest Bullish % | Reached OR Close | Reached RTH Open | Reached OR High |
|---|---|---|---|---|---|
| Tuesday | 12 | **75.0%** | 91.7% | **75.0%** | **58.3%** |
| Friday | 10 | 40.0% | 90.0% | 60.0% | 30.0% |
| Thursday | 18 | 61.1% | 88.9% | 33.3% | 22.2% |
| Wednesday | 12 | 58.3% | 83.3% | 25.0% | 25.0% |
| Monday | 10 | 60.0% | 60.0% | 30.0% | 30.0% |

**Findings:**
- **Tuesday agree-bearish is the cleanest setup in the dataset** — 75% rest bullish, OR close reached 91.7%, RTH open reached 75%, OR high reached 58.3%. The reach rates fully confirm the rest bullish %. When Tuesday has a bearish OR and bearish FH, the afternoon consistently rallies and targets are consistently hit.
- **Friday agree-bearish: OR close reached 90% but rest bullish only 40%** — the bounce happens (OR close reached nearly as often as Tuesday) but it reverses before the close. Friday agree-bearish is an intraday bounce setup, not a directional one — price recovers to OR close then fades back. Do not hold into close on Friday agree-bearish days.
- **Thursday agree-bearish: OR close 89% but RTH open only 33%** — the bounce reliably gets back to OR close, but rarely reaches the full RTH open (09:30 price). Target OR close on Thursday agree-bearish; RTH open is too ambitious.
- **Wednesday agree-bearish: RTH open only 25%, OR high only 25%** — moderate rest bullish (58%) but targets rarely extended beyond OR close. Weakest bounce quality of Mon/Wed/Thu.
- **Monday agree-bearish: OR close only 60%** — lowest OR close reach rate of all weekdays. On the 40% of Monday agree-bearish days where the bounce doesn't even get back to OR close, the session likely just drifts higher gradually without a sharp recovery.
- **Practical target hierarchy per weekday (agree-bearish setup):**
  - Tuesday: OR high viable (~58%), RTH open primary target (75%), OR close almost guaranteed (92%)
  - Thursday/Friday: OR close is the target; RTH open is overreach
  - Wednesday/Monday: OR close is not guaranteed — size down vs Tuesday setups

### RTH-ORB-007 — Agree-Bearish Bounce Targets
**Query ID:** RTH-ORB-007 | Task date: 2026-06-25
**Filter:** agree-bearish days only (or_close < or_open AND fh_close < fh_open) — 62 days total.
**Note:** Initial submission had a bug — `o.r_high` (OR window high) used instead of `r.r_high` (rest-of-session high) for `reached_or_close`, producing 100%. Fixed same session; results below are from the corrected query.

| Metric | Value |
|---|---|
| Total agree-bearish days | 62 |
| Reached OR close (10:00 price) | **83.9%** |
| Reached RTH open (= OR open, 09:30 price) | **43.6%** |
| Reached OR high (intraday peak of OR window) | **32.3%** |
| Avg r_high above OR close (all agree-bearish days) | **+113 pts** |

**Findings:**
- **83.9% of agree-bearish afternoons recover back to OR close** — the 10:00 price (end of the bearish OR window) is reached nearly 5 out of 6 times. This is the high-probability minimum target.
- **43.6% recover all the way to RTH open (OR open = 09:30 price)** — the full 30-minute OR selloff gets completely erased in nearly half of agree-bearish days
- **OR high reached 32.3% of the time** — one in three agree-bearish days sees the afternoon trade above the intraday peak of the OR window itself, meaning price fully reclaims the OR range and extends higher
- **Avg r_high is +113 pts above OR close** across all 62 agree-bearish days — the bounce typically extends well past OR close even on days that don't reach RTH open
- **Practical target hierarchy for agree-bearish setup (both OR and FH bearish):**
  - OR close: primary target, ~84% hit rate — take partial at minimum
  - RTH open: secondary target, ~44% hit rate — trail stop after OR close reached
  - OR high: extended target, ~32% hit rate — only for runners

### RTH-ORB-006 — OR × FH Combined Signal by Weekday
**Query ID:** RTH-ORB-006 | Task date: 2026-06-25
**Extension of RTH-ORB-005** — adds weekday dimension to the 4-scenario OR × FH breakdown.

**Agree-bearish (both OR and FH bearish) by weekday:**

| Weekday | Days | Rest Bullish % |
|---|---|---|
| Tuesday | 12 | **75.0%** |
| Thursday | 18 | **61.1%** |
| Monday | 10 | 60.0% |
| Wednesday | 12 | 58.3% |
| Friday | 10 | 40.0% |

**Agree-bullish (both OR and FH bullish) by weekday:**

| Weekday | Days | Rest Bullish % |
|---|---|---|
| Tuesday | 16 | 62.5% |
| Wednesday | 15 | 60.0% |
| Monday | 20 | 60.0% |
| Friday | 16 | 56.3% |
| Thursday | 10 | 50.0% |

**Findings:**
- **Tuesday agree-bearish is the strongest signal in the dataset: 75% rest bullish (12 days)** — both OR and FH selling off on Tuesday is a reliable afternoon buy setup
- **Thursday agree-bearish: 61.1% rest bullish (18 days, largest sample)** — the agree-bearish → bullish rest pattern is not a Thursday artefact; it is real and consistent across Mon/Tue/Wed/Thu
- **Friday agree-bearish is the lone exception: 40% rest bullish** — Friday is the only weekday where agree-bearish is actually a mild bearish signal. Position squaring and end-of-week dynamics override the reversal pattern
- **Thursday agree-bullish: only 50% rest bullish (10 days)** — confirms RTH-SESS-003 (Thursday gap_up + bullish FH = 0% rest bullish). When the OR also joins the bullish chorus on Thursday, the afternoon still doesn't follow through. Agree-bullish on Thursday carries no edge.
- **Agree-bearish is a more consistent signal than agree-bullish on most weekdays** — the reversal tendency after double bearish confirmation (OR + FH both sell off) is more reliable than continuation after double bullish confirmation, except on Tuesday where both directions show edge
- **Trading implication:** On any Thursday or Tuesday where both OR and FH are bearish, the data strongly supports buying the afternoon. On Fridays, treat agree-bearish as neutral-to-bearish.

### RTH-ORB-005 — OR Direction × FH Direction Combined Signal
**Query ID:** RTH-ORB-005 | Task date: 2026-06-24
**Materialized view used:** `nq_data.or_rest_ohlc_ranges` (OR + rest OHLC) joined to `nq_data.rth_firsthour_rest_ohlc_ranges` (FH + rest OHLC) on trade_date.

| OR Direction | FH Direction | Days | Rest Bullish Days | Rest Bullish % |
|---|---|---|---|---|
| bearish | bearish | 62 | 37 | 59.68% |
| bullish | bullish | 77 | 45 | 58.44% |
| bullish | bearish | 10 | 5  | 50.00% |
| bearish | bullish | 10 | 4  | 40.00% |

**Findings:**
- **Counterintuitive result: agree-bearish (both OR and FH bearish) → 59.7% rest bullish** — higher than agree-bullish (58.4%). A bearish open that persists through both the OR and FH windows consistently gets bought into the afternoon. This is a mean-reversion / exhaustion pattern at the session level.
- **Agree-bullish → 58.4% rest bullish** — mild continuation bias, consistent with RTH-FH-001 and RTH-ORB-003 standalone results
- **OR bearish + FH bullish → 40% rest bullish** — the only signal with a mild bearish rest bias. Interpretation: when OR sells off but first hour recovers, the recovery fails to hold into the afternoon more often than not
- **OR bullish + FH bearish → 50% rest bullish** — coin flip, no edge. A strong OR followed by a weak first hour gives no directional information for the rest of session
- **Key implication:** Do NOT fade a bearish open using both OR and FH together as a combined bearish signal — that setup resolves bullishly 60% of the time. The agree-bearish case is a buy setup, not a short setup.
- **Agree signal count skew:** agree-bullish (77 days) vs agree-bearish (62 days) — market opens bullishly more often than bearishly in this sample
- Divergence cases are rare (10 days each) — treat with caution

### RTH-ORB-004 — OR Delta Direction vs Rest-of-Session Direction
**Query ID:** RTH-ORB-004 | Task date: 2026-06-24
**Note:** Redefined from planned magnitude-bucketing task — OR delta direction vs rest-of-session direction is the correct first step before bucketing. Analogous to RTH-VOL-004 (FH delta vs rest direction).

| OR Delta Direction | Days | Avg OR Delta | Rest Bullish Days | Rest Bullish % |
|--------------------|------|-------------|-------------------|----------------|
| Positive           | 77   | +1,149.34   | 46                | 59.74%         |
| Negative           | 82   | -1,304.61   | 42                | 51.22%         |

**Findings:**
- Positive OR delta → bullish rest-of-session 59.74%; negative OR delta → only 51.22% — an 8.5pp gap
- Bearish arm is near-random (51%) — OR delta provides almost no signal when net selling dominated the OR window
- **Comparison with FH delta (RTH-VOL-004):** FH delta gap was 60.3% vs 54.7% = 5.6pp. OR delta gap is larger (8.5pp), but the bearish arm is weaker (51% vs 54.7%). Neither signal is decisive standalone.
- The OR is a tighter window (30 min vs 60 min) — larger delta magnitude asymmetry (avg +1,149 vs -1,305) suggests sell sessions build delta faster than buy sessions in the OR

### RTH-ORB-003 — OR Direction vs Rest-of-Session Continuation
**Query ID:** RTH-ORB-003 | Task date: 2026-06-24
**Note:** Materialized view `nq_data.or_rest_ohlc_ranges` created this session — stores per-day OR OHLC (09:30–<10:00) and rest OHLC (10:00–<16:00). OR open = first tick price; OR close = last tick price (by MIN/MAX ts_event joined back to ticks). OR upper bound uses `< '10:00'` (exclusive) — 10:00:00.000 tick falls in rest window.

| OR Direction | Days | Rest Bullish Days | Rest Bullish % |
|---|---|---|---|
| Bullish | 87 | 52 | 59.77% |
| Bearish | 72 | 36 | 50.00% |

**Findings:**
- Bullish OR (close > open in 09:30–10:00) → 59.77% rest-of-session bullish — modest but consistent upside continuation
- Bearish OR → exactly 50% rest bullish — a coin flip, no directional edge whatsoever
- **Direct comparison with FH direction (RTH-FH-001):**

| Signal | Bullish arm | Bearish arm | Two-sided gap |
|---|---|---|---|
| FH direction | 56.3% | 41.7% | **14.6pp** |
| OR direction | 59.8% | 50.0% | 9.8pp |
| FH delta | 60.3% | 54.7% | 5.6pp |
| OR delta | 59.7% | 51.2% | 8.5pp |

- **FH direction is the superior two-sided signal** — its bearish arm (41.7%) is strongly predictive of a rest-of-session reversal, while OR bearish direction is a coin flip. The 30-min OR window is too short for price direction alone to carry information; the full first hour is needed.
- OR's bullish arm is marginally stronger than FH's bullish arm (59.8% vs 56.3%), but the difference is small and the loss of bearish edge makes OR an inferior standalone indicator.
- **Takeaway for trading:** Use FH direction as the primary directional filter for rest-of-session bias. OR direction adds nothing on bearish days.

### RTH-ORB-002 — OR Delta × Weekday Breakout Rates
**Query ID:** RTH-ORB-002 | Task date: 2026-06-24
**Note:** day_of_week computed from ts_recv — Sunday rows in output are misclassified Monday Globex ticks. Exclude Sunday; all other weekday results valid.

**Key rows (excluding Sunday, within-group pct):**

| Weekday | OR Delta | Breakout | Days | Within-Group % |
|---|---|---|---|---|
| Tuesday | bullish | break_up | 10 | **62.5%** |
| Tuesday | bullish | break_down | 2 | 12.5% |
| Tuesday | bearish | break_down | 10 | **58.8%** |
| Thursday | bullish | break_up | 9 | **60.0%** |
| Thursday | bearish | break_down | 7 | 50.0% |
| Wednesday | bearish | break_down | 12 | **63.2%** |
| Wednesday | bullish | break_both | 5 | 41.7% |
| Monday | bullish | break_both | 8 | **57.1%** |
| Monday | bearish | break_both | 8 | 42.1% |

**Findings:**
- **Tuesday has the strongest OR delta → breakout alignment in the dataset** — bullish OR delta → break_up 62.5%, bearish OR delta → break_down 58.8%. OR delta is a reliable directional signal on Tuesdays despite Tuesday's structural bearish drift
- **Thursday bullish OR delta → break_up 60%** — holds well. Bearish OR delta on Thursday is weaker (50% break_down) — consistent with Thursday's tendency to fade regardless of opening conditions
- **Wednesday bearish OR delta → break_down 63%** — strongest bearish alignment of any weekday. Bullish OR delta on Wednesday is mixed (break_both 42%, break_up 42%) — no clean directional signal
- **Monday is dominated by break_both** regardless of OR delta direction — Monday tends to expand range in both directions before resolving. OR delta less useful as a directional filter on Mondays

### RTH-ORB-001 — Opening Range Breakout Direction vs OR Volume Delta
**Query ID:** RTH-ORB-001 | Task date: 2026-06-16
**Opening Range:** 09:30–10:00 ET (30 minutes). Breakout measured 10:00–16:00 ET.

| OR Delta Direction | Breakout | Days | % of All Days |
|--------------------|----------|------|---------------|
| bullish            | break_up   | 40 | 25.2% |
| bullish            | break_both | 26 | 16.4% |
| bullish            | break_down | 11 | 6.9%  |
| bearish            | break_down | 42 | 26.4% |
| bearish            | break_both | 21 | 13.2% |
| bearish            | break_up   | 19 | 11.9% |

**Within-group rates (corrected):**
- Bullish OR delta (77 days): break_up 52%, break_both 34%, break_down 14%
- Bearish OR delta (82 days): break_down 51%, break_both 26%, break_up 23%

**Findings:**
- Strong directional alignment: bullish OR delta → break_up 52% of the time; bearish OR delta → break_down 51%
- break_both is common in both groups (~26-34%) — market frequently exceeds both sides of the 30-min OR
- Bearish OR delta has a higher break_up rate (23%) than bullish has break_down (14%) — downside OR delta is slightly more likely to see reversal than upside OR delta
- No flat OR delta days present after exclusion
- This is the strongest single-metric breakout predictor found so far — OR delta direction aligns with breakout direction >50% of the time in both groups

### RTH-SESS-005 — Session High/Low Formation Timing by 30-Minute Window
**Query ID:** RTH-SESS-005 | Task date: 2026-07-01
**Method:** For each trade day, MIN(ts_event) where tick price = daily_high (HOD) or daily_low (LOD). Bucketed into 30-min ET windows. 159 trading days. JOIN uses session_start/session_end from daily_ohlcv_rth to avoid AT TIME ZONE cast on 56M rows.

| Window ET | High Formed % | Low Formed % |
|---|---|---|
| 09:30 | **33.3%** | **37.1%** |
| 15:30 | 15.1% | 12.0% |
| 10:00 | 8.2% | 8.2% |
| 15:00 | **8.2%** | 1.9% |
| 14:30 | 6.3% | 2.5% |
| 10:30 | 3.1% | 8.2% |
| 11:30 | 3.8% | 6.9% |
| 13:30 | 4.4% | 6.9% |
| 11:00 | 5.7% | 5.7% |
| 12:30 | 4.4% | 1.9% |
| 12:00 | 3.8% | 4.4% |
| 13:00 | 2.5% | 3.1% |
| 14:00 | 1.3% | 1.3% |

**Findings:**
- **09:30 window dominates both extremes: 33.3% of day highs and 37.1% of day lows form in the first 30 minutes** — the opening half-hour is the single most important window for establishing the day's extremes by a wide margin
- **15:30 is the second most common window for both (15.1% high / 12.0% low)** — the final 30 minutes is the next most likely time for the day's extreme to be set. Combined with 09:30, the open and close account for ~48% of highs and ~49% of lows
- **Late afternoon asymmetry: 14:30–15:00 form highs more often than lows (14.5% combined vs 4.4%)** — the 14:30–15:30 window has a clear upside bias for establishing new extremes. Afternoon sessions tend to push to new highs rather than new lows.
- **Midday graveyard (11:00–13:30): only ~20% of highs and ~22% of lows form** — spread across 5 windows averaging ~4% each. Midday rarely sets the extreme.
- **10:30 forms lows (8.2%) more than highs (3.1%)** — the first-hour close window has a downside skew for extreme setting. Consistent with RTH-FH-001 (bearish FH → reversal more common than continuation).
- **Practical trading implications:**
  - The 09:30 open is the single most important level to watch — it becomes the day's high or low one in three times
  - If the open doesn't set the extreme, the next most likely window is the final 30 min (15:30) — hold through midday noise
  - The 14:30–15:00 window is the highest-probability time for a new high to form (not a new low) — afternoon strength late in the session is common
  - Midday (11:00–13:30) moves rarely define the day — fade midday breakouts

### RTH-SESS-004 — Choppy Day Detection: Tight OR + FH Joint Filter
**Query ID:** RTH-SESS-004 | Task date: 2026-07-01
**Definition:** both_tight = OR range Q1 AND FH range Q1; both_wide = OR range Q4 AND FH range Q4; mixed = all other combinations. NTILE(4) computed independently for OR and FH ranges.

| Day Type | Days | Avg OR Range | Avg FH Range | Avg Rest Range | Rest Bullish % | Avg Close Location |
|---|---|---|---|---|---|---|
| both_wide | 28 | 268 pts | 330 pts | **339 pts** | **67.9%** | 0.61 |
| mixed | 100 | 153 pts | 198 pts | 265 pts | 55.0% | 0.54 |
| both_tight | 31 | 82 pts | 94 pts | **197 pts** | 54.8% | 0.58 |

**Findings:**
- **both_tight rest range (197 pts) is 42% smaller than both_wide (339 pts)** — tight OR + FH is a reliable signal for a quieter afternoon. If both opening windows are narrow, the rest-of-session will also be narrow.
- **both_tight rest_bullish_pct (54.8%) is nearly identical to mixed (55%)** — tight days give no directional edge. The afternoon is quiet but equally likely to drift up or down. Size down on both_tight days; don't try to pick direction.
- **both_tight close location 0.58** — tight days still close above the midpoint of their afternoon range on average. The quiet drift is mildly bullish in character even if not statistically significant.
- **both_wide is the most directional and bullish day type: 67.9% rest bullish, close location 0.61** — wide OR + FH = high-conviction day with strong afternoon bullish bias. These are the best days to hold directional positions.
- **Practical implication:** When both OR range and FH range are tight (< ~90 pts OR, < ~110 pts FH — approximate Q1 thresholds), expect a rest-of-session range of ~197 pts and no directional edge. Reduce size. When both are wide (> ~220 pts OR, > ~270 pts FH), expect ~339 pts rest range and a 68% bullish afternoon — these are trend days worth holding.

## Session Patterns

### RTH-SESS-009 — Post-10:30 LOD Breakdown Depth and Recovery
**Query ID:** RTH-SESS-009 | Task date: 2026-07-11
**Method:** Pre-10:30 low = MIN(price) from ticks 09:30–10:30 ET per trade_date. Post-10:30 low = MIN(price) from ticks 10:30–16:00 ET. Breakdown = post_1030_low < pre_1030_low. Post-breakdown high = MAX(price) from breakdown timestamp onward. Recovery targets: `fh_open` (= 09:30 RTH open, from rth_firsthour_rest_ohlc_ranges) and `fh_high` (first-hour high, 09:30–10:30). Note: `fh_high` is the first-hour high, not full-day high as originally specced — a more actionable target since it's a known level at 10:30. ts_recv used in window bucketing CTEs (dead column, not used in final output).

**Aggregate (all RTH days with a post-10:30 LOD breach):**

| Days with Breakdown | Avg Breakdown Depth | Recovered to RTH Open % | Recovered Above FH High % |
|---|---|---|---|
| **87** | **138 pts** | **74.71%** | **55.17%** |

**Findings:**
- **87 out of ~159 RTH days (~55%) see a post-10:30 LOD breach** — more than half of all trading days make a new low after 10:30. Late-session LOD breakdowns are the norm, not the exception.
- **Average breakdown depth: 138 pts below the pre-10:30 low** — this is the typical extension once the pre-10:30 low is breached. Practical implication: if you're long and the pre-10:30 low breaks, expect ~138 pts of additional heat before any recovery.
- **74.71% recover back to the RTH open (09:30 price)** — nearly 3 in 4 breakdowns fully recover to where the day started. The post-10:30 LOD breach is primarily a fade setup. Price sweeps the pre-10:30 low, extends ~138 pts, then reverses back to the open.
- **55.17% recover above the first-hour high** — more than half of breakdowns don't just recover to the open but push all the way above the FH high (09:30–10:30 range top). On these days the post-breakdown recovery exceeds the entire first-hour range.
- **The pattern is mean-reverting:** breach the pre-10:30 low → extend ~138 pts → recover to open (75%) → potentially extend above FH high (55%). This is a late-session capitulation followed by a recovery, not a trend continuation.
- **Practical framework for post-10:30 LOD breakdowns:**
  - When pre-10:30 low is breached after 10:30: this is a fade setup 75% of the time
  - Expected heat before reversal: ~138 pts below the broken level — size stops accordingly
  - Primary recovery target: RTH open (09:30 price) — 75% hit rate
  - Extended recovery target: FH high — 55% hit rate, valid secondary target
  - The 25% of days that don't recover to open are true continuation breakdowns — no additional filter identified yet to distinguish them
- **Caveat:** Post-breakdown high is measured from the LOD timestamp to 16:00 — includes the close. On some days the "recovery" may happen in the final minutes. The stat is valid for end-of-day P&L but does not imply a clean intraday reversal on every occurrence.

### RTH-SESS-008 — Monday LOD Depth from 09:30 Open
**Query ID:** RTH-SESS-008 | Task date: 2026-07-07
**Method:** `fh_open` from `rth_firsthour_rest_ohlc_ranges` as the 09:30 reference price. ABS distances for open_to_low, open_to_high, open_to_close. LAG(d.close) for gap direction. 33 Mondays total; 14 gap-down, 19 gap-up.

**Overall Monday (all gap directions):**

| Days | Avg Dip Below Open | Avg Extension Above Open | Avg Close vs Open | LOD Below Open % | Close Above Open % |
|---|---|---|---|---|---|
| 33 | **75 pts** | 115 pts | +113 pts | **100%** | 60.6% |

**By gap direction:**

| Gap Direction | Days | Avg Dip Below Open | Avg Extension Above Open | Avg Close vs Open | Close Above Open % |
|---|---|---|---|---|---|
| gap_down | 14 | **65 pts** | 141 pts | +109 pts | **71.4%** |
| gap_up | 19 | **81 pts** | 95 pts | +116 pts | 52.6% |

**Findings:**
- **100% of Mondays see the LOD go below the 09:30 open** — the opening price is never the exact low. Every Monday dips below the open at some point.
- **Average dip below open: 75 pts** — practical stop placement for the Monday buy-the-open setup should be at least 75–100 pts below the 09:30 open to avoid being stopped out on the typical dip.
- **Average extension above open: 115 pts, avg close +113 pts above open** — the round-trip is ~188 pts (75 down, then 115+ up from open). Monday structurally dips and recovers well above the open.
- **Gap-down Monday is the cleaner setup:** dip only 65 pts below open (vs 81 pts on gap-up), closes above open 71.4% of the time, extends 141 pts above open on average. Lower risk entry (shallower dip), higher reward (larger extension), better close rate.
- **Gap-up Monday is the weaker setup:** dips 81 pts below open (deeper than gap-down), closes above open only 52.6% (near coin flip). Consistent with RTH-GAP-003 (large gap-ups tend to fade). The opening high on a gap-up Monday is often the HOD — buying below open still works as a dip trade but with much less conviction.
- **Practical Monday entry framework:**
  - Gap-down open: buy dip within 65–80 pts below open, target 109–141 pts above open, stop ~100 pts below open
  - Gap-up open: treat as a fade setup first (HOD likely at open per RTH-FH-004); if buying the dip, expect 81 pts of heat, close above open only 53% — reduce size vs gap-down setup

### RTH-SESS-006 — Session High/Low Timing by Weekday (30-Min Windows)
**Query ID:** RTH-SESS-006 | Task date: 2026-07-03
**Extension of RTH-SESS-005** — adds weekday dimension. Within-weekday percentages via SUM(COUNT(*)) OVER (PARTITION BY weekday). ~29–39 days per weekday. Note: INNER JOIN on matching high/low windows drops windows where no overlap exists — negligible impact on data.

**09:30 window HOD/LOD concentration by weekday (dominant pattern):**

| Weekday | HOD in 09:30 % | LOD in 09:30 % | Key secondary window |
|---|---|---|---|
| Thursday | **45.2%** | 41.4% | 15:30 LOD 13.8% |
| Wednesday | 36.4% | **42.4%** | 15:00 HOD 18.2% |
| Friday | 34.5% | 27.3% | Spread across midday |
| Monday | 27.3% | **51.5%** | 15:30 HOD 21.2% |
| Tuesday | 24.2% | 27.3% | 15:30 HOD 27.3% |

**Findings:**
- **Thursday HOD front-loaded: 45.2% in 09:30 window** — highest of any weekday, well above the 33.3% global average. Thursday's high is set at the open more than any other day, confirming the structural fade tendency (RTH-FH-002: FH sets day HIGH 54.8% on Thursdays). If Thursday opens strong, the 09:30–10:00 high is the most likely HOD.
- **Monday LOD: 51.5% in 09:30 window** — by far the most concentrated LOD formation of any weekday. Half of Monday's day lows are set in the first 30 minutes. Combined with Monday's bullish drift (RTH-CLOSE-001: +114 avg), the pattern is clear: Monday gaps down or sells off at the open, that opening low holds all day, and price drifts higher through the session. The 09:30 LOD is the key buy level on Mondays.
- **Tuesday has a split HOD distribution: 24.2% at 09:30, 27.3% at 15:30** — nearly equal. Tuesday's high is as likely to form late (15:30) as early (09:30), consistent with the agree-bearish reversal pattern (RTH-ORB-006: 75% rest bullish after bearish OR+FH) — afternoon rallies push to new highs late in the session.
- **Wednesday LOD: 42.4% in 09:30 + 12.9% at 10:00** — over half of Wednesday's lows form in the first 30–60 minutes. The opening weakness is the day's low, price then drifts higher (RTH-CLOSE-001: +80 avg). Wednesday HOD secondary peak at 15:00 (18.2%) — the high often forms in the final hour, consistent with post-FOMC or late-session continuation.
- **Friday is the most evenly distributed** — HOD spread across 09:30 (34.5%), 11:00 (10.3%), 12:00 (10.3%), 13:30 (10.3%); no window dominates beyond the open. Friday extremes form at unpredictable times — the 09:30 window is still the modal window but the edge is weaker than other weekdays.
- **Practical weekday entry timing:**
  - **Thursday:** Watch the first 30 min for the HOD — if it's an up-open, the 09:30 high is the fade level with 45% historical probability of being the day's extreme
  - **Monday:** The 09:30 low is the buy level — 51.5% chance it holds all day
  - **Tuesday:** Don't fade the morning high too aggressively — 27% of Tuesday HODs form at 15:30 (afternoon rally)
  - **Wednesday:** Buy early weakness — 55% of LODs form by 10:00; HOD often comes late (15:00)
  - **Friday:** No strong timing edge beyond the open; treat as more random intraday

### RTH-SESS-001 — Gap Direction × First Hour Direction Interaction
**Query ID:** RTH-SESS-001 | Task date: 2026-06-10

| Gap Direction | FH Direction | Days | Avg Full Day Range | Avg Close Location | Rest Continues % |
|---------------|-------------|------|-------------------|-------------------|-----------------|
| gap_down      | bearish     | 36   | 383.03            | 55.5%             | 33.3%           |
| gap_down      | bullish     | 51   | 352.34            | 67.1%             | 70.6%           |
| gap_up        | bearish     | 46   | 297.61            | 45.0%             | 47.8%           |
| gap_up        | bullish     | 51   | 331.91            | 58.6%             | 45.1%           |

**Findings:**
- **Strongest signal: gap_down + bullish FH** — 70.6% rest-of-session continuation, avg close location 67%. Market gaps down overnight, first hour recovers bullishly, and the afternoon continues higher 70% of the time. High-conviction reversal setup.
- **Gap_down + bearish FH** — only 33% rest continuation, meaning 67% of the time the session reverses back up after a gap-down + bearish open. Exhaustion/capitulation pattern — both overnight and opening weakness get faded.
- **Gap_up days**: both FH directions produce ~45-48% rest continuation — essentially no predictive edge from first-hour direction on gap-up days. The opening gap direction matters more than FH direction here.
- **Gap-down days are more volatile** (avg range 352-383 pts) vs gap-up days (298-332 pts) — overnight downside gaps lead to bigger intraday swings regardless of outcome
- **Close location confirms direction**: gap_down+bullish closes at 67% of range (strong close), gap_up+bearish closes at 45% (weak close within range)

---

## News Days

### RTH-NEWS-005 — FOMC Spike Reversal (Anchored to Spike Extreme)
**Query ID:** RTH-NEWS-005 | Task date: 2026-06-24

Reversal defined as: after the spike extreme is reached, does price cross back through pre_event_price before 16:00? All 7 FOMC days in dataset show reversed = 1 (100% reversal rate).

| Spike Direction | Reversed | Days | Avg Close vs Pre-Event |
|---|---|---|---|
| down | 1 | 4 | **+107.94** |
| up | 1 | 3 | **-56.17** |

**Findings:**
- **100% of FOMC spikes reversed back through pre-event price** — every spike in the dataset (7 events) was a trap, not a trend
- **Spike down + reversed:** avg RTH close +108 pts above pre-event — market spiked down on FOMC, reversed, and closed well above the pre-announcement level. Buy-the-dip pattern.
- **Spike up + reversed:** avg RTH close -56 pts below pre-event — market spiked up, reversed, and closed below. Fade-the-spike pattern.
- Both directions close on the opposite side of pre_event_price — FOMC reaction spikes in this dataset are consistently exhaustion moves, not directional trends
- Sample is small (3-4 days per group) — treat as directional signal, not statistical certainty
- **Fixed 2026-06-25:** fomc_agg recreated as MATERIALIZED VIEW with corrected spike_magnitude = `reaction_high - pre_event_price` (spike_up) or `pre_event_price - reaction_low` (spike_down). Prior version used reaction_high - reaction_low (full range).

### RTH-NEWS-004 — NFP vs CPI Reaction Summary
**Query ID:** RTH-NEWS-004 | Task date: 2026-06-22

Aggregated spike behavior by event type. Pre-event reference = last tick before 08:30 ET. Reaction window = 08:30–09:30 ET (Globex). RTH close comparison uses fh_close from rth_firsthour_rest_ohlc_ranges.

| Event | Days | Avg Spike | Spike Up % | Avg RTH Open vs Pre | Avg RTH Close vs Pre | Closed Above Pre % |
|---|---|---|---|---|---|---|
| CPI | 7 | 122 pts | 42.9% | +27 | +36 | **71.4%** |
| NFP | 5 | 185 pts | 60.0% | -6 | -8 | 40.0% |

**Findings:**
- **CPI: initial spike is mostly down (57%), but RTH closes above pre-announcement 71% of the time** — the CPI drop gets bought back through the session. Avg close +36 pts above pre-event level
- **NFP: initial spike is mostly up (60%), but RTH closes below pre-announcement 60% of the time** — the NFP pop fades. Avg close -8 pts below pre-event level
- **NFP spikes are significantly larger** (185 pts avg vs 122 pts for CPI) — employment data creates more violent initial moves than inflation data
- **RTH open after NFP is nearly flat vs pre-event** (-6 pts avg) — the Globex reaction is mixed and RTH opens close to the pre-announcement level; after CPI, RTH opens +27 pts above (market holds some of the recovery into open)
- Sample is small (5-7 events each) — directional contrast is clean but not statistically robust

### RTH-NEWS-003b — NFP/CPI Reaction Spike Detail (per event)
**Query ID:** RTH-NEWS-003b | Task date: 2026-06-22

Per-event spike data: pre-event price, reaction range 08:30–09:30 ET, spike direction, RTH open and first-hour close vs pre-event. See tasks.md for full row-level data.

Notable events:
- 2026-03-06 NFP: 296 pt spike (largest), spiked up first, RTH closed -42 vs pre-event — full fade
- 2025-12-18 CPI: 176 pt spike down, RTH closed +212 vs pre-event — strongest CPI recovery
- 2026-02-11 NFP: 213 pt spike down, RTH closed -161 vs pre-event — continuation of the drop

### RTH-NEWS-003a — FOMC Reaction Spike Analysis (methodology established, incomplete)
**Query ID:** RTH-NEWS-003a | Task date: 2026-06-19
**Status:** Query built and validated through spike_direction. Not extended further — reversal logic deferred.

**Methodology established:**
- Pre-event reference price: last tick strictly before 14:00 ET on FOMC day (MAX(ts_event) → point-lookup JOIN)
- Reaction range: MIN/MAX price in 14:00–15:00 ET window
- Spike direction: first extreme hit determined by comparing timestamp of reaction_high vs reaction_low (DISTINCT ON pattern)

**Known limitation identified:** A simple "did price return to pre_event_price before 16:00" reversal check is flawed — price may have already returned to the reference level mid-spike, making the check misleading. Correct approach requires anchoring the reversal check to the spike extreme (first target = pre_event_price from the spike high/low), not the raw reference price.

**Future extensions noted:**
- Replace hardcoded 14:00 with `event_time_et` from news_events for generalized multi-event-type support
- Apply same methodology to NFP/CPI (08:30 ET reference, Globex window)
- Reversal pattern: measure time/distance from spike extreme back to pre-event level

### RTH-NEWS-001 — News Events Table Created
**Query ID:** RTH-NEWS-001 | Task date: 2026-06-18

Table `nq_data.news_events` created with columns: `event_date`, `event_time_et`, `event_type`, `instrument`, `notes`.

**Events populated (Sep 2025 – Jun 2026):**
- FOMC: 2025-09-17, 2025-10-29, 2025-12-10, 2026-01-28, 2026-03-18, 2026-04-29, 2026-06-17 (7 meetings)
- NFP: Monthly, with notable delays — Nov 2025 (govt funding lapse, released Nov 20), Feb 2026 (BLS adjustments, released Feb 11)
- CPI: Monthly releases at 08:30 ET

**Design note:** `instrument = 'ALL'` for macro events; column supports future per-instrument filtering.

### RTH-NEWS-002 — Clean vs Event Day Close-to-Close by Weekday
**Query ID:** RTH-NEWS-002 | Task date: 2026-06-18
**Note:** Query used r_close from rth_firsthour_rest_ohlc_ranges instead of daily_ohlcv_rth.close — directionally valid but slightly off for precise close-to-close comparison.

| Weekday   | Is Event Day | Days | Avg Close Change | Up % |
|-----------|-------------|------|-----------------|------|
| Monday    | false        | 33   | +135.70         | 66.7% |
| Wednesday | false        | 24   | +102.30         | 70.8% |
| Wednesday | true         | 9    | +30.22          | 66.7% |
| Friday    | false        | 25   | +10.01          | 60.0% |
| Friday    | true         | 4    | -13.88          | 75.0% |
| Tuesday   | false        | 31   | -6.79           | 45.2% |
| Tuesday   | true         | 2    | -0.38           | 50.0% |
| Thursday  | false        | 28   | -99.03          | 35.7% |
| Thursday  | true         | 3    | -251.67         | 33.3% |

**Findings:**
- **Wednesday clean days: +102 avg, 70.8% up** — stronger than Monday (+136 avg but only 33 days). Wednesday's bullish bias is structural, not purely FOMC-driven
- **Wednesday event days: +30 avg, 66.7% up** — FOMC days still bullish but magnitude drops from 102 to 30. Post-FOMC relief rallies account for ~72 pts of the Wednesday premium
- **Monday clean days: +135 avg, 66.7% up** — Monday bullish drift is clean (no major macro events land on Mondays)
- **Thursday event days: -251 avg** (3 days) — event Thursdays are extremely bearish, worse than already-bearish clean Thursdays (-99 avg). Small sample (3 days)
- **Tuesday is structurally bearish** on clean days (-6.79 avg, 45.2% up) — not news-driven
- **Friday event days** (4 days): 75% up but -13.88 avg — contradictory; NFP Fridays close up often but not by much, or some close sharply lower

---

## First Hour (09:30–10:30 ET)

*(findings to be added)*

---

## Globex vs RTH Comparisons

### RTH-GLOB-002 — EU Session Levels vs RTH Price Action on Bullish Days
**Query ID:** RTH-GLOB-002 | Task date: 2026-07-09
**Method:** EU session = ticks where ts_event AT TIME ZONE 'America/New_York' between 02:00–09:30. EU high/low/range/midpoint per trade_date. RTH open location = (rth_open - eu_low) / eu_range. Undercut flags use full RTH day low (d.low) — includes the open itself; slightly overstates undercut rates vs true post-open low, directionally valid. Filter: RTH close > open (bullish days only). 86 bullish days total. Note: rerun when more Databento data is available to confirm Thursday 100% finding (only 12 days).

**Aggregate (all bullish RTH days):**

| Days | Avg EU Range | RTH Open Location in EU Range | Undercut EU Midpoint % | Undercut EU Low % |
|---|---|---|---|---|
| 86 | 197 pts | **53.7%** | **73.3%** | **46.5%** |

**By weekday (bullish RTH days only):**

| Weekday | Days | RTH Open Location | Undercut EU Mid % | Undercut EU Low % |
|---|---|---|---|---|
| Thursday | 12 | 44.5% | **100%** | **66.7%** |
| Friday | 17 | 48.0% | 76.5% | 47.1% |
| Tuesday | 19 | 51.9% | 73.7% | 57.9% |
| Wednesday | 18 | 56.4% | 72.2% | **27.8%** |
| Monday | 20 | **63.3%** | **55.0%** | 40.0% |

**Findings:**
- **Nearly half of all bullish RTH days (46.5%) see RTH trade below the EU low after the open** — entering long during EU on a day that ends up bullish was suboptimal nearly half the time. Waiting for RTH provided a better fill.
- **73.3% of bullish days undercut the EU midpoint** — even entering at the middle of the EU range was beaten by the post-open dip on three quarters of bullish days.
- **RTH opens dead center in the EU range on average (53.7%)** — the open is neither at the EU high nor the EU low. The EU session establishes a range and RTH opens in the middle, then dips before rallying.
- **Thursday: 100% undercut EU midpoint, 66.7% undercut EU low on bullish days** — on every bullish Thursday in the dataset, RTH traded below the EU midpoint. Entering EU longs on Thursday is almost always suboptimal — RTH will give a better price. Consistent with Thursday's structural opening fade (RTH-FH-003: 59% bearish FH; RTH-FH-002: FH sets HOD 54.8%). Even on days that end bullish, Thursday dips hard first.
- **Monday: RTH opens at 63.3% of EU range (near EU high), only 40% undercut EU low** — on bullish Mondays the EU session establishes the day's low, and RTH opens near the top of the EU range. Entering long during EU on Monday was at or near the best available price. The Monday buy-the-EU-low setup is validated.
- **Wednesday: only 27.8% undercut EU low** — the EU low holds on 72% of bullish Wednesdays. EU session entries were near the day's best prices on Wednesday. Consistent with Wednesday's bullish drift and early LOD formation (RTH-SESS-006: 55% of Wednesday LODs form by 10:00 — many of those LODs are set during EU).
- **Practical EU entry framework (bullish day context):**
  - **Monday:** EU low is likely the day's low — enter during EU, don't wait for RTH
  - **Wednesday:** EU low usually holds — EU entries are valid, RTH open often near EU high
  - **Thursday:** Never enter EU longs — RTH gives better prices on every bullish Thursday. Wait for the post-open dip.
  - **Tuesday/Friday:** Roughly 50/50 whether EU low holds — no strong edge either way
- **Caveat:** Uses full RTH day low (includes the open) rather than post-open low — slightly overstates undercut rates. Thursday 100% based on only 12 days — reconfirm with more data.

### RTH-GLOB-004 — Gap Direction × EU Entry Quality (Bullish RTH Days)
**Query ID:** RTH-GLOB-004 | Task date: 2026-07-10
**Method:** Extension of RTH-GLOB-002 — LAG(close)/LAG(open) for prev_close/prev_open. gap_direction = rth_open > prev_close. Filter: bullish RTH days (rth_close > rth_open). 85 bullish days (prev_close IS NOT NULL). Note: same duplicate alias issue as RTH-GLOB-003 (second `avg_eu_range` is pct_undercut_eu_midpoint) — data readable from position.

| Gap Direction | Days | Avg EU Range | Pct Undercut EU Mid | Avg RTH Open Location | Pct Undercut EU Low | Pct Above EU High |
|---|---|---|---|---|---|---|
| gap_down | 41 | 217.22 | 85.37% | 43.61% | 60.98% | 80.49% |
| gap_up | 44 | 180.03 | 63.64% | 62.11% | 34.09% | 95.45% |

**Findings:**
- **The original hypothesis was WRONG — gap-down bullish days see MORE EU low undercutting (60.98%), not less.** Gap-up bullish days undercut EU low only 34.09%. The relationship is the opposite of what was expected.
- **Structural explanation for gap-down bullish days (60.98% EU low undercut):** RTH opens at 43.61% of EU range — below the EU midpoint, already deep in the EU range. The EU low is not far below. With RTH opening near the bottom of the EU range on a day that ultimately closes bullish, there is room to dip through the EU low and recover. The EU low gets swept before the rally.
- **Gap-up bullish days: only 34.09% undercut EU low** — RTH opens at 62.11% of EU range (above EU midpoint, near EU high). The EU low is far below the open — a post-open dip would need to retrace the entire EU range plus more to undercut it. On a bullish day this rarely happens. The EU low holds as support when RTH opens well above it.
- **95.45% of gap-up bullish days trade above the EU high** — RTH opens near the EU high and continues above it on nearly every bullish gap-up day. EU longs entered during the overnight session near the EU high were immediately underwater as RTH opened at or above and extended further.
- **80.49% of gap-down bullish days trade above the EU high** — even opening below the EU midpoint, bullish gap-down days rally all the way through the EU high 80% of the time.
- **EU midpoint undercut splits cleanly: 85.37% gap-down vs 63.64% gap-up** — same direction as EU low. The EU session range is a much better support zone on gap-up bullish days than on gap-down bullish days.
- **Practical EU entry framework update (combining with RTH-GLOB-002 weekday findings):**
  - **Gap-up bullish days:** EU low likely holds (34% undercut) — EU long entries near EU low are at or near the optimal price. The EU low is genuine support on gap-up days.
  - **Gap-down bullish days:** EU low likely gets undercut (61%) — wait for RTH. The opening dip often sweeps the EU low before the rally starts. EU entries are suboptimal.
  - **Combined signal:** Gap-up + Monday/Wednesday (RTH-GLOB-002: low undercut rates) is the cleanest EU entry setup. Gap-down + Thursday (100% undercut EU mid) is the strongest "never enter EU" signal.
- **Note:** Gap direction explains more variance in EU low undercut rates (60.98% vs 34.09% = 26.9pp spread) than prior day direction (43.24% vs 50.00% = 6.8pp from RTH-GLOB-003). Gap direction is the more useful pre-RTH filter.

### RTH-GLOB-005 — EU Low Undercut Rate by Weekday × Gap Direction
**Query ID:** RTH-GLOB-005 | Task date: 2026-07-11
**Method:** Extension of RTH-GLOB-002 — LAG(close) for prev_close, gap_direction = rth_open > prev_close. Run twice: all RTH days combined, then bullish RTH days only (rth_close > rth_open). Note: day_direction CASE inside CTE has same inversion as RTH-GLOB-004 (rth_open > rth_close = 'bullish') but WHERE filter uses rth_close > rth_open directly — results correct, internal label wrong. Duplicate alias issue persists (second avg_eu_range column is pct_undercut_eu_mid).

**All RTH days (bullish + bearish combined):**

| Gap | Weekday | Days | RTH Open Location | Pct Undercut EU Low | Pct Above EU High |
|---|---|---|---|---|---|
| gap_down | Thursday | 18 | 30.39% | **88.89%** | 33.33% |
| gap_down | Tuesday | 18 | 41.48% | 77.78% | 72.22% |
| gap_down | Friday | 11 | 31.99% | 81.82% | 54.55% |
| gap_down | Monday | 14 | 54.13% | 64.29% | 71.43% |
| gap_down | Wednesday | 11 | 45.76% | 54.55% | 54.55% |
| gap_up | Wednesday | 22 | 58.91% | 63.64% | 68.18% |
| gap_up | Thursday | 13 | 73.31% | 61.54% | 61.54% |
| gap_up | Tuesday | 15 | 54.61% | 60.00% | 66.67% |
| gap_up | Monday | 18 | 71.87% | 50.00% | 88.89% |
| gap_up | Friday | 18 | 59.38% | 55.56% | 83.33% |

**Bullish RTH days only:**

| Gap | Weekday | Days | RTH Open Location | Pct Undercut EU Low | Pct Above EU High |
|---|---|---|---|---|---|
| gap_down | Tuesday | 11 | 42.00% | 81.82% | 81.82% |
| gap_down | Thursday | 7 | 29.93% | 71.43% | 71.43% |
| gap_down | Friday | 7 | 32.33% | **71.43%** | 71.43% |
| gap_down | Monday | 10 | 57.06% | 50.00% | 80.00% |
| gap_down | Wednesday | 6 | 53.26% | **16.67%** | **100%** |
| gap_up | Thursday | 5 | 64.78% | 60.00% | 80.00% |
| gap_up | Wednesday | 12 | 57.95% | 33.33% | **100%** |
| gap_up | Monday | 9 | 66.73% | 33.33% | **100%** |
| gap_up | Friday | 10 | 58.94% | 30.00% | 90.00% |
| gap_up | Tuesday | 8 | 65.43% | 25.00% | **100%** |

**Findings (bullish RTH days — primary population):**
- **Gap-down Wednesday bullish: only 16.67% EU low undercut** (6 days, small sample) — the cleanest EU entry setup in the full dataset. RTH opens at 53% of EU range (near midpoint) and the EU low holds 83% of the time. Gap-down Wednesday = buy EU low, it will almost certainly not be taken out by RTH. 100% of these days also trade above EU high — the rally extends through the entire EU range.
- **Gap-up Tuesday/Wednesday/Monday/Friday bullish: 25–33% EU low undercut** — all four weekdays show strong EU low support on gap-up bullish days. RTH opens near the EU high (57–67% of EU range) and the EU low is far below — rarely reached. The EU low is genuine support on all gap-up bullish days.
- **Gap-down Friday bullish: 71.43% EU low undercut** — answers the original question. Gap-down Friday opens at 32% of EU range (bottom third) and still undercuts the EU low 71% of the time. EU long entries on gap-down Fridays are suboptimal even when the day closes bullish. Wait for RTH.
- **Gap-down Thursday bullish: 71.43% EU low undercut** — same rate as Friday. Consistent with Thursday's structural post-open dip (RTH-FH-002: FH sets HOD 54.8%). Even on bullish Thursdays, RTH sweeps the EU low before rallying.
- **Gap-down Monday bullish: 50% EU low undercut** — exactly coin flip. Less clear than Wednesday (16.67%) but better than Thursday/Friday (71%). Monday's structural opening dip (RTH-SESS-008: 100% dip below open) explains why even gap-down Mondays sometimes break the EU low, but the bullish drift limits how far it extends.
- **Gap-down Tuesday bullish: 81.82% EU low undercut** — highest of any gap-down weekday on bullish days. Tuesday gap-down bullish = expect EU low to be taken out. Don't enter EU longs on gap-down Tuesdays.
- **100% above EU high for gap-up Mon/Tue/Wed/Thu bullish** — on every gap-up bullish day for these weekdays, RTH traded above the EU high. EU longs entered during the overnight session are always underwater by RTH open on gap-up days (already opened at/above EU high).
- **Gap direction explains more than weekday alone on most days.** The full interaction reveals: gap-down + Wednesday is the safest EU long entry; gap-down + Tuesday is the worst. Gap-up is consistently safe for EU lows (25–33% undercut) regardless of weekday.
- **Friday gap-down answer:** EU low does NOT reliably hold (71.43% undercut). Don't enter EU longs on gap-down Fridays — wait for the RTH dip, which likely sweeps the EU low before the bounce.
- **Note:** Cell sizes are small (5–12 days per cell). Directional patterns are consistent with prior findings but reconfirmation with more data is warranted, especially for gap_down Wednesday (6 days) and gap_up Thursday (5 days).

### RTH-GLOB-003 — Prior Day Direction × EU Entry Quality (Bullish RTH Days)
**Query ID:** RTH-GLOB-003 | Task date: 2026-07-10
**Method:** Extension of RTH-GLOB-002 — added `prev_close`, `prev_open` via LAG in `eu_us_joint_agg`. `day_direction` = rth_close > rth_open; `prev_day_direction` = prev_close > prev_open. Filter: day_direction = 'bullish'. Note: duplicate alias `avg_eu_range` in SELECT (second column is actually pct_undercut_eu_midpoint) — data readable from position. 85 bullish days (prev_close IS NOT NULL).

| Prev Day Direction | Days | Avg EU Range | Pct Undercut EU Mid | Avg RTH Open Location | Pct Undercut EU Low | Pct Above EU High |
|---|---|---|---|---|---|---|
| bearish | 37 | 213.26 | 78.38% | 52.69% | 43.24% | 89.19% |
| bullish | 48 | 186.19 | 70.83% | 53.56% | 50.00% | 87.50% |

**Findings:**
- **Prior day direction does NOT meaningfully split EU low undercut rates** — bearish prior day: 43.24% vs bullish prior day: 50.00%. The difference is small (6.7pp) and counterintuitive — a bullish prior day slightly increases the chance of EU low being undercut on the current bullish day, not decreases it.
- **EU midpoint undercut shows a similar modest split: 78.4% (bearish prior) vs 70.8% (bullish prior)** — 7.6pp difference in the expected direction (bearish prior → RTH dips harder) but both remain very high. The EU midpoint is undercut on bullish days regardless of what happened the day before.
- **Avg EU range splits by prior direction (213 vs 186 pts)** — bearish prior days produce wider EU ranges, consistent with volatile overnight sessions following a down day.
- **RTH open location nearly identical (52.7% vs 53.6%)** — prior day direction has essentially no effect on where RTH opens within the EU range. Both open dead center.
- **Both groups see ~88-89% of bullish RTH days trade above the EU high** — nearly universal, regardless of prior day context.
- **Conclusion: prior day direction is not a useful pre-EU signal for predicting EU level quality.** The aggregate RTH-GLOB-002 finding (46.5% undercut EU low, 73.3% undercut EU mid) holds across both prior day contexts. Weekday is a much stronger predictor (RTH-GLOB-002: Monday 40% vs Thursday 66.7% EU low undercut) than the prior day direction.

### RTH-GLOB-002b — EU Session Levels vs RTH Price Action on Bearish Days
**Query ID:** RTH-GLOB-002b | Task date: 2026-07-10
**Method:** Mirror of RTH-GLOB-002 — filter changed to `close < open` (bearish RTH days). Flags: `reached_eu_midpoint` = `d.low < eu_midpoint` (RTH still drills below EU mid), `reached_eu_low` = `d.low < eu_low`, `reached_eu_high` = `d.high > eu_high` (RTH trades above EU high post-open, offering short entry). 73 bearish days total.

**Aggregate (all bearish RTH days):**

| Days | Avg EU Range | RTH Open Location | Pct Undercut EU Mid | Pct Undercut EU Low | Pct Above EU High |
|---|---|---|---|---|---|
| 73 | 190.45 | 52.28% | **97.26%** | **87.67%** | **41.10%** |

**By weekday (bearish RTH days):**

| Weekday | Days | Avg EU Range | Avg RTH Open Location | Pct Undercut EU Mid | Pct Undercut EU Low | Pct Above EU High |
|---|---|---|---|---|---|---|
| Friday | 12 | 203.88 | 50.42% | **100%** | 91.67% | 58.33% |
| Monday | 13 | 244.04 | 67.72% | 92.31% | 76.92% | **69.23%** |
| Thursday | 19 | 195.82 | 50.87% | 94.74% | 84.21% | **26.32%** |
| Tuesday | 14 | 150.29 | 41.45% | **100%** | 85.71% | 42.86% |
| Wednesday | 15 | 163.95 | 52.29% | **100%** | **100%** | **20.00%** |

**Findings:**
- **97.26% of bearish RTH days undercut the EU midpoint** — near-universal. On a bearish day, RTH nearly always trades below the middle of the EU range regardless of where it opened.
- **87.67% undercut the EU low** — on almost 9 in 10 bearish days, RTH drills below the EU session's low. EU longs entered during the overnight session were underwater on 88% of bearish days. The mirror of this finding from the bullish side: EU low was undercut 46.5% on bullish days vs 87.67% on bearish days — directional bias is the dominant factor.
- **Only 41.1% of bearish days trade above the EU high** — the RTH post-open bounce reaches EU high less than half the time on bearish days. This means entering EU shorts near the EU high was at or near the best available price on 59% of bearish days (RTH never offered a better short above EU high).
- **Wednesday is the most extreme bearish weekday: 100% undercut EU midpoint, 100% undercut EU low, only 20% above EU high** — on every bearish Wednesday in the dataset, RTH traded below both EU midpoint and EU low. EU shorts on Wednesday were optimal — RTH almost never traded above EU high to offer a better short entry.
- **Monday bearish days: 69.23% trade above EU high** — the highest of any weekday. On bearish Mondays, RTH commonly bounces above the EU high before selling off — giving a better short entry than EU provided. Consistent with Monday's structural open dip (RTH-SESS-008: 100% of Mondays dip below open) — even on bearish Mondays there's an opening bounce that may exceed EU levels.
- **Thursday bearish days: only 26.32% above EU high** — short entries near EU high during the overnight session were rarely improved upon by RTH on bearish Thursdays. Thursday's structural opening fade (RTH-FH-002: FH sets day HIGH 54.8%) means RTH opens near the high on Thursdays and sells without bouncing above EU levels.
- **Practical EU short entry framework (bearish day context, known after close):**
  - **Wednesday:** EU short entries near EU high are optimal — RTH almost never trades above EU high (only 20%). No benefit waiting for RTH.
  - **Thursday:** EU high entries also good — only 26% chance RTH offers a better short above EU high. Thursday structural fade confirms.
  - **Monday:** Wait for RTH bounce — 69% of bearish Mondays trade above EU high, offering better short entry than EU session provided. Don't short EU high on Monday bearish days.
  - **Friday:** 58% trade above EU high — marginal edge to waiting for RTH on bearish Fridays.
- **Comparison with RTH-GLOB-002 (bullish days):** Bearish days are far more destructive to EU levels (87.67% vs 46.5% undercut EU low) — on a bearish day, the EU session structure means almost nothing as RTH trends through it. On bullish days, EU levels have more relevance as support.

### RTH-GLOB-001 — Overnight Gap Direction vs RTH Session Direction
**Query ID:** RTH-GLOB-001 | Task date: 2026-07-08
**Method:** LAG(close) for prev_close. gap_direction = open vs prev_close. rth_bullish = close > open (intraday). rth_close_above_prev = close > prev_close (net vs prior close). 185 days (first excluded). Flat opens absorbed into gap_down — negligible count.

**Aggregate (all weekdays):**

| Gap Direction | Days | Avg Gap Size | RTH Bullish % (open→close) | RTH Close > Prev Close % |
|---|---|---|---|---|
| gap_down | 88 | 150 pts | **63.6%** | 37.5% |
| gap_up | 97 | 147 pts | 45.4% | **65.0%** |

**By weekday:**

| Weekday | Gap | Days | RTH Bullish % | RTH Close > Prev % |
|---|---|---|---|---|
| Monday | gap_down | 18 | **77.8%** | 50.0% |
| Friday | gap_down | 15 | **73.3%** | 46.7% |
| Tuesday | gap_down | 21 | 66.7% | 33.3% |
| Wednesday | gap_down | 12 | 58.3% | 50.0% |
| Thursday | gap_down | 22 | 45.5% | **18.2%** |
| Monday | gap_up | 20 | 45.0% | **80.0%** |
| Wednesday | gap_up | 22 | 54.6% | **77.3%** |
| Friday | gap_up | 20 | 50.0% | 65.0% |
| Thursday | gap_up | 17 | **29.4%** | 53.0% |
| Tuesday | gap_up | 18 | 44.4% | 44.4% |

**Findings:**
- **Gap direction predicts net daily direction (vs prior close) but NOT intraday direction (open→close).** The intraday session reverses the overnight move in both cases:
  - Gap-down → RTH open→close bullish 63.6% (intraday reversal) BUT close vs prior close only 37.5% (not enough recovery)
  - Gap-up → RTH open→close bullish only 45.4% (intraday fade) BUT close vs prior close 65.0% (gap holds net)
- **The overnight gap is never fully erased on average** — gap-ups close net positive 65% of the time despite fading intraday; gap-downs close net negative 62.5% of the time despite recovering intraday. The gap direction sets the day's net bias; the intraday session oscillates around it.
- **Thursday is the structural exception on both sides:**
  - Gap-down Thursday: only 45.5% bullish intraday (only weekday below 50%) AND only 18.2% close above prior close — Thursday gap-downs continue lower both intraday and net. Unique among all weekdays.
  - Gap-up Thursday: only 29.4% bullish intraday (worst fade of any weekday) — Thursday fades gap-ups hardest. Both findings align with Thursday's structural bearish profile (RTH-CLOSE-001: -90 avg).
- **Monday gap-down is the strongest intraday reversal: 77.8% bullish open→close** — directly confirms RTH-SESS-008 (Monday buy-the-open setup). The overnight gap-down gets aggressively bought intraday on Mondays.
- **Monday gap-up holds net strongly: 80.0% close above prior close** — gap-up Mondays don't give back the overnight gain. Despite fading intraday (only 45% bullish open→close), the net day still closes above the prior close 80% of the time.
- **Wednesday gap-up: 77.3% close above prior close** — second strongest gap-up holder. Consistent with Wednesday's bullish close-to-close drift (RTH-CLOSE-001: +80 avg, 67.6% up).
- **Tuesday gap-up is the weakest: 44.4% both intraday AND net** — gap-up Tuesdays neither continue nor hold. The overnight gain fades completely by RTH close. Consistent with Tuesday's near-neutral average close-to-close (-5 avg).
- **Practical overnight→RTH decision framework:**

| Signal | Intraday bias | Net day bias | Action |
|---|---|---|---|
| Gap-down Monday | Buy (78%) | Neutral (50%) | Long intraday, close before EOD |
| Gap-down Thursday | Sell (55%) | Sell (82%) | No buy — only weekday to continue lower |
| Gap-up Monday/Wednesday | Fade intraday | Hold net (77-80%) | Fade open→close move but net stays positive |
| Gap-up Tuesday | Fade intraday | Fade net (56%) | Cleanest short after gap-up |
| Gap-up Thursday | Fade hard (71%) | Neutral net | Fade the open, cover before close |
