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

*(findings to be added)*

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

## Session Patterns

*(findings to be added)*

---

## News Days

*(findings to be added)*

---

## First Hour (09:30–10:30 ET)

*(findings to be added)*

---

## Globex vs RTH Comparisons

*(findings to be added — after RTH analysis is mature)*
