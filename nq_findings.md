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
**Query ID:** RTH-RANGE-002 | Task date: 2026-06-04
**Finding:** First hour (09:30–10:30 ET) accounts for a significant but variable portion of the full RTH day range. Query extended to include full OHLC for both first hour and rest of session (10:30–16:00), enabling future analysis of whether first hour high/low/close predicts rest-of-session behavior.
**Note:** Duplicate rows present in output due to timestamp ties in point-lookup JOINs — needs DISTINCT ON fix before using for aggregate analysis.

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
