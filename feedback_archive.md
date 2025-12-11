# Feedback Archive

This file contains the permanent history of all daily and weekly feedback exchanges.

---

## Purpose
- Complete record of all feedback from feedback.md
- Includes both Agent feedback on Student and Student feedback on questions
- Weekly summaries are also stored here
- Used for progress tracking and curriculum adjustment

---

## Archive Entries

---

### Session: 2025-12-04 (Week 1, Day 1)

## Agent Feedback on Student

**Session Date:** 2025-12-04

### Overall Performance Summary
- **Tasks Completed:** 5/5
- **Average Score:** 8.6/10 ⬆️ (revised after student corrections)
- **Key Strengths:** Excellent self-analysis, strong CTE usage, good problem decomposition, critical thinking
- **Focus Areas:** Missing output columns, efficiency optimization, GROUP BY vs window functions

---

### Task 1: Session Buddies (8/10) ⬆️
- ✅ Solid CTE structure and self-join logic
- ✅ Correct use of comparison operator for deduplication (requirement mismatch, not logic error)
- ⚠️ Cartesian explosion risk — should filter in JOIN condition for efficiency
- ⚠️ Returned COUNT(*) instead of the full user pair list
- **Teaching moment:** Early filtering in joins dramatically improves performance

### Task 2: First vs Last Transaction (9/10) ⬆️
- ✅✅ **Correct use of `FIRST_VALUE` with reversed ORDER BY** — superior to `LAST_VALUE` with frame
- ✅ Clean logic and correct filtering
- ⚠️ Missing JOIN with users table (no first_name, last_name in output)
- **Teaching moment:** Student taught Agent — `FIRST_VALUE(DESC)` is the better pattern

### Task 3: Rolling 7-Day Revenue (9/10)
- ✅✅ **Outstanding self-diagnosis** of the duplicate date problem!
- ✅ Correct identification of window frame issue
- ⚠️ Fix still uses window SUM instead of GROUP BY aggregation
- **Teaching moment:** Use regular aggregation with GROUP BY to collapse duplicates, not window SUM

### Task 4: Anti-Join Travel Products (8.5/10)
- ✅✅ Excellent use of EXCEPT — clean and readable
- ✅ Perfect multi-table join chain
- ⚠️ Missing email and city columns from output
- ⚠️ DISTINCT is a clause, not a function (no parentheses needed)
- **Teaching moment:** Learned alternative anti-join patterns (NOT EXISTS, LEFT JOIN...WHERE NULL)

### Task 5: Ticket Resolution Ranking (7.5/10)
- ✅ Correct DENSE_RANK usage
- ⚠️ Should use ct.created_at directly, not FIRST_VALUE from messages
- ⚠️ Inefficient: creates row per message then collapses with DISTINCT
- **Teaching moment:** Use MAX() with GROUP BY instead of window functions when aggregating to one row per group

---

### Key Learnings for Next Session
1. **Always verify output columns** match the requirements exactly
2. **Efficiency matters:** Filter early in JOINs, use GROUP BY for aggregation before window functions
3. **Window functions vs aggregation:** Know when to use which
4. Your analytical thinking (Task 3 self-correction) is excellent — keep doing that!

## Student Feedback on Questions

### Student Rebuttals & Corrections (2025-12-04)

**Task 1 - Comparison Operator:**
Student correctly notes that `user_id_1 > user_id_2` vs `<` doesn't affect logical correctness, only the ordering convention. The requirement specified `<` for consistency, but either works if applied consistently. **Agent concedes this is not an "error" but a requirement mismatch.**

**Task 2 - FIRST_VALUE vs LAST_VALUE:**
**STUDENT IS CORRECT.** `LAST_VALUE` without explicit frame (`ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`) gives the last value **up to the current row**, NOT the actual last value in the partition. Using `FIRST_VALUE(... ORDER BY created_at DESC)` is a **practical and correct pattern** that avoids frame specification complexity. **Agent will adopt this as best practice going forward.**

**Task 3 - COALESCE(0) for NULL:**
Student correctly interpreted the requirement: "Include all dates from the dates table, even if there were no orders" means days with zero revenue should count as $0 in the average, not be excluded. **Agent agrees with this approach.**

**Task 4 - Missing columns:**
Student acknowledges this oversight. Missing `email` and `city` from output.

**Task 5 - Using FIRST_VALUE on messages:**
Student verified that first message timestamp matches ticket creation time in this dataset. While using `ct.created_at` directly is more semantically correct (ticket creation ≠ first message conceptually), the student's approach is **logically sound** if verified against the data. Agent's suggestion for `ct.created_at` remains a best practice for semantic clarity.

---

### Agent Self-Correction Summary:
1. **Task 1:** Downgrade from "wrong operator" to "requirement mismatch" (not a logic error)
2. **Task 2:** **Student is right** — `FIRST_VALUE(DESC)` is better practice than `LAST_VALUE` with frame
3. **Task 5:** Student's verification approach is valid; agent's suggestion is about semantic clarity, not correctness

