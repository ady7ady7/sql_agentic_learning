## Week 31, Day 3 — Agent Feedback on Student

**Score: 17/20**

**Task A (RTH-INTRA-003):** Dobry wybór źródła — `or_rest_ohlc_ranges` ma `or_open`/`or_close` natywnie, nie trzeba było pullować 10:00 bucket ręcznie. LAG na `r_close` przed JOIN poprawny. Kluczowa poprawa: eliminacja `close_same_dir_pct`, jedna czytelna kolumna `close_above_snapshot_pct`. Dwa minusy: `TO_CHAR(date, 'Day')` zamiast natywnego `weekday` z `daily_ohlcv_rth` (trailing spaces wykryte i naprawione TRIM-em w drugiej iteracji — dobry instynkt); `HAVING COUNT(*) > 5` zamiast `>= 5` w pierwszej wersji. Findings mocne: Monday bullish OR above open = 75% (N=20) najsilniejszy sustained signal; Tuesday bearish OR above open po 13:00 = 14–25% (nowy sygnał niewyławiany przez gap direction); Friday bearish OR below open po 13:00 = 12–37% (czystsze niż INTRA-002b). 8/10.

**Task B (RTH-FH-008):** Dobra struktura CTE. `fh_high_location` i `fh_low_location` poprawne nawiasy od razu. `close_location` — brakujące nawiasy w pierwszej wersji (`CLOSE - low / NULLIF(...)`) dało wartości ~25000 zamiast 0–1, wykryte i naprawione samodzielnie. `NTILE(3)` globalnie poprawne. Findings bardzo mocne: bucket 3 bearish = 0% close_above_open na każdym weekdayu (N=5–14) — najczystszy binary signal w całej serii FH. Bucket 1 bullish = 88–100% close_above_open, avg_close_location 0.72–0.85 — slow-burn bullish dzień. Dobra inicjatywa z follow-up pomysłem (fh_high_location = 1.0 / fh_low_location = 0.0). 9/10.

---

## Week 31, Day 2 — Agent Feedback on Student

**Score: 17/20**

**Task A (RTH-INTRA-002):** Structurally clean — CTE pattern correct, LAG computed inside CTE before JOIN (no window contamination), HH24:MI correct (no repeat of Day 1 bug), HAVING COUNT(*) ≥ 5 added without prompting. The `above_open = close_above_snapshot` equality check for `close_same_dir_pct` is logically sound for PostgreSQL booleans. Results show striking asymmetry: Tuesday gap_up above open = 0% same_dir across every cell with sufficient N (10:00 through 11:00 and beyond) — essentially no false positives. Complementary cells (Tuesday gap_down above open = 78–93%) confirm the gap direction is the decisive variable. Monday gap_down above open at 10:15–10:30 = 85–90% (N=21), the dataset's highest-confidence large-N finding. 9/10 — deduction for not adding a `day_count` alias on COUNT(*) as specified, and for not commenting on the 15:45 artefact row behavior.

**Task B (RTH-INTRA-002b):** Clean filter — `WHERE bucket_window IN ('10:30', '11:00', '12:00', '13:00', '14:00')` applied correctly, same CTE logic. No structural issues. This version confirms all RTH-INTRA-002 findings with higher N per cell and cleaner signal. Thursday gap_down below open = 0% same_dir (100% close above snapshot) at every window from 10:30–14:00 — consistent and notable. Tuesday gap_up above open N is small (5–8) but the 0% rate is absolute and consistent across multiple windows. Monday gap_down collapses from 90% at 10:30 to 33% at 14:00 — correctly captured. 8/10 — same COUNT(*) alias issue, and the brief for findings could have been more explicit about the edge decay timing.

---

## Week 31, Day 1 — Agent Feedback on Student

**Score: 17/20**

**Task A (RTH-INTRA-001):** Architecture clean — CTE pattern correct, JOIN on `bucket_start::date` correct, FILTER syntax right. Key bug: `HH:MI` instead of `HH24:MI` in TO_CHAR truncated results to 10:00–12:59 only (13:00 formatted as "01:00", filtered out by `>= '10:00'`). Fixed immediately on second run. Finding: aggregate direction does NOT lock in through the day — signal peaks at 10:00–10:30 (61%) and decays to near-random by 12:00. The aggregate hides everything useful. 8/10.

