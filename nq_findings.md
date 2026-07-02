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

*(findings to be added — after RTH analysis is mature)*