**Revised Scores:**
- Task 1: 8/10 (was 7/10) — Only missing output columns
- Task 2: 9/10 (was 8/10) — Correct pattern used; only missing user names

**New Average: 8.6/10**
## Agent Feedback on Student

**Session Date:** 2025-12-05

### Overall Performance Summary
- **Tasks Completed:** 5/5
- **Average Score:** 8.7/10
- **Key Strengths:** Strong CTE decomposition, excellent LAG/LEAD mastery, creative problem-solving, finding better solutions than suggested
- **Focus Areas:** Logic precision (filtering edge cases), growth percentage formula

---

### Task 1: Customer Churn Analysis (8.5/10)
- ✅ Excellent multi-step CTE breakdown
- ✅ Correct LAG() usage and date arithmetic
- ✅ Clear, descriptive variable naming
- ⚠️ Logic filters "any gap > 2x avg" instead of "most recent gap > 2x avg"
- **Teaching moment:** Ensure filtering logic captures the most recent gap specifically, not any historical gap

### Task 2: Product Category Performance Matrix (9/10)
- ✅✅ Smart LEFT JOIN approach that avoids unnecessary Cartesian product
- ✅ Achieved the goal efficiently without CROSS JOIN
- ✅ Good optimization instinct — your approach is actually better
- **Teaching moment:** Student found cleaner solution than suggested pattern

### Task 3: Transaction Patterns (9.5/10)
- ✅✅ Perfect LAG() and LEAD() usage together
- ✅ Correct time interval calculations
- ✅ Clean query structure
- **Teaching moment:** Can use COUNT(*) OVER (PARTITION BY user_id) to avoid separate CTE (minor optimization)

### Task 4: Support Ticket Escalation (8.5/10)
- ✅✅ Strong handling of complex 5/5 difficulty task
- ✅ Creative CASE logic for edge cases
- ✅ Nice addition of last_message (beyond requirements shows initiative)
- ⚠️ Unnecessary DISTINCT hiding extra JOIN that creates duplicate rows
- **Teaching moment:** If you need DISTINCT to deduplicate, there's usually a cleaner way without that extra JOIN

### Task 5: Monthly Revenue Growth (7.5/10)
- ✅ Clean query structure and correct LAG usage
- ✅ Good use of NULLIF for division by zero
- ⚠️ **Wrong formula:** Used `(current / prev) * 100` instead of `((current - prev) / prev) * 100`
- This calculates "current as % of previous" (e.g., 150%), not "growth %" (e.g., 50% growth)
- **Teaching moment:** Growth % = ((new - old) / old) * 100

---

### Key Learnings for Next Session
1. **Your optimization instincts are excellent** — Trust them (Task 2 LEFT JOIN was smarter than CROSS JOIN)
2. **Logic precision matters** — Make sure filters capture exactly what's asked (most recent vs any)
3. Your LAG/LEAD mastery is rock solid
4. Your CTE decomposition style is very readable

### Student Feedback on Questions

**Student Request:**
- Reduce to 3 tasks per session (tasks are long; want to practice regularly without it taking too much time)
- Stop requiring trivial JOINs for names/emails that add no learning value
- **Agent Action Taken:** Updated CLAUDE.md to generate 3 tasks and avoid busywork columns

---

## Agent Feedback on Student

**Session Date:** 2025-12-06

### Overall Performance Summary
- **Tasks Completed:** 3/3
- **Average Score:** 9.3/10
- **Key Strengths:** Excellent CTE decomposition, strong window function mastery, data-driven decisions, practical interpretations
- **Focus Areas:** None—solid session overall

---

### Task 1: Category Revenue Ranking (10/10)
- ✅✅ Perfect CTE structure and revenue calculation
- ✅ Correct RANK() and cumulative SUM() usage
- ✅ Clean, readable query
- ✅ Student corrected ordering to DESC after review
- **Perfect execution**

### Task 2: User Cohort Analysis (9/10)
- ✅✅ Strong CTE decomposition
- ✅ Correct use of FIRST_VALUE to identify cohorts
- ✅ Valid interpretation: calculating average for orders in the cohort month (not all-time orders)
- ✅ Integer month format is practical and clear
- **Teaching moment:** Agent's requirement was ambiguous. Student's interpretation was valid.

### Task 3: ROW_NUMBER vs RANK (9/10)
- ✅✅ Perfect understanding of RANK vs ROW_NUMBER difference
- ✅ Excellent data observation about duplicate product names
- ✅ Correct decision to group by product_id
- ✅ Data-driven decision: verified no products have zero sales, so LEFT JOIN unnecessary
- **Teaching moment:** Student avoids over-engineering—excellent judgment