**Task B (RTH-INTRA-001b):** Same fix applied correctly, weekday extension clean. Strong findings: Monday 10:45 below-open = 87% close-above-snapshot (strongest mean-reversion cell in dataset); Friday above-open by 10:30 = fade signal (39–41% same_dir by 10:45–12:00); Wednesday above-open 11:00–14:15 = 62–76% close confirmation; Thursday below-open doesn't confirm (54–68% close above snapshot); Tuesday morning below-open = fade, afternoon below-open = follow. Self-initiated extension with genuine insight. 9/10.

---

## Week 30, Day 5 — Agent Feedback on Student

**Score: 17/20 + bonus Friday extension**

**Task 1 (RTH-CLOSE-003):** Close location × gap × day direction × weekday. 9/10 — query architecture clean, ROUND(ABS(...)) pattern correct, FILTER syntax for pct_closed_upper_half correct. Close location formula uses ABS((close - low) / (high - low)) — correct. Key finding: day direction is the near-sole determinant of close location (bullish → 75–84%, bearish → 19–35% universally). Wednesday gap_down bearish exception (43.89%) is the only outlier. Friday gap_down bullish = widest range day in the dataset (428 pts avg). Minus: one more session with the duplicate alias going unmentioned, but query output was correct.

**Task 2 (RTH-FH-007 + self-initiated RTH-FH-007b):** Good self-correction — identified the Friday-only scope mismatch and added both Friday-specific and all-days aggregation without being prompted. The all-days finding (gap_size bucket 3 gap_down → 85.19% rest bullish, avg FH 265 pts) is genuinely new and significant. Friday large gap-down: avg day 492 pts, rest bullish 67%. Student correctly treated the general aggregation as a separate query ID (RTH-FH-007b). Bonus for initiative. 8/10 for architecture (NTILE scope was correct after fix). Note: full Friday sample is small (6 days per large-bucket cell).

---

## Week 30, Day 4 — Agent Feedback on Student

**Score: 17/20 + bonus Task 3**

**Task 1 (RTH-GLOB-007b):** Weekday × gap breakdown na bearish dniach. 8/10 — query clean, poprawny GROUP BY. Duplikat aliasu avg_eu_range po raz piąty. Key finding: Wednesday gap_down bearish 0% exceeded (najczystszy EU short w datasecie), Thursday gap_down 9.09% — oba prawie gwarantowane. Tuesday gap_down bearish = pułapka (57% exceeded), Tuesday gap_up bearish = zaskakująco czysty (28.57%).

**Task 2 (RTH-VOL-019):** OR delta × FH delta combined signal. 9/10 — clean architecture, dwie oddzielne CTEs, JOIN poprawny. Minor: FH window < '10:45' zamiast < '10:30' (wlicza bucket 10:30 do FH). Finding: agree_bearish 57.97% rest bullish (mean reversion potwierdzona), agree_bullish 60.66% (nie stackuje ponad standalone FH delta 60.3%). Delta nie jest lepszym stacking signal niż price direction.

**Task 3 (RTH-VOL-019b, self-initiated):** Weekday breakdown agree signal — bardzo dobry ruch. Tuesday agree_bearish 75% rest bullish (potwierdza RTH-ORB-006 price direction), Monday agree_bullish 68.75% (najsilniejszy bullish kombination), Friday agree_bearish 42.86% (jedyny weekday gdzie agree_bearish jest bearish). Bonus za inicjatywę.

---

## Week 30, Day 3 — Agent Feedback on Student

**Score: 17/20**

**Task 1 (RTH-GLOB-006b):** Weekday breakdown poprawny — dorzucenie weekday do GROUP BY wystarczyło. Zostawiłeś prev_day_direction w grupowaniu (nie było w specyfikacji) ale daje użyteczny kontekst przy małych sample sizes. Duplikat aliasu avg_eu_range po raz czwarty (reached_eu_midpoint pod złą nazwą). Key finding: Wed/Thu bearish EU high trzyma 73-80%, Tuesday ~62% viable, Monday 69-83% exceeded — czekaj na RTH. 8/10.

**Task 2 (RTH-GLOB-007):** Clean extension — gap_direction CASE poprawny, LAG już był w CTE. Finding mocny: gap direction wyjaśnia 26.5pp spreadu (74% vs 48% hold rate) — identyczna siła jak na bullish side (26.9pp z RTH-GLOB-004). Gap-down bearish: EU high optymalny short 74% czasu. Gap-up bearish: 52% exceeded, czekaj na RTH. Weekday × gap breakdown odłożony jako RTH-GLOB-007b. Duplikat aliasu jedyny minus. 9/10.

