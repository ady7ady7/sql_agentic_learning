# NQ Futures — Analytical Findings

**Dataset:** NQ.c.0 continuous front month, Sep 2025 – May 2026 (~7.5 months)
**Session default:** RTH (09:30–16:00 ET) unless noted as Globex
**Query IDs:** reference SQL queries by ID (e.g. RTH-001) for reproducibility

---

## Volume & Aggressor Pressure

### RTH-VOL-001 — Buy/Sell Volume Ratio by Weekday (RTH)
**Query:** Task 2026-06-03, daily_ohlcv_rth aggregated by weekday
**Finding:** Buy/sell volume ratio is nearly perfectly balanced across all weekdays (range: 0.995–1.001). No weekday shows persistent directional aggressor bias at the aggregate level.
**Implication:** Daily-level buy/sell symmetry — aggressor imbalance, if it exists, operates at intraday or tick-by-tick level, not at the weekday aggregate.

---

## Daily Ranges

### RTH-RANGE-001 — Average Daily Range by Weekday (RTH)
**Query:** Task 2026-06-03, daily_ohlcv_rth aggregated by weekday
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