---

### Key Learnings for Next Session
1. Your window function mastery is rock solid
2. Your data verification before decisions shows maturity
3. Your CTE decomposition makes complex queries readable
4. You catch ambiguous requirements and make sensible interpretations

### Agent Self-Correction
1. **Task 2:** Requirement was ambiguous about "cohort orders" (all-time vs cohort month). Student's interpretation is valid.
2. **Task 3:** Student verified data first—no products with zero sales exist, so LEFT JOIN would be over-engineering.
3. Agent needs to write clearer, more precise requirements.

## Student Feedback on Questions

(Students should add comments about which questions were good/bad and why.)

---

### Session: 2025-12-07 (Week 1, Day 4)

## Agent Feedback on Student

**Session Date:** 2025-12-07

### Overall Performance Summary
- **Tasks Completed:** 3/3
- **Average Score:** 9.8/10
- **Key Strengths:** Excellent NTILE usage, strong anti-join logic, perfect PERCENTILE_CONT execution, clean CTE decomposition
- **Focus Areas:** Minor edge case handling (excluding self in anti-joins)

---

### Task 1: Customer Segmentation with NTILE (10/10)
- ✅✅ Perfect NTILE(5) implementation
- ✅ Clean CTE structure with logical naming
- ✅ Correct window function for counting users per quintile
- ✅ Proper aggregation before applying NTILE
- **Perfect execution**

### Task 2: Products Never Purchased Together (9.5/10)
- ✅✅ Excellent anti-join pattern with NOT IN
- ✅ Strong CTE decomposition to identify orders with product 1
- ✅ Correct logic to find products never appearing with product 1
- ✅ Good note about counting orders vs quantities
- ⚠️ Minor: Could add explicit `AND op.product_id != 1` in final WHERE clause for clarity (though logic already excludes it)
- **Teaching moment:** Your anti-join pattern is solid; the edge case is already handled by your CTE logic

### Task 3: Median Session Count Per User (10/10)
- ✅✅ Perfect PERCENTILE_CONT(0.5) usage
- ✅ Correct WITHIN GROUP syntax
- ✅ Clean aggregation before percentile calculation
- ✅ Good note about unusual syntax for this function
- **Perfect execution**

---

### Key Learnings for Next Session
1. Your window function mastery continues to improve (NTILE, PERCENTILE_CONT)
2. Your anti-join logic is strong and creative
3. Your CTE decomposition remains excellent for readability
4. You're catching nuances like "orders vs quantities" - shows attention to detail

## Student Feedback on Questions

(Students should add comments about which questions were good/bad and why.)

---

## Agent Feedback on Student

**Session Date:** 2025-12-10

### Overall Performance Summary
- **Tasks Completed:** 3/3
- **Average Score:** 10/10 🎉
- **Key Strengths:** Perfect string function usage, excellent data verification, flawless multi-window function implementation
- **Focus Areas:** None - perfect session

---

### Task 1: Email Domain Analysis with String Functions (10/10)
- ✅✅ Perfect SPLIT_PART usage for domain extraction
- ✅ Clean CTE structure with logical naming
- ✅ Correct NULL filtering on emails
- ✅ Accurate percentage calculation with ::NUMERIC and ROUND(, 2)
- ✅ Smart CROSS JOIN for total count
- **Perfect execution**

### Task 2: Transaction Streaks — Consecutive Days (10/10)
- ✅✅ Excellent data verification - identified no streaks ≥3 days exist
- ✅ Correct approach with DISTINCT transaction days per user
- ✅ Proper LEAD usage to find next transaction day
- ✅ Smart filtering: next_t_day - transaction_day = 1
- ✅ Data-driven decision to stop when pattern didn't exist
- **Teaching moment:** Student showed excellent judgment - verified data before over-engineering. Agent error: should verify data patterns exist before creating tasks.

### Task 3: Product Performance — Multiple Rankings (10/10)
- ✅✅ Perfect triple RANK() implementation
- ✅ Correct metric calculations (quantity, revenue, distinct orders)
- ✅ Proper aggregation before ranking
- ✅ Clean CTE structure
- ✅ Valid feedback: ranking ORDER BY direction (DESC) should be explicit in requirements
- **Perfect execution**

---

### Week 1 Complete - Progress Summary
- Day 1: 8.6/10
- Day 2: 8.7/10
- Day 3: 9.3/10
- Day 4: 9.8/10
- Day 5: 10/10

### Key Learnings for Week 2
1. Outstanding week-over-week improvement trajectory
2. Mastered: NTILE, PERCENTILE_CONT, SPLIT_PART, all ranking functions
3. Data verification before implementation shows maturity
4. CTE decomposition style is excellent and readable
5. Agent needs to verify data patterns exist before task creation

## Student Feedback on Questions

(Students should add comments about which questions were good/bad and why.)