---

## Week 30, Day 2 — Agent Feedback on Student

**Score: 17/20**

**Task 1 (RTH-VOL-014b):** Signed delta surge ratio — clean refactor, signed ratio correct, WHERE local_delta IS NOT NULL dobrze usuwa pierwsze buckety. Dead column `is_surge` z VOL-014 zostawiony w surge_agg (nieużywany, nie wpływa na wynik). Key finding: symetria — up surge 44.92% vs down surge 43.62% next bucket up. Asymetrii nie ma, ABS w VOL-014 był wystarczający. 9/10.

**Task 2 (RTH-GLOB-006):** EU high jako resistance na bearish dniach — logika EU session poprawna, wyniki sensowne (EU high trzyma 60% czasu, RTH otwiera ~90 pts poniżej EU high). Dwa problemy: duplikat aliasu avg_eu_range (mislabeled kolumna reached_eu_midpoint), brak weekday breakdown który był kluczową częścią pytania (odłożone na jutro jako RTH-GLOB-006b). 8/10.

---

## Week 30, Day 1 — Agent Feedback on Student

**Score: 18/20**

**Task 1 (RTH-VOL-014):** Poprawna naprawa window frame (4 PRECEDING AND 1 PRECEDING) — kluczowa zmiana eliminująca circular reference. Query clean, LEAD dla next_bucket_direction poprawny, FILTER syntax correct. Wynik nieoczekiwany ale solidny: surge = exhaustion, nie momentum. Up surge → 43.94% next up vs non-surge 50.06%. ABS delta traci sign — słuszna obserwacja studenta, warto przetestować jako RTH-VOL-014b. Minus: wyniki nie zostały zaktualizowane w tasks.md po pierwszej wersji — trzeba było dopytać. 8/10.

**Task 2 (RTH-VOL-015):** Dwie poprawki zidentyfikowane i naprawione (full_day_range, close_location). Final query clean, NTILE(3) poprawny, trzy JOINy na trade_date bez problemów. Wynik ciekawy: medium delta opening (bucket 2) → 74.19% bullish rest, najsilniejszy signal — wyższy niż hot open (51.61%). Range monotonicznie rośnie z delta pressure. Close location spójne z bullish_rest. Minor: local_delta CTE nadal używa ROWS BETWEEN 3 PRECEDING AND CURRENT ROW (nie naprawione z Task 1) — minimalne znaczenie praktyczne tutaj. 10/10.

---

## Week 29, Day 5 — Agent Feedback on Student

**Score: 20/20**

**Task 1 (RTH-GLOB-005):** Smart call running all days first then bullish-only — gives context for the bullish split and lets you sanity-check the numbers. Query structure clean. Same day_direction CASE inversion as yesterday (rth_open > rth_close = 'bullish') but WHERE filter uses rth_close > rth_open directly — results correct, internal label wrong. Key findings: gap-down Wednesday bullish is the cleanest EU entry setup (16.67% EU low undercut); gap-up days universally safe for EU lows (25–33%); gap-down Friday/Thursday/Tuesday all see 71–82% EU low undercut. Gap direction explains more variance than weekday alone. 10/10.

**Task 2 (RTH-SESS-009):** Genuinely hard 6-CTE query built correctly under time pressure. Architecture differs from spec but is logically sound — computing post_1030_low directly (MIN price) and filtering to breakdowns afterward is equivalent to finding the first breakdown tick. One subtle issue: `post_1030_low_timestamp::time` discards the date — saved by the `t.ts_event::date = p.trade_date` condition. Recovery target uses fh_high (FH high) instead of full-day high as specced — actually a better trading target since it's known at 10:30. Results: 87/159 days (55%) see post-10:30 LOD breach; avg 138 pts depth; 74.71% recover to RTH open; 55.17% above FH high. Clear fade setup. 10/10.

---

## Week 29, Day 4 — Agent Feedback on Student

**Score: 19/20**

