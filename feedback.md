## Week 28, Day 5 — Agent Feedback on Student

**Score: 19/20**

**Task 1 (RTH-VOL-011):** Five-CTE architecture handling a genuinely hard problem. Correctly separated high/low (MAX/MIN, no deduplication needed) from open/close (DISTINCT ON + MIN/MAX ts_event JOIN to ticks). Identified and solved the nanosecond-timestamp duplicate problem independently — professional-level instinct. Running MAX/MIN window function with default frame (RANGE UNBOUNDED PRECEDING) is correct here since ORDER BY is on a time column with no ties at the per-day level. 16:00 row at 102% is a boundary artefact, not a query bug. Key finding: median day fully ranged by 13:30; 51% of range captured in the opening 30 minutes. 10/10.

**Task 2 (RTH-VOL-012):** Clean extension of RTH-VOL-010. LAG for prev_close/prev_open correct, CASE WHEN for prev_direction clean. Minor: flat days (prev_close = prev_open) fall into 'bearish' via ELSE — negligible count but worth noting. Key finding: after large bullish day 65.7% continuation; after large bearish day 50% (coin flip); after small bearish day only 37% bullish — strongest directional signal is actually on small bearish days, not large ones. Prior direction adds real information on bullish side and small-bearish case, but not after large down days. 9/10.

---

## Week 28, Day 4 — Agent Feedback on Student

**Score: 19/20**

**Task 1 (RTH-SESS-006):** Clean weekday extension of RTH-SESS-005. Correct propagation of weekday through both HOD/LOD CTE chains. SUM(COUNT(*)) OVER (PARTITION BY weekday) for within-weekday denominators is exactly right. DISTINCT ON on the final JOIN is the same workaround as RTH-SESS-005 — same minor limitation (INNER JOIN drops non-overlapping windows), negligible impact. Key findings: Thursday HOD 45.2% in 09:30 (highest weekday); Monday LOD 51.5% in 09:30 (half of Monday lows are set at the open); Tuesday HOD splits 24%/27% between 09:30 and 15:30 (bi-modal, consistent with afternoon reversal pattern). 9/10.

**Task 2 (RTH-VOL-010):** Clean two-CTE structure. LAG correct, NTILE(3) on both ranges in same CTE is efficient. Minor: hardcoded `>= 380` for `pct_today_wide` instead of computing Q3 via subquery as specced — pragmatic and result is clear. Finding is one of the most actionable of the project: 3.5x difference in wide-day probability (55.7% after large vs 16.1% after small). Volatility clusters strongly — NQ daily ranges are serially correlated. Monotonic relationship across all three buckets confirms it's a real gradient, not noise. 10/10.

---

## Week 28, Day 3 — Agent Feedback on Student

**Score: 19/20**

**Task 1 (RTH-SESS-004):** Clean two-CTE structure. NTILE(4) computed in first CTE, classification in second, final GROUP BY clean. NULLIF on close_location denominator correct. FILTER (WHERE r_close > r_open) syntax correct. Results crisp and actionable. Key finding: both_tight rest range 197 pts vs both_wide 339 pts — 42% smaller range on joint tight days. Direction random on both_tight (54.8% ≈ mixed 55%); both_wide most directional and most bullish (67.9%). 10/10.

**Task 2 (RTH-SESS-005):** Four-CTE structure with the smart optimization: joined ticks to daily_ohlcv_rth using session_start/session_end (UTC timestamps from the view) rather than AT TIME ZONE cast on 56M rows — correct professional instinct. Window bucketing formula correct (floor(minute/30) * 30min). Subquery `(SELECT COUNT(*) FROM hod_lod_agg)` reused cleanly for both HOD and LOD pct denominators. Minor: final JOIN uses INNER JOIN — should be FULL OUTER JOIN to catch windows where high and low windows differ. In practice no rows were lost from the data. 9/10.

---

## Week 28, Day 2 — Agent Feedback on Student

**Score: 17/20**