**Task 1 (RTH-GLOB-003):** Correct identification of the original spec flaw — computing prev_day_direction inside a bullish-day-filtered CTE would have contaminated the LAG window, giving a skewed sample. Fix is the right approach: compute both day_direction and prev_day_direction in a clean CTE over all days, filter to bullish only in the final SELECT. Query structure sound. Minor: duplicate alias `avg_eu_range` used twice in final SELECT (second instance is pct_undercut_eu_midpoint) — won't crash but one value is silently labelled wrong. Results honest: prior day direction barely splits the aggregate (43% vs 50% EU low undercut). Finding is: prior day direction is NOT a useful pre-EU signal. Weekday remains the stronger predictor. 9/10.

**Task 2 (RTH-GLOB-004):** Good architecture — same CTE extension pattern as Task 1, gap_direction logic clean. One inversion bug: `CASE WHEN rth_open > rth_close THEN 'bullish'` is backwards (open > close = bearish). So `WHERE day_direction = 'bullish'` actually filtered to bearish days. Results are internally consistent for bearish days and still interpretable — gap-down bearish: 100% undercut EU mid / 93.55% EU low makes perfect structural sense (RTH opened below EU and kept selling). Rerun with `rth_close > rth_open` to get the original bullish-day hypothesis confirmed. Worth noting the bug rather than leaving it attributed to bullish days. 9/10.

**Task 3 (RTH-GLOB-002b):** Exactly the right mirror — one filter change, flag direction flipped. Good extension to both weekday and overall aggregations. Clean findings: 97.26% undercut EU midpoint, 87.67% undercut EU low on bearish days — near-universal. Wednesday standout: 100/100/20% (EU mid/EU low/EU high) — the cleanest "EU short entry is the best entry" weekday. Monday opposite: 69.23% trade above EU high — always wait for RTH on bearish Mondays. 10/10 (including the self-initiated extension).

---

## Week 29, Day 3 — Agent Feedback on Student

**Score: 20/20**

**Task 1 (RTH-GLOB-002):** Clean architecture — EU tick filter correct, MAX/MIN aggregation per trade_date clean, DISTINCT ON on eu_us_joint_agg harmless (one row per date already). Minor: uses d.low (full RTH low including open) instead of post-open low as specced — slightly overstates undercut rates but directionally valid and simpler. Methodology change is defensible. Key findings: 73.3% of bullish days undercut EU midpoint, 46.5% undercut EU low — nearly half of bullish days give better post-open entries than EU. Thursday 100% undercut EU mid on bullish days (12 days — needs more data). Monday opposite: RTH opens at 63% of EU range, only 40% undercut — EU is the entry window on Mondays. 10/10.

---

## Week 29, Day 2 — Agent Feedback on Student

**Score: 20/20**

**Task 1 (RTH-GLOB-001):** Two cuts executed cleanly in one go without step-by-step testing — both correct. LAG for prev_close correct, flat opens absorbed into gap_down (negligible). Key structural finding: gap direction predicts net daily direction but NOT intraday direction — intraday session reverses overnight move in both gap directions. Thursday is the only weekday where gap-down continues lower both intraday (45.5%) and net (18.2%). Monday gap-down strongest intraday reversal (77.8%). Gap-up Tuesday weakest net hold (44.4% close above prior close). The two-metric approach (open→close vs close vs prev_close) was the right design — tells completely different stories. 10/10.

---

## Week 29, Day 1 — Agent Feedback on Student

**Score: 18/20**

**Task 1 (RTH-SESS-008):** Smart use of rth_firsthour_rest_ohlc_ranges for fh_open (09:30 price). ABS() on all distance metrics correct. GAP direction CASE logic correct (prev_close > fh_open = gap_down). LAG computed on full table before Monday filter — correct behavior, gets the actual prior day's close. Key finding: 100% of Mondays dip below open; gap-down Monday is the cleaner setup (65 pt avg dip, 71.4% close above open vs gap-up 81 pt dip, 52.6%). 9/10.

**Task 2 (RTH-VOL-013):** Clean weekday extension of RTH-VOL-011 — just added weekday to GROUP BY, correct reuse of architecture. ts_recv AT TIME ZONE for window bucketing is a latent bug (should be ts_event) but negligible practical impact on MAX/MIN price. Wednesday standout: only 43.3% at 09:30 (lowest), median 100% at 14:00 (latest). Tuesday has slowest early expansion (67% at 10:30). Friday shows biggest single-window jump (09:30→10:30: 47.7%→74.2%). 9/10.

---

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