**Task 1 (RTH-FH-005):** CTE chain clean, LAG correct, gap_direction logic correct. Good call dropping the or_open metric (correctly identified it's the same as fh_open). Filter issue: used `fh_high > r_high` (FH high > rest-of-session high) instead of `fh_high >= d.high` (FH high = full day high) — different population, subtly broader condition. Directionally valid results. Key finding: 100% of 10 gap-up Tuesday FH-sets-high days retrace all the way to the 09:30 open; avg drop to close 235 pts from FH peak. 8/10.

**Task 2 (RTH-FH-006):** Clean single-table aggregation. NTILE(4) correct, formula correct. Minor: column aliased `avg_fg_range` (typo, fg vs fh) — harmless. Key finding: same expansion pattern as RTH-ORB-009 — wide FH → wider rest-of-session; Q4 FH claims 50.2% of day's range vs OR Q4 at 41%. Q4 bullish close bias 64.1% matches OR Q4 (66.7%). 9/10.

---

## Week 28, Day 1 — Agent Feedback on Student

**Score: 27/30**

**Task 1 (RTH-GAP-003):** LAG correct, NTILE(3) across all gaps correct, fill logic correct. Minor: ELSE branch in gap_direction catches flat days but WHERE gap_size > 0 makes it harmless. Key finding: large gap-ups (292 pts avg) fill only 29% — today's ~300 pt gap-up is in this bucket. Large gap-ups produce bearish closes 62% of the time. Large gap-downs fill 56% AND produce 74% bullish closes — opposite asymmetry. 9/10.

**Task 2 (RTH-ORB-009):** Clean aggregation, NTILE(4) correct, or_pct_of_day formula correct. Column headers missing in pasted results but data readable. Key finding: wide OR → wider rest-of-session (not compression); Q4 rest range 383 pts vs Q1 219 pts. Q4 also has 66.7% bullish close bias. 9/10.

**Task 3 (RTH-GAP-004):** LAG correct, FILTER for directional averages clean. Minor: signed avg_gap not computed (only abs avg_gap_size) — spec called for it, but directional split + per-direction size gives equivalent info. Key finding: Wednesday gaps up 64.7% (best overnight hold for longs); Thursday/Tuesday gap down more often than up; Monday has largest gap size variance (223/170 pts). 9/10.

---

## Week 27, Day 5 — Agent Feedback on Student

**Score: 19/20**

**Task 1 (RTH-ORB-008):** Fixed `r.` prefix correctly after check — clean. CASE flags, FILTER aggregation all right. Key finding: Friday agree-bearish reaches OR close 90% but only 40% rest bullish — the bounce happens but reverses before close. Tuesday agree-bearish confirms across all three targets (92%/75%/58%). 10/10.

**Task 2 (RTH-FH-004):** LAG for gap direction correct, IS NOT NULL filter clean, daily_ohlcv_rth.weekday reused correctly. Unused `fh_direction` column computed but not in final SELECT — harmless. Key finding: Thursday FH-sets-high is gap-agnostic (57-59% both gap directions); Monday/Friday gap-down → FH sets day LOW 66.7% each. 9/10.

---

## Week 27, Day 4 — Agent Feedback on Student

**Score: 28/30**

**Task 1 (RTH-ORB-006):** Clean weekday extension of RTH-ORB-005 — correct grouping, FILTER syntax right, results rich. Minor: `signals_agree` column computed in `pre_agg` CTE but never used in final SELECT (same pattern as yesterday). Key finding: Tuesday agree-bearish 75% is the strongest single weekday signal; Friday agree-bearish 40% is the lone exception where agree-bearish is bearish. Thursday agree-bullish 50% confirms RTH-SESS-003. 9/10.

**Task 2 (RTH-ORB-007):** Logic mostly correct. One bug caught and fixed same session: `reached_or_close` used `o.r_high` (OR window high, always ≥ or_close on bearish OR days) instead of `r.r_high` — produced spurious 100%. Corrected to `r.r_high >= o.or_close` → 83.9%. All other columns were correct throughout. Good instinct to fix immediately. 9/10.

**Bonus (fomc_agg fix):** Dropped and recreated as MATERIALIZED VIEW — pragmatic call given the error. Corrected spike_magnitude CASE is exactly right. 10/10.

---

## Week 27, Day 3 — Agent Feedback on Student

**Score: 27/30**

**Task 1 (RTH-ORB-003):** Smart call building `or_rest_ohlc_ranges` as a materialized view first — correct professional instinct on a 56M-row table, now reusable for Tasks 2 and 3. DISTINCT ON via MIN/MAX ts_event join is clean and correct. Main query: CASE, FILTER, flat exclusion all right. Minor: OR upper bound uses `< '10:00'` (exclusive) — the 10:00:00 tick lands in the rest window. Negligible practical impact but worth knowing. 9/10.

**Task 2 (RTH-ORB-004):** Good judgment redefining this as delta direction vs rest-of-session first (analogous to RTH-VOL-004) before going to magnitude buckets — right sequencing. Query structure clean: FILTER aggregation, JOIN to materialized view, CASE logic all correct. Dead `rest_range` column computed but never used in final SELECT — harmless. Key finding: positive OR delta → 59.7% vs negative → 51.2% (8.5pp gap, bearish arm near-random). 9/10.

**Task 3 (RTH-ORB-005):** Clean JOIN of both materialized views, CASE logic correct across all CTEs. Minor: `pre_agg` CTE exists solely to compute `signals_agree`, which is then never used in the final SELECT — could skip that CTE and just GROUP BY directly on `directions_agg`. The results are the most counterintuitive finding of the session: bearish OR + bearish FH → rest bullish 59.7% (higher than agree-bullish at 58.4%). Divergence case (OR bearish, FH bullish → 40% rest bullish) is the only mild bearish signal. Student didn't flag this surprise explicitly in findings — worth noting in nq_findings for the trading implication. 9/10.

---

## Week 27, Day 2 — Agent Feedback on Student

**Score: 16/20**

**Task 1 (RTH-NEWS-005):** Query logic correct — date filter on JOIN fixed, MAX for boolean reversal correct, spike extreme anchoring right. Two issues: (1) spike_magnitude in the view still uses reaction_high - reaction_low (noted by student — should be spike extreme vs pre_event_price, fix deferred); (2) 100% reversal rate (all 7 FOMC days reversed) — not a bug, but worth flagging as a finding. Good initiative creating the view first. 8/10.

**Task 2 (RTH-ORB-002):** Breakout logic clean, within-group pct correct. Recurring ts_recv issue for day_of_week — Sunday rows confirm it again. Fix: use TRIM(TO_CHAR(trade_date, 'Day')). Findings strong despite bug — Tuesday bullish OR delta → break_up 62.5% is the strongest weekday alignment. 8/10.

---

## Week 27, Day 1 — Agent Feedback on Student

**Score: 29/30**

**Extra (RTH-VOL-009):** Done Sunday trading brief — delta vs candle direction alignment. Clean, correct. 10/10.

**Task 1 (RTH-NEWS-003b):** FOMC query skeleton reused cleanly, time windows correct, fh_open as RTH open proxy is a smart shortcut. Minor: CTE still named ticks_fomc_days — should be ticks_news_days. Not penalized. 9/10.

**Task 2 (RTH-NEWS-004):** Clean aggregation wrapping Task 1. All columns correct, FILTER syntax right. Key finding: CPI 71% closed above pre-event (initial drop gets bought), NFP 40% closed above pre-event (initial pop fades). Strong contrast for only 5-7 days each. 10/10.

---

## Week 26, Day 5 — Agent Feedback on Student

**Score: 18/20**

**Task 1 (RTH-NEWS-003a):** Query skeleton correct — pre-event price via aggregate-then-JOIN, reaction range, spike direction via timestamp comparison, all clean. Stopped before adding reversal logic after correctly identifying the flaw: reversal check is meaningless without first anchoring to the spike extreme (price could have already returned to pre-event level mid-spike). Stopping at the right moment is professional judgment. Future improvement: event_time-driven window instead of hardcoded 14:00. 8/10.

**Task 2 (RTH-VOL-008):** Clean extension, good initiative running full unfiltered version too. Key finding: 09:45 positive prior delta is the ONLY window with both strong direction (66.2%) AND strong magnitude (+20 pts avg_move_all). Other "edges" from RTH-VOL-007 don't survive the magnitude check — 14:45 and 15:00 show only +5 pts avg move, which is marginal after spread. 10/10.

---

## Week 26, Day 4 — Agent Feedback on Student

**Score: 27/30**

**Task 1 (RTH-NEWS-001):** Table design correct — instrument column, event_time_et, notes all present. Delayed NFP (Nov 2025 govt lapse, Feb 2026 BLS) is professional-level detail. Oct 2025 NFP missing (delayed from Oct 3 to Nov 20 — likely intentional given the Nov 20 entry covers it). 9/10.

**Task 2 (RTH-NEWS-002):** Two issues: (1) base table is rth_firsthour_rest_ohlc_ranges using r_close — should be daily_ohlcv_rth.close for true RTH close-to-close comparison; (2) weekday computed via TO_CHAR when daily_ohlcv_rth already has the column. Findings directionally valid. Key insight: clean Wednesdays +102 avg, 70.8% up — stronger than Monday, the bullish Wednesday pattern is real. 8/10.

**Task 3 (RTH-VOL-007):** Minimal change, maximal insight. Standout windows: 09:45 positive prev delta 66.2%, 14:45 positive 65.3%, 15:00 negative 60%, 13:30 negative 60.2%. Opening momentum (09:45) is the cleanest single edge found in the 15-min analysis. 10/10.

---

## Week 23, Day 5 — Agent Feedback on Student

**Score: 18/20**

**Task 1 (dominant_type RANK fix):** Clean. RANK() correct, ties preserved, pattern solid. 10/10.

**Task 2 (NULLIF denominator):** SUM(NULLIF(amount, 0)) still wrapping data values — zeros incorrectly excluded. Plain SUM(amount) is correct; SUM ignores NULLs naturally. NULLIF only belongs on the denominator. 8/10.

---

## Weekly Summary — Week 23 (2026-05-18 to 2026-05-22)

### Scores
| Day | Focus | Score |
|-----|-------|-------|
| Day 1 | GROUP BY + HAVING job_db + MoM LAG | 20/20 |
| Day 2 | Type A CTE + cohort LEFT JOIN + FIRST_VALUE | 26/30 |
| Day 3 | Cohort retention retry + cumulative SUM + cross-schema | 27/30 |
| Day 4 | dominant_type CTE + PERCENT_RANK + NULLIF | 26/30 |
| Day 5 | dominant_type RANK fix + NULLIF denominator | 18/20 |

**Week total: 117/130. Solid week with real learning on new patterns.**

---

### Key Wins

- **Cohort LEFT JOIN cracked** — boundary logic (>= month+1, < month+4), LEFT JOIN preserved, retention formula all correct on Day 3 retry. Pattern is now locked.
- **dominant_type pattern introduced and absorbed** — two-CTE aggregate-then-rank approach understood. RANK() vs ROW_NUMBER distinction clear by Day 5.
- **PERCENT_RANK on job_db** — clean execution, correct partition and order, no issues.
- **Cumulative SUM with explicit frame** — ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW correct first attempt.
- **Cross-schema JOIN** — city = miasto join between crappy_data_db and job_db working cleanly.
- **FIRST_VALUE** — correct usage with DESC sort for latest value, DISTINCT deduplication clean.

---

### Focus Areas

- **NULLIF on data vs denominator** — recurring mistake: wrapping SUM/COUNT values in NULLIF(col, 0) instead of using it only on denominators. Rule: `SUM(amount)` and `COUNT(amount)` ignore NULLs naturally. NULLIF belongs only in `/ NULLIF(count, 0)` to prevent division by zero. Missed on both Day 4 and Day 5.
- **ROW_NUMBER vs RANK for ties** — fixed on Day 5 after Day 4 slip. When the spec says "return all ties", always RANK().
- **COUNT(*) vs COUNT(DISTINCT col) on tables without PK** — on job_db.oferty (no PK), COUNT(DISTINCT pozycja) counts distinct titles not offers. Use COUNT(*) as offer count proxy.
- **Redundant JOINs** — Day 2 Task 1 included an unnecessary JOIN to orders for line revenue calculation.

---

### Week 24 Plan

- **NULLIF** — one more clean drill with a real dirty-data scenario, denominator-only usage
- **gaps-and-islands time proximity** — group events within 30-minute windows (session detection pattern), introduced as new concept #3
- **YoY comparison** — LAG(revenue, 12) OVER (ORDER BY month), hasn't appeared since Week 21
- **Type B recursive CTE** — still blocked on schema (no self-referencing table); defer until PK/parent_id added or use a different table
- **dominant_type** — one more rep in a different context (e.g. per platform or per city instead of per user)

---

## Student Feedback on Questions

Week complete.

---

## Week 26, Day 3 — Agent Feedback on Student

**Score: 18/20**

**Task 1 (RTH-SESS-003):** Clean CTE chain, LAG correct, good call not filtering to Mon/Thu only. Two label inversions: (1) fh_direction CASE has `fh_open > fh_close` = 'bullish' — backwards, should be `fh_close > fh_open`; (2) avg_close_location uses `daily_high - r_close` in numerator instead of `r_close - daily_low` — gives inverse of close location. Results directionally interesting but labels flipped. 7/10.

**Task 2 (RTH-ORB-001b):** `SUM(COUNT(*)) OVER (PARTITION BY or_delta_direction)` as denominator is exactly right. Clean fix. Dead ts_recv columns still present but harmless. 9/10.

**Task 3 (RTH-VOL-006):** Materialized view build was complex and correct. Main query clean — LAG partitioned by trade_day, FILTER for conditional count. Finding (no aggregate edge) is the honest answer. Instinct to drill by hour_min is correct next step. 10/10.

---

## Week 26, Day 2 — Agent Feedback on Student

**Score: 17/20**

**Task 1 (RTH-VOL-005):** RTH filter correct this time. FILTER syntax clean. Two issues: (1) hour/weekday grouping computed from ts_recv instead of ts_event — always use ts_event for session analysis; (2) global pct would be more useful as within-group pct. 8/10.

**Task 2 (RTH-ORB-001):** Excellent structure — four clean CTEs, break_both handled first in CASE (correct priority). Dead ts_recv columns carried from Task 1 but unused. Pct is global (% of all 159 days) rather than within or_delta_direction — within-group rates: bullish→break_up 52%, bearish→break_down 51%. Strongest directional signal found so far. 9/10.

---

## Week 26, Day 1 — Agent Feedback on Student

**Score: 15/20**

**Task 1 (RTH-VOL-003):** FILTER syntax clean, correct aggregation. Two issues: (1) no RTH filter — included all 24 hours instead of 09:30–16:00 ET; (2) `ts_recv AT TIME ZONE` computed in CTE but never used — dead column. RTH rows present and correct, finding still valid. 7/10.

**Task 2 (RTH-VOL-004):** Logical structure sound — CTE chain clean, FILTER aggregation correct, flat exclusion correct. Measuring rest-of-session (not full-day) was the correct actionable choice — task spec was flawed, student's approach was right. Label bug (`r_open - r_close > 0` = bearish, not bullish) fixed by student. Fixed results: positive FH delta → bullish rest 60.3% vs 54.7%. Minor: dead `ts_recv` column in first CTE. 9/10.

---

## Week 25, Day 3 — Agent Feedback on Student

**Score: 18/20**

**Task 1 (RTH-FH-003):** FILTER syntax for conditional counts clean and concise. Flat FH exclusion correct. Added bearish columns beyond spec — not penalized. Minor: avg_fh_range covers all days for the weekday, not split by direction — correct per spec but worth knowing. 9/10.

**Task 2 (RTH-SESS-001):** Three clean CTEs, each doing one job. Using fh_open as gap reference is valid. no_gap label + WHERE filter cleaner than ELSE branch. NULLIF missing on close_location denominator again — same recurring miss as Day 2. Findings are the richest of the project so far. 9/10.

---

## Week 25, Day 2 — Agent Feedback on Student

**Score: 18/20**

**Task 1 (RTH-CLOSE-002):** Extra metric and 2-decimal rounding — not penalized, better output. One latent bug: missing NULLIF on `(high - low)` denominator — didn't crash because no zero-range days exist in the data, but should be `/ NULLIF(high - low, 0)`. 9/10.

**Task 2 (RTH-FH-002):** Clean CTE reuse for both summary and weekday breakdown. DISTINCT ON on the 1-to-1 join is harmless but unnecessary — both views have one row per trade_date. Findings are strong and cross-reference well with prior results. 9/10.

---

## Week 25, Day 1 — Agent Feedback on Student

**Score: 16/20**

**Task 1 (RTH-FH-001):** Creating the materialized view first was smart — right professional call on a 56M-row table. DISTINCT ON inside the view handles timestamp fan-out. Main query clean: CASE + SUM/COUNT pattern solid. Minor: flat first-hour days (fh_close = fh_open) silently fall into 'bearish' via the ELSE branch — harmless in practice but imprecise. 9/10.

**Task 2 (RTH-GAP-002):** Fill condition logic correct (low <= prev_close for gap-up, high >= prev_close for gap-down). Bug: CASE uses ELSE 'gap_up' so flat opens (open = prev_close) get absorbed into gap_up instead of excluded. Task spec said to filter flat opens. Flat open days trivially satisfy the fill condition, slightly inflating gap_up fill rate. Fix: add `AND open != prev_close` to the WHERE in rth_gaps. 7/10.

**Task 3 (RTH-CLOSE-002):** Cancelled by student — session was long enough.

---

## Week 24, Day 4 — Agent Feedback on Student

**Score: 18/20**

**Task 1 (RTH-RANGE-002 DISTINCT ON fix + weekday summary):** DISTINCT ON placement is correct — on the outer SELECT wrapping the pre-aggregated CTEs, not inside one of them. Structure is clean: two aggregate CTEs (first_hour_agg, rest_agg), DISTINCT ON in the middle CTE (trade_dates_agg), then a plain GROUP BY weekday on top. NULLIF on the denominator of fh_pct_of_day is correct usage. Minor: ORDER BY at weekday summary level wasn't specified but omitting it is fine for exploration. 9/10.

**Task 2 (RTH-CLOSE-001):** LAG(close) OVER (ORDER BY trade_date) is clean and correct. FILTER (WHERE ...) syntax for conditional counts is good. NULLIF on prev_close denominator is correct. Minor: close_gap >= 0 for up_days includes flat days (change = 0); using > 0 would be a more precise "up" definition — same for down_days. Not penalized but worth noting for precision. 9/10.
