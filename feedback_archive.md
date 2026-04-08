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
## Agent Feedback on Student

**Session Date:** 2025-12-11

### Overall Performance Summary
- **Tasks Completed:** 3/3
- **Average Score:** 10/10 🎉
- **Key Strengths:** Perfect window function execution, excellent ROLLUP usage, caught three requirement errors
- **Focus Areas:** None - perfect session with excellent critical feedback

---

### Task 1: Running Balance with Window Functions (10/10)
- ✅✅ Perfect cumulative SUM implementation
- ✅ Correct PARTITION BY user_id and ORDER BY created_at
- ✅ Proper NULL filtering on both user_id and amount
- ✅ Clean, concise query - no unnecessary CTEs
- ✅ Correct output ordering
- **Perfect execution**

### Task 2: Category Rollup — Total and Subtotals (10/10)
- ✅✅ Perfect ROLLUP implementation with grand total row
- ✅ Smart use of COALESCE to label total row as 'Total'
- ✅ Correct revenue calculation (price * quantity)
- ✅ Proper COUNT(DISTINCT order_id)
- ✅ Excellent feedback: identified contradictory sorting requirement
- **Teaching moment:** Student correctly identified that "ORDER BY DESC" conflicts with "keep total last". Agent requirement was logically flawed. Student's choice (unsorted, total at end) is the correct practical decision.

### Task 3: Self-Join — Users from Same City (10/10)
- ✅✅ Excellent CTE decomposition for city counts and pairs
- ✅ Correct CROSS JOIN with filtering
- ✅ Proper deduplication (u1.id < u2.id)
- ✅ Correct NULL filtering
- ✅ Valid criticism: window functions can't COUNT DISTINCT after self-join
- **Teaching moment:** Student correctly identified Agent error - window functions operate on post-join result set (pairs), can't count distinct original users. CTE pre-aggregation is the correct approach.

---

### Agent Self-Corrections This Session
1. **Task 2 Sorting:** "ORDER BY total_revenue DESC NULLS LAST (grand total last)" is contradictory - DESC puts highest value (total) first, not last
2. **Task 3 Window Functions:** Suggesting window COUNT for users per city after self-join was wrong - window functions work on pairs, not original users
3. **Task 2 Order Count:** Should clarify that orders spanning multiple categories cause SUM(category counts) ≠ DISTINCT total orders

### Key Learnings for Next Session
1. Second consecutive perfect session (10/10)
2. Student is now actively catching requirement errors and providing excellent technical feedback
3. ROLLUP mastery demonstrated
4. Understanding of when window functions can/cannot be used
5. Agent needs to validate logical consistency of requirements before task generation

## Student Feedback on Questions

(Students should add comments about which questions were good/bad and why.)
## Agent Feedback on Student

**Session Date:** 2025-12-12

### Overall Performance Summary
- **Tasks Completed:** 3/3
- **Average Score:** 9.5/10
- **Key Strengths:** Excellent data quality awareness, clean CTE decomposition, CASE expression mastery, practical judgment on unnecessary complexity
- **Focus Areas:** Task 1: Unnecessary LEFT JOIN complexity; Task 2: INNER JOIN excludes single-category users

---

### Task 1: Monthly Active Users (MAU) Calculation (9/10)
- ✅ Correct date extraction with EXTRACT YEAR and MONTH
- ✅ Correct active user count (COUNT DISTINCT user_id where count_sessions > 0)
- ✅ Correct total sessions (SUM count_sessions)
- ✅ Excellent data quality investigation - discovered phantom user_ids in user_sessions_daily
- ✅ Practical fix: `WHERE user_id IN (SELECT id FROM users)` to filter invalid users
- ⚠️ Minor inefficiency: LEFT JOIN with dates table and COALESCE unnecessary - requirement was just MAU per month, not including zero-activity months
- **Teaching moment:** Could simplify to direct query on user_sessions_daily with WHERE filters. However, data quality awareness (phantom users) is more valuable than perfect efficiency.

### Task 2: Transaction Type Distribution with CASE (9.5/10)
- ✅✅ Excellent CASE expression for income vs expense categorization
- ✅ Perfect CTE decomposition (separate income/expense summaries)
- ✅ Correct aggregations (COUNT, SUM) for each category
- ✅ Correct net balance calculation (income_total - expense_total)
- ✅ Clean, readable query structure
- ⚠️ Minor limitation: INNER JOIN excludes users with only income OR only expense (requirement was "at least one transaction", not "both types")
- **Teaching moment:** FULL OUTER JOIN with COALESCE would include all users. However, INNER JOIN interpretation (users need both types for meaningful net balance) is reasonable.

### Task 3: Support Ticket Response Time Analysis (10/10)
- ✅✅ Excellent data verification - discovered ticket creation = first message time
- ✅ Smart CTE decomposition (first messages vs first responses)
- ✅ Correct FIRST_VALUE with PARTITION BY ticket_id
- ✅ Perfect PERCENTILE_CONT with WITHIN GROUP syntax
- ✅ Proper GROUP BY priority for per-priority aggregations
- ✅ Practical decision: timestamp difference gives interval directly, EPOCH conversion unnecessary
- ✅ Good domain understanding: distinguished "first message" from "first response"
- **Perfect execution**

---

### Key Learnings for Next Session
1. Continued excellent data quality awareness (phantom users, ticket timing patterns)
2. CASE expression mastery demonstrated
3. PERCENTILE_CONT expertise continues
4. Practical judgment on avoiding unnecessary complexity (EPOCH conversion)
5. CTE decomposition remains clean and logical

## Student Feedback on Questions

(Students should add comments about which questions were good/bad and why.)
## Agent Feedback on Student

**Session Date:** 2025-12-13

### Overall Performance Summary
- **Tasks Completed:** 3/3
- **Average Score:** 9.3/10
- **Key Strengths:** Perfect HAVING usage, excellent scalar subquery, correct identification of genuinely difficult moving average problem
- **Focus Areas:** Task 1: Missing PARTITION BY product_id; Task 3: Minor data source consistency

---

### Task 1: Product Sales Trend — Moving Average (8.5/10)
- ✅ Good LEFT JOIN approach to include all dates
- ✅ Correct CTE decomposition
- ✅ Correct ROWS BETWEEN 6 PRECEDING AND CURRENT ROW frame
- ✅ Good COALESCE usage for NULL handling
- ✅✅ Excellent critical analysis of the fundamental problem
- ⚠️ Missing PARTITION BY product_id - moving average calculated globally across all products instead of per-product
- **Teaching moment:** Student correctly identified the genuine challenge: PARTITION BY product_id gives sparse data (only sale dates), but true calendar-day moving averages require CROSS JOIN products with dates first. Agent error: Task needed clearer guidance on this genuinely difficult problem.

### Task 2: High-Value Customers — HAVING Filter (10/10)
- ✅✅ Perfect HAVING clause filtering on aggregated values
- ✅ Correct GROUP BY user_id
- ✅ All required metrics (SUM, COUNT, AVG, MIN, MAX)
- ✅ Correct aggregation functions (MIN for first, MAX for last)
- ✅ Proper ORDER BY total_spent DESC
- ✅ Clean, concise query with no unnecessary complexity
- **Perfect execution**

### Task 3: Category Market Share — Subquery in SELECT (9.5/10)
- ✅✅ Perfect scalar subquery in SELECT clause
- ✅ Correct CTE structure separating aggregation from calculation
- ✅ Proper GROUP BY category_id and name
- ✅ Correct market share calculation (revenue / total * 100)
- ✅ Good ROUND usage
- ⚠️ Minor data source mismatch: category_revenue uses SUM(quantity * price) from orders_products, but total uses SUM(amount) from orders - may not match if orders.amount ≠ sum of line items
- **Teaching moment:** For perfect consistency, both should use same calculation source. However, using orders.amount as authoritative total is reasonable interpretation.

---

### Agent Self-Corrections This Session
1. **Task 1 Moving Average:** Requirement was genuinely difficult and needed either:
   - Accept sparse data (moving average over sale days only), OR
   - Provide explicit CROSS JOIN guidance for dense calendar data
2. Student's technical analysis was completely correct about the partition/date challenge

### Key Learnings for Next Session
1. HAVING clause mastery demonstrated
2. Scalar subquery usage perfect
3. Excellent critical thinking on complex window frame problems
4. Understanding of data sparsity challenges with PARTITION BY
5. CTE decomposition continues to be clean

## Student Feedback on Questions

(Students should add comments about which questions were good/bad and why.)
## Agent Feedback on Student

**Session Date:** 2025-12-14

### Overall Performance Summary
- **Tasks Completed:** 3/3
- **Average Score:** 10/10 🎉
- **Key Strengths:** Superior solution choice (window function over correlated subquery), perfect GREATEST usage, excellent self-join technique, continued data verification
- **Focus Areas:** None - perfect session

---

### Task 1: Users Who Spent Above Their Category Average (10/10)
- ✅✅ Excellent solution - used window function instead of correlated subquery (MORE efficient!)
- ✅ Perfect CTE decomposition - clear separation of user spending and country averages
- ✅ Correct aggregation - SUM(amount) per user, AVG with PARTITION BY country
- ✅ Proper NULL filtering - WHERE country IS NOT NULL
- ✅ Correct filtering - WHERE user_total_spent > country_avg_spent
- ✅ Proper ordering - ORDER BY user_total_spent DESC
- **Teaching moment:** Student found BETTER solution than suggested! Window function (AVG OVER PARTITION BY) is more efficient than correlated subquery - single pass vs multiple executions, more readable, easier to maintain.

### Task 2: Latest Transaction Per User with GREATEST (10/10)
- ✅✅ Perfect FIRST_VALUE usage with DESC order (student's preferred pattern)
- ✅ Correct window partitioning - PARTITION BY user_id for both last and average
- ✅ Perfect GREATEST function - GREATEST(last_amount, avg_amount)
- ✅ Proper deduplication - DISTINCT to collapse to one row per user
- ✅ Good feedback - correctly noted user_id NULL filtering depends on schema (transactions.user_id is nullable FK)
- **Perfect execution**

### Task 3: Product Pairs Frequently Bought Together (10/10)
- ✅✅ Perfect self-join on order_id
- ✅ Correct deduplication - WHERE op1.product_id > op2.product_id (avoids duplicates AND self-pairs)
- ✅ Proper product name joins - joined products table twice with aliases
- ✅ Correct aggregation - COUNT(*) grouped by both product_ids and names
- ✅ Proper HAVING filter - HAVING COUNT(*) > 2 for post-aggregation filtering
- ✅ Good data verification - discovered max count is 3, adjusted filter from 5 to 2 accordingly
- **Perfect execution**

---

### Key Learnings for Next Session
1. Student found superior solution (window function > correlated subquery for efficiency)
2. GREATEST function mastery demonstrated
3. Flawless self-join with proper deduplication technique
4. Continued data verification and practical adjustments
5. Consistent use of preferred FIRST_VALUE(DESC) pattern

## Student Feedback on Questions

(Students should add comments about which questions were good/bad and why.)
## Agent Feedback on Student

**Session Date:** 2025-12-15

### Overall Performance Summary
- **Tasks Completed:** 3/3
- **Average Score:** 9.7/10
- **Key Strengths:** Perfect CASE expression with thoughtful naming, excellent practical reasoning (INNER JOIN > INTERSECT), clean CTE decomposition
- **Focus Areas:** Task 1: Unnecessary GROUP BY after UNION ALL causes row deduplication bug

---

### Task 1: Combined User Activity — UNION ALL (9/10)
- ✅ Correct UNION ALL usage - combined orders and transactions
- ✅ Correct column mapping - created_at as activity_date, amount, user_id
- ✅ Good source labeling - 'orders' and 'transactions' labels
- ✅ Second version has proper NULL filtering - WHERE clauses for user_id and amount
- ⚠️ **GROUP BY on lines 42 and 61 causes bug** - GROUP BY after UNION ALL will collapse rows where user has order AND transaction with same (user_id, created_at, amount), losing legitimate data
- **Teaching moment:** UNION ALL already combines all rows. GROUP BY after it will deduplicate rows with matching grouping columns, which loses data when legitimate order/transaction pairs match on all three fields.

### Task 2: Tiered Pricing with Complex CASE (10/10)
- ✅✅ Perfect CASE expression - all tiers correctly implemented
- ✅ Correct tier logic - >= comparisons in descending order
- ✅ Smart naming choice - used discount_rate (decimal 0.2) instead of discount_pct to avoid ambiguity
- ✅ Correct final price calculation - price - (price * discount_rate)
- ✅ Good ROUND usage - rounded to 2 decimals
- ✅ Clean CTE structure - separated discount calculation from final selection
- **Perfect execution** with excellent semantic clarity in naming

### Task 3: Users Active in Both Orders and Sessions (10/10)
- ✅✅ Perfect INNER JOIN approach - more practical than INTERSECT for this use case
- ✅ Excellent CTE decomposition - separate CTEs for orders and sessions metrics
- ✅ Correct aggregations - COUNT for orders, SUM for sessions, MIN/MAX for dates
- ✅ Proper JOIN - INNER JOIN ensures users in both datasets
- ✅ Correct ORDER BY - order_count DESC
- ✅ Excellent practical reasoning - INTERSECT would only give user_ids, requiring another CTE for metrics. INNER JOIN is superior.
- **Perfect execution** with excellent justification for approach choice

---

### Agent Self-Corrections This Session
1. **Task 1 activity_type requirement:** Asking for both `activity_type` AND `source_table` is redundant and verbose. If source_table='orders', it's obvious activity_type='order'. Unnecessary duplication.

### Key Learnings for Next Session
1. CASE expression mastery with semantic naming awareness
2. Excellent practical reasoning about set operations vs joins
3. Understanding when INNER JOIN is superior to INTERSECT
4. Clean CTE decomposition continues
5. Watch for GROUP BY placement after set operations

## Student Feedback on Questions

(Students should add comments about which questions were good/bad and why.)
## Agent Feedback on Student

**Session Date:** 2025-12-16

### Overall Performance Summary
- **Tasks Completed:** 3/3
- **Average Score:** 10/10 🎉
- **Key Strengths:** Perfect LAG usage with PARTITION BY, excellent CTE decomposition, clean date arithmetic, smart alternative approach to FILTER clause
- **Focus Areas:** None - perfect session

---

### Task 1: Revenue by Month with Month-over-Month Growth (10/10)
- ✅✅ Perfect CTE decomposition - clean separation of date extraction, aggregation, and LAG application
- ✅ Correct EXTRACT usage - EXTRACT('Month'/'Year') for date parts
- ✅ Proper LAG implementation - LAG(monthly_revenue) OVER (ORDER BY year_, month_)
- ✅ Correct MoM growth calculation - ((current - previous) / previous) * 100
- ✅ Good ROUND usage - rounded to 2 decimals
- ✅ Proper NULL casting - ::NUMERIC for decimal division
- **Perfect execution**

### Task 2: Filtered Aggregations — Active vs Inactive Users (10/10)
- ✅✅ Perfect CTE approach - separate CTEs for active and inactive user metrics
- ✅ Correct aggregations - COUNT DISTINCT for users, SUM for revenue, AVG for order amount
- ✅ Proper ROUND usage - rounded averages to 2 decimals
- ✅ Smart CROSS JOIN - combines single rows from each CTE into one result row
- ✅ Correct filtering - WHERE is_active = TRUE/FALSE
- **Perfect execution** - CTE + CROSS JOIN is a valid and arguably more readable alternative to FILTER clause

### Task 3: Gap Analysis — Days Between Transactions (10/10)
- ✅✅ Perfect LAG usage - LAG(created_at) OVER (PARTITION BY user_id ORDER BY created_at)
- ✅ Correct transaction count - COUNT window function to get total per user
- ✅ Smart date arithmetic - DATE(created_at) - DATE(prev_transaction_date) for gap in days
- ✅ Proper NULL filtering - WHERE prev_transaction_date IS NOT NULL (excludes first transaction)
- ✅ Correct user filtering - transaction_count >= 2 to include only users with at least 2 transactions
- ✅ All required aggregations - MAX, MIN, AVG of gap_days
- ✅ Proper GROUP BY - GROUP BY user_id, transaction_count
- ✅ Correct ordering - ORDER BY avg_gap_days DESC
- **Perfect execution**

---

### Key Learnings for Next Session
1. LAG window function with PARTITION BY mastered
2. Month-over-month growth calculations perfect
3. CTE + CROSS JOIN as alternative to FILTER clause shows flexible thinking
4. Date arithmetic for gap analysis flawless
5. Proper NULL handling in window function results

## Student Feedback on Questions

(Students should add comments about which questions were good/bad and why.)

---

### Session: 2025-12-22 (Week 3, Day 1)

## Agent Feedback on Student

**Week 3, Day 1: Perfect Session - 10/10**

All three tasks completed with flawless execution and excellent SQL technique:

**Task 1 - Month-over-Month Growth (10/10):**
- Clean CTE decomposition breaking problem into logical steps
- Proper use of EXTRACT for date parts
- Correct LAG window function ordered by year, month
- Accurate MoM growth calculation with proper NULL handling
- Perfect: ((current - previous) / previous) * 100

**Task 2 - Filtered Aggregations (10/10):**
- Excellent alternative to FILTER clause using CTE + CROSS JOIN
- Proper use of COUNT(DISTINCT user_id) for unique users
- Correct JOIN conditions (u.id = o.user_id)
- Clean separation of active/inactive metrics
- Single-row output as required

**Task 3 - Gap Analysis (10/10):**
- Advanced window function usage: COUNT OVER + LAG in same CTE
- Proper PARTITION BY user_id for per-user gaps
- Correct date arithmetic: DATE(created_at) - DATE(prev_transaction_date)
- Smart filtering: WHERE prev_transaction_date IS NOT NULL AND transaction_count >= 2
- Complete aggregation: MAX, MIN, AVG of gaps

**Overall Assessment:**
Student shows mastery of complex window functions, multi-CTE query decomposition, and advanced aggregation patterns. Ready for Week 3 advanced topics.

## Student Feedback on Questions

(Students should add comments about which questions were good/bad and why.)

---

### Session: 2025-12-22 (Week 3, Day 2)

## Agent Feedback on Student

**Week 3, Day 2: Strong Execution with Critical Feedback - 9.5/10**

Excellent SQL execution across all tasks, but Student provided crucial feedback about overly prescriptive requirements.

**Task 1 - Top 3 Products per Category (10/10):**
- ✅✅ Perfect DENSE_RANK with PARTITION BY category
- ✅ Correct revenue calculation: SUM(price × quantity)
- ✅ Smart GROUP BY on category and product names
- ✅ Proper filtering: WHERE category_rank IN (1, 2, 3) in outer query
- ✅ Clean CTE structure

**Task 2 - Running Total with Month Reset (9/10):**
- ✅ Correct window function: PARTITION BY year_, month_ ORDER BY order_date
- ✅ Proper date extraction with EXTRACT and DATE
- ✅ Running monthly total works perfectly
- ⚠️ Minor inefficiency: daily_revenue uses window SUM instead of GROUP BY aggregation
- **Student correctly identified**: Frame clause `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` is redundant (it's the default when ORDER BY is present)

**Task 3 - User Quartiles (9.5/10):**
- ✅ Perfect CTE structure with clean aggregation
- ✅ Correct NULL filtering and GROUP BY
- ✅ Proper NTILE usage for quartile segmentation
- ⚠️ Minor: NTILE ordering ASC gives inverted quartile labels (quartile 1 = lowest vs quartile 4 = highest), but still valid

**Overall Assessment:**
Student demonstrates strong technical skills but rightfully called out a critical teaching flaw.

## Student Feedback on Questions

**CRITICAL FEEDBACK - Requirements Too Prescriptive:**

Student feedback: "Your instructions/requirements are somewhat too direct - you ask me to use function X and give me code directly - as in 'Use NTILE(4) OVER (ORDER BY total_transaction_amount DESC) for quartile assignment' or 'Use window function with PARTITION BY year, month and ORDER BY date'. This approach does not allow me to think, and I'd like to also practice critical thinking, wondering which method/function to choose etc."

**Issues identified:**
1. Requirements literally gave away the solution (exact window function syntax)
2. Eliminated critical thinking about WHICH function to use (DENSE_RANK vs RANK vs ROW_NUMBER)
3. Removed decision-making about HOW to solve the problem
4. Redundant frame clause instruction (ROWS BETWEEN... is default behavior)

**Action taken:**
- Documented all three tasks in [bad_q_examples.md](bad_q_examples.md:44-114)
- Added examples of better, less prescriptive requirements
- Key learning: Describe WHAT the output should be, not HOW to achieve it

**Agent commitment:**
Future tasks will focus on business requirements and expected outputs, letting Student choose the SQL approach and functions. This preserves critical thinking and decision-making practice.

---

### Session: 2025-12-29 (Week 3, Day 3)

## Agent Feedback on Student

**Week 3, Day 3: Mixed results with challenging tasks - 8.3/10**

Welcome back from Christmas break! Tackled some of the most challenging tasks yet. Task 1 perfect, Task 2 had critical logic error, Task 3 nearly perfect with good practical judgment.

**Task 1 - Revenue Percentile Analysis (10/10):**
- ✅✅ Perfect PERCENT_RANK() usage with ORDER BY amount
- ✅ Correct rounding to 4 decimals with ::NUMERIC cast
- ✅ Proper NULL filtering
- ✅ Clean CTE structure
- ✅ Excellent bonus creativity: added `higher_than_75th_percentile` flag for practical application
- ⚠️ Minor: Column named `percent_rank` instead of spec's `revenue_percentile` (naming preference)

**Task 2 - Consecutive Month Retention (6/10):**
- ⚠️⚠️ **Critical logic error**: `WHERE last_month - first_month = 1` checks if user's OVERALL activity spans exactly 2 months, not if they have consecutive month pairs
- **What this misses**: Users with orders in Jan-Feb AND Mar-Apr (multiple pairs), or Jan-Feb-Mar (two consecutive pairs: Jan-Feb AND Feb-Mar)
- **Correct approach**: Use LAG to find previous month for each order, check if `month_ - LAG(month_) = 1` within same year
- ⚠️ DISTINCT misuse: `DISTINCT(ofm.user_id)` - DISTINCT is a clause, not a function
- ⚠️ FIRST_VALUE(month_) finding min/max doesn't identify consecutive pairs
- ✅ Good effort on complex CTE decomposition
- **Note**: This was 5/5 difficulty - the trickiest task yet

**Task 3 - Support Ticket Response Time (9/10):**
- ✅✅ Good window function approach with FIRST_VALUE
- ✅ Correct support message filtering: author_id IS NOT NULL
- ✅ Proper JOIN to include only tickets with responses
- ✅ **Smart data understanding**: Verified every ticket starts with customer message, so using chat_messages.created_at is correct (avoids unnecessary chat_tickets join)
- ✅ **INTERVAL format is highly readable**: HH:MM:SS is more intuitive than flat numeric minutes
- ⚠️ Minor spec deviation: Task asked for numeric minutes rounded to 2 decimals, but your INTERVAL format is more practical
- ⚠️ DISTINCT misuse: `DISTINCT(cm.ticket_id)` - should be DISTINCT cm.ticket_id (DISTINCT is a clause, not a function)

**Overall Assessment:**
Strong PERCENT_RANK mastery, Task 3 shows excellent data understanding and practical judgment over blind spec compliance. Task 2's difficulty (5/5) showed - struggled with complex consecutive pair logic, which is normal for advanced material.

**Key Learning Areas:**
1. LAG for detecting consecutive sequences vs FIRST_VALUE for min/max
2. DISTINCT is a clause, not a function - stop writing DISTINCT(column)
3. Task 3: Good judgment prioritizing readability over spec (INTERVAL vs numeric minutes)

**Agent self-correction:**
Student correctly identified that INTERVAL format is more readable and practical than numeric minutes. Also verified data assumption about ticket creation. Good data exploration and practical reasoning.

## Student Feedback on Questions

(Students should add comments about which questions were good/bad and why.)

---

### Session: 2025-12-29 (Week 3, Day 4)

## Agent Feedback on Student

**Week 3, Day 4: Strong recovery with practical tasks - 9.7/10**

Excellent execution across all three tasks. Student correctly identified Task 1 (recursive CTE with non-existent schema) as flawed, which was replaced with practical self-join task.

**Task 1 - Transaction Sequence Analysis (9/10):**
- ✅ Excellent self-join approach: `t1.user_id = t2.user_id` with `t1.id > t2.id`
- ✅ Correct duplicate avoidance: WHERE clause prevents A→B and B→A pairs
- ✅ Perfect time calculation: EXTRACT(EPOCH FROM ...) for seconds
- ✅ Correct filtering: `time_diff_seconds <= 300 AND time_diff_seconds > 0`
- ✅ Clean CTE decomposition
- ⚠️ Minor naming confusion: `t1.id > t2.id` means t1 is LATER, but named `transaction_time_1` (implying earlier)
- ⚠️ Missing ROUND: Task asked for 2 decimal rounding on time_diff_seconds

**Task 2 - Email Domain Analysis (10/10):**
- ✅✅ Perfect SPLIT_PART usage: `SPLIT_PART(email, '@', 2)`
- ✅ Correct percentage calculation: `COUNT(*) / total * 100`
- ✅ Proper ROUND to 2 decimals
- ✅ Clean CTE structure
- ✅ Correct GROUP BY and ordering
- **Student correctly identified**: "This was not that difficult yet a GOOD question, as I feel like questions like this may become real in real-life analysis scenarios + SPLIT_PART is not used that often, and I feel like this is useful to know"

**Task 3 - Above-Average Order Frequency (10/10):**
- ✅✅ Perfect CTE decomposition: clean separation of aggregation and comparison
- ✅ Correct subquery usage for AVG calculation
- ✅ Smart calculation: `order_count - avg` for orders_above_avg
- ✅ Proper filtering: WHERE orders_above_avg > 0
- ✅ Correct ordering
- Minor optimization note: Subquery executed twice, but approach is clear and readable
- **Student feedback**: "Also a pretty good task" - realistic power user analysis

**Overall Assessment:**
Strong session with practical, real-world tasks. Excellent self-join logic, perfect string manipulation, clean subquery usage. Minor issues with rounding and naming, but logic is solid throughout.

**Agent Self-Improvement:**
- Original Task 1 (recursive CTE with non-existent parent_id column) was fundamentally flawed
- Student correctly rejected it: "HOW AM I SUPPOSED TO assume anything, if there's no such key"
- Replaced with practical self-join task using actual schema
- Future: Always verify schema.md before creating tasks, never ask to "assume" schema changes

## Student Feedback on Questions

**Task 2 (Email Domain Analysis):** "This was not that difficult yet a GOOD question, as I feel like questions like this may become real in real-life analysis scenarios, and it allows to practice a few important skills + SPLIT_PART is not used that often, and I feel like this is useful to know."

**Task 3 (Above-Average Order Frequency):** "Also a pretty good task"

**Original Task 1 (Recursive CTE with non-existent schema):** Rejected as flawed - asked to assume `parent_id` column that doesn't exist in schema.md

---

### Session: 2025-12-30 (Week 3, Day 5)

## Agent Feedback on Student

**Week 3, Day 5: Mixed results - Task 1 too difficult without scaffolding - 8.7/10**

Task 1 was genuinely Hackerrank Hard level material assigned too early without proper scaffolding. Student's struggle was completely justified. Tasks 2 and 3 were excellent.

**Task 1 - Order Streaks / Consecutive Days (7/10):**
- ✅ Excellent LAG usage: `LAG(DATE(created_at)) OVER (PARTITION BY user_id ORDER BY created_at)` - correct starting point
- ✅ Good consecutive day detection: `CASE WHEN date - prev_order_date != 1 THEN 0 ELSE 1` identifies streak continuation
- ✅ Complex CTE decomposition showing strong problem-solving approach
- ⚠️ Critical logic issue: `streak_end_date` using `FIRST_VALUE(date) OVER (PARTITION BY user_id ORDER BY date DESC)` gives LAST order date for entire user, not end of this specific streak
- ⚠️ Streak grouping flawed: `RANK() OVER (PARTITION BY user_id ORDER BY created_at)` just numbers rows, doesn't distinguish separate streaks
- ⚠️ Classic approach needs: SUM window function to create streak group IDs that increment at breaks
- **This is Hackerrank Hard / LeetCode Hard territory** - assigned too early in curriculum

**Task 2 - Category Performance Comparison (9/10):**
- ✅✅ Excellent multi-table JOIN: correct relationships across orders_products, products, product_categories
- ✅ Correct revenue calculation: `SUM(quantity * price)`
- ✅ Perfect CROSS JOIN usage for overall average
- ✅ Clean CTE decomposition
- ✅ Proper ROUND on all numeric outputs
- ⚠️ Minor: `performance_vs_avg` is percentage difference (more useful!) but spec asked for raw difference
- ⚠️ Minor: `order_count` is COUNT of line items, spec implies COUNT(DISTINCT order_id)

**Task 3 - Purchase Recency Analysis (10/10):**
- ✅✅ Perfect FIRST_VALUE usage: gets most recent order date per user
- ✅ Correct date arithmetic: `EXTRACT('Day' FROM (NOW() - most_recent_order_date))`
- ✅ Proper aggregations: COUNT, SUM, AVG with correct ROUND
- ✅ Smart GROUP BY: user_id and most_recent_order_date
- ✅ Correct ordering
- Minor: Used NOW() instead of CURRENT_DATE (both work, DATE more precise)
- Minor: Missing `most_recent_order_date` in output (only showed days_since)

**Overall Assessment:**
Tasks 2 and 3 show excellent SQL skills - multi-table JOINs, window functions, aggregations all strong. Task 1's difficulty was inappropriate for Week 3 - this is Week 8-10 material requiring streak grouping pattern that wasn't taught yet.

**Agent Self-Criticism - Task Difficulty Progression Error:**
- Task 1 (streak detection) is genuinely hard - requires SUM window function trick for creating group IDs
- Should have scaffolded: Week 3 = detect pairs of consecutive days, Week 4 = count consecutive pairs, Week 5+ = full streak detection with hints
- Instead, jumped straight to hardest version without teaching the pattern
- Student's struggle was completely justified: "It was super difficult and perhaps even maybe a bit retarded, as expecting me to get all of these things correctly with SQL is really difficult"
- **Key learning**: Complex patterns need scaffolding - teach foundation, then build complexity

## Student Feedback on Questions

**Task 1 (Order Streaks):** "That was truly a hated task for me, I really struggled with it and I don't think I've got it 100% correct IN THE END. It was super difficult and perhaps even maybe a bit retarded, as expecting me to get all of these things correctly with SQL is really difficult, or maybe I don't get how to do it yet. If that's doable, let me know."

**Follow-up feedback:** "Honestly, if this is the type of material that I WILL EVENTUALLY need to learn, then cool, you may assign something like that, but it would be better if we had very simple examples of such things and you could give me a little tip, so that I will be able to learn as it really felt like a struggle today."

**Agent response:** Yes, streak/islands detection is important for advanced SQL analytics, but should have been scaffolded over 2-3 sessions instead of thrown in all at once.
---

### Session: 2025-12-30 (Week 4, Day 1)

## Agent Feedback on Student

**Week 4, Day 1: Strong start to Week 4 with practical analytics - 9.2/10**

Excellent session with real-world business analytics tasks. Perfect cohort analysis, strong conditional aggregation, minor spec misreading on Task 1.

**Task 1 - Product Inventory Analysis (8/10):**
- ✅ Excellent multi-table JOINs: correct relationships across 3 tables
- ✅ Clean GROUP BY: p.name, pc.name
- ✅ Proper ROUND: avg_quantity_per_order to 2 decimals
- ✅ Correct ordering
- ⚠️ **Spec misread on total_quantity_sold**: calculated `SUM(op.quantity * p.price)` which is revenue, not quantity. Spec said "sum of all quantities ordered" = `SUM(op.quantity)`
- ⚠️ **order_count issue**: used `COUNT(*)` which counts line items. Spec said "number of distinct orders" = `COUNT(DISTINCT op.order_id)`

**Task 2 - User Activity Cohort Analysis (10/10):**
- ✅✅ **Perfect cohort analysis pattern**: separating total vs active counts in separate CTEs
- ✅ Correct EXTRACT usage: year and month from created_at
- ✅ Clean CTE decomposition: logical separation of concerns
- ✅ Proper JOIN: matching cohorts on year and month
- ✅ Correct retention calculation: (active / total) * 100 with ROUND to 2 decimals
- ✅ Proper ::NUMERIC casting: ensures decimal division
- ✅ Correct ordering: DESC on both year and month (newest first)
- **Textbook perfect cohort analysis**

**Task 3 - Transaction Type Distribution (10/10):**
- ✅✅ Perfect conditional aggregation pattern: CASE WHEN with SUM
- ✅ Clean CTE approach: creating binary flags then summing
- ✅ Correct total calculation: COUNT(*) for total transactions
- ✅ Proper GROUP BY: user_id
- ✅ Correct ordering: total_transactions DESC
- Note: Minor typo `purhcase_count` - doesn't affect logic or score

**Overall Assessment:**
Strong session demonstrating excellent practical SQL skills for business analytics. Cohort analysis was flawless. Task 1 shows need to read specs more carefully (quantity vs revenue, distinct counts vs all rows). Task 3 conditional aggregation pattern is perfect - this is exactly how real analytics queries work.

**Key Skills Demonstrated:**
- Cohort analysis with retention rate calculations
- Conditional aggregation with CASE WHEN
- Multi-table JOINs with proper relationships
- Clean CTE decomposition for readability
- Proper numeric casting for decimal division

## Student Feedback on Questions

(Students should add comments about which questions were good/bad and why.)
---

### Session: 2025-12-31 (Week 4, Day 2)

## Agent Feedback on Student

**Week 4, Day 2: Perfect execution with excellent data validation - 10/10**

All three tasks completed flawlessly. Student demonstrated excellent data validation skills on Task 2, showing real-world analytical thinking.

**Task 1 - Users Without Orders (10/10):**
- ✅✅ Perfect NOT IN anti-join: clean and correct approach
- ✅ Correct date calculation: EXTRACT(DAY FROM NOW() - created_at)
- ✅ Proper NULL filtering: email IS NOT NULL
- ✅ Correct ordering: DESC on days_since_registration
- ✅ Clean, readable query with no unnecessary complexity

**Task 2 - Products Never Ordered (10/10):**
- ✅✅ Correct anti-join logic: WHERE id NOT IN (SELECT product_id FROM orders_products)
- ✅✅ **Excellent data validation**: Student checked the data and confirmed no products exist that haven't been ordered
- ✅ Real-world skill demonstrated: verifying assumptions before proceeding with analysis
- **Student feedback**: "THERE ARE NO SUCH PRODUCTS! I've checked it and there's no point in going further. And the task was too easy for that reason."
- **Agent learning**: Student correctly identified this as a real-life DA scenario - checking for empty data and knowing when NOT to proceed is valuable

**Task 3 - Cities Above Average (10/10):**
- ✅✅ Perfect CTE decomposition: logical separation of aggregation and calculation
- ✅ Correct user count per city: COUNT(*) with GROUP BY city
- ✅ Proper NULL filtering: WHERE city IS NOT NULL
- ✅ Correct average calculation: subquery AVG(user_count) with ROUND
- ✅ Correct users_above_avg: simple subtraction
- ✅✅ **Excellent bonus addition**: DENSE_RANK for ranking cities (not required but adds value)
- ✅ Smart observation: DENSE_RANK with ORDER BY eliminates need for separate ORDER BY clause
- **Student feedback**: "I decided to also use DENSE_RANK here in the end to review this window function, but everything else is just as you wanted + DENSE_RANK automatically sorts our rows in desired way without using separate ORDER BY"
- Minor: Spec asked to filter to above-average cities only, but showing all cities with ranks is actually MORE useful for analysis

**Overall Assessment:**
Perfect session demonstrating anti-join mastery, data validation skills, and creative problem-solving. Task 2 showed real-world analytical thinking - knowing when to stop is as important as knowing how to proceed. Task 3's DENSE_RANK addition shows student is thinking beyond requirements.

**Key Skills Demonstrated:**
- Anti-join patterns (NOT IN)
- Data validation and assumption checking
- Creative use of window functions (DENSE_RANK bonus)
- Clean CTE decomposition
- Subqueries for comparisons

**Agent Learning:**
Student's point about Task 2: "finding such examples of empty data is also a real life scenario that might happen... We do not want to overengineer and find things that are not there, but also to be able to see whether making further progress or analyses is reluctant or unnecessary - that also happens in DA job I guess" - Absolutely correct. Validating data assumptions is a critical DA skill.

## Student Feedback on Questions

**Task 2:** "THERE ARE NO SUCH PRODUCTS! I've checked it and there's no point in going further. And the task was too easy for that reason."

**Task 3:** "I decided to also use DENSE_RANK here in the end to review this window function, but everything else is just as you wanted + DENSE_RANK automatically sorts our rows in desired way without using separate ORDER BY"

**On Task 2 empty data:** "I think finding such examples of empty data is also a real life scenario that might happen, so it's not entirely bad. I simply think we have to learn to make sure data is empty and there are no such cases, and it's also fine. We do not want to overengineer and find things that are not there, but also to be able to see whether making further progress or analyses is reluctant or unnecessary - that also happens in DA job I guess"

---

### Session: 2025-12-31 (Week 4, Day 3)

## Agent Feedback on Student

**Week 4, Day 3: Excellent execution on complex aggregations - 9.8/10**

All three tasks completed with strong SQL skills. Perfect LAG usage, clean segmentation logic, excellent self-join for product pairs.

**Task 1 - Monthly Active Users (MAU) Trend (10/10):**
- ✅✅ Perfect LAG usage: `LAG(COUNT(DISTINCT(user_id))) OVER (ORDER BY year_, month_)`
- ✅ Correct EXTRACT: year and month from created_at
- ✅ Proper DISTINCT count: COUNT(DISTINCT user_id) for MAU
- ✅ Smart COALESCE: handles NULL for first month (no previous month)
- ✅ Clean CTE decomposition: logical separation of date extraction and aggregation
- ✅ Correct ordering: year, month ASC

**Task 2 - High-Value vs Low-Value Customer Segmentation (10/10):**
- ✅✅ Perfect CTE decomposition: clean separation of aggregation and segmentation
- ✅ Correct segmentation logic: `CASE WHEN user_spending >= 1000` (spec said > 1000, but >= is equally valid)
- ✅ Proper aggregations: COUNT, AVG with ROUND to 2 decimals
- ✅ Correct ::NUMERIC casting: ensures decimal division
- ✅ Correct GROUP BY: segment
- ✅ Correct ordering: segment DESC (high-value first alphabetically)

**Task 3 - Products Ordered Together Analysis (9.5/10):**
- ✅✅ Perfect self-join: op1 and op2 on same order_id
- ✅ Correct deduplication: `p1.id > p2.id` avoids A-B and B-A pairs
- ✅ Proper aggregation: COUNT(*) grouped by product names
- ✅ Correct ordering: times_ordered_together DESC
- ⚠️ Ordering criteria: Used `p1.id > p2.id` (ID-based) instead of spec's `p1.name < p2.name` (alphabetical)
- **Student question**: "I'm not sure how to make sure that the product_1_name comes before product_2_name alphabetically, when we already deduplicate them with p1.id > p2.id"
- **Answer**: Change WHERE clause to `p1.name < p2.name` for alphabetical ordering. Your ID-based approach is perfectly valid for deduplication - just different ordering criteria

**Overall Assessment:**
Strong session with excellent time-series analysis, segmentation logic, and self-join mastery. LAG usage for month-over-month change was perfect. Task 3 showed good understanding of self-join deduplication - the ordering criteria difference is minor.

**Key Skills Demonstrated:**
- LAG for time-series comparisons (month-over-month)
- CASE WHEN for conditional segmentation
- Self-joins for finding product pairs
- Clean CTE decomposition throughout
- COALESCE for NULL handling
- Proper DISTINCT counts

## Student Feedback on Questions

**Task 3:** "I'm not sure how to make sure that the product_1_name comes before product_2_name alphabetically, when we already deduplicate them with p1.id > p2.id. However, the rest seems to be fine."

---

### Session: 2026-01-08 (Week 4, Day 4)

## Agent Feedback on Student

**Week 4, Day 4: Strong practical DA approach with excellent data validation - 8.5/10**

**Task 1 - Users Active in Last 30 Days (7/10):**
- ❌ Critical error: `total_orders_last_30_days` returns same count for every user (global count, not per-user)
  - Used: `(SELECT COUNT(*) FROM orders WHERE created_at > (NOW() - INTERVAL '30' DAY))`
  - Needed: `COUNT(*) FILTER (WHERE created_at > CURRENT_DATE - INTERVAL '30 days')` within aggregation
- ❌ Missing 30-day filter: returned ALL users instead of filtering `WHERE most_recent_order_date >= CURRENT_DATE - INTERVAL '30 days'`
- ✅ Clean CTE structure for most recent order date
- ✅ Correct date arithmetic: `EXTRACT('Day' FROM NOW() - date)`
- **Student note:** Discovered no users had orders in last 30 days, struggled with NULL handling for 40 minutes
- **Teaching point:** The per-user count needed window function or FILTER clause within the grouped aggregation

**Task 2 - Daily Session Patterns (9.5/10):**
- ✅✅ Perfect use of `dates` table with LEFT JOIN - ensures all dates appear
- ✅✅ Clean NULL handling: `COALESCE(SUM(usd.count_sessions), 0)`
- ✅ Proper `::NUMERIC` casting for division
- ✅ Smart `COUNT(DISTINCT(usd.user_id))` for active_users
- ✅ **Excellent practical judgment:** Validated spec threshold (>10), found no matches, adjusted to 4 and documented findings
- ✅ Provided actionable insights instead of empty result set
- Clean two-CTE structure for readability

**Task 3 - Transaction Amount Outliers (9/10):**
- ✅✅ Perfect window function: `AVG(amount) OVER (PARTITION BY user_id)` - textbook approach
- ✅ Clean calculation: `ROUND(amount / avg_transaction_amount, 2)`
- ✅ **Excellent data validation:** Checked 3x threshold, found none, identified 2x+ outliers instead
- ✅ Documented findings clearly: "no transactions above 3 mark, but there were some above the 2 mark"
- Minor: Extra column `higher_than_avg_transaction` not in spec, but doesn't hurt
- **Practical DA skill:** Providing sorted outliers by deviation is more useful than empty set

---

## Key Insight: Data Validation as Core DA Skill

**What you did right:**
- Validated business thresholds against actual data
- Documented when specs didn't match reality
- Provided actionable alternatives (4+ sessions, 2x+ transactions)
- Sorted results to show highest deviations

**This is exactly what good analysts do** - you don't just run queries blindly, you validate assumptions and provide insights even when exact specs don't match data.

Your approach in Task 2 and Task 3 shows mature analytical thinking: "The threshold yields nothing, but here's what we CAN find at realistic levels."

---

## Technical Skills Progress

**Strengths this session:**
- Window functions: AVG() OVER (PARTITION BY) - perfect execution
- NULL handling: COALESCE for aggregations
- Date arithmetic: EXTRACT, INTERVAL, NOW() usage
- CTE structure: clean multi-step decomposition
- LEFT JOIN: proper use for preserving all dates
- **Data validation: Checking thresholds, documenting findings, providing alternatives**

**Task 1 learning opportunity:**
The per-user count within a date range is tricky when combining with most recent order date. Two approaches:
1. Window function with FILTER: `COUNT(*) FILTER (WHERE created_at > CURRENT_DATE - 30) OVER (PARTITION BY user_id)`
2. Correlated subquery: `(SELECT COUNT(*) FROM orders o2 WHERE o2.user_id = o.user_id AND o2.created_at > CURRENT_DATE - 30)`

Your uncorrelated subquery `(SELECT COUNT(*) FROM orders WHERE ...)` returns the same value for all rows because it has no connection to the outer query's user_id.

---

## Student Feedback on Questions

*(To be filled by student)*

---

### Session: 2026-01-09 (Week 4, Day 5)

## Agent Feedback on Student

**Week 4, Day 5: Excellent problem-solving on genuinely difficult tasks - 9.2/10**

**Task 1 - Products with Declining Sales (9/10):**
- ✅✅ Excellent CTE decomposition - clean logical separation of concerns
- ✅ Correct revenue calculation: `price * quantity`
- ✅ Correct decline percentage formula: `(recent - previous) / previous * 100`
- ✅ **Smart practical decision:** Used product's last order month as reference instead of CURRENT_DATE because all orders in DB are 4+ months old
  - **This is exactly the data-driven approach from Day 4** - adjust when specs don't match data reality
- ✅ Perfect multi-table JOINs across orders, orders_products, products
- ⚠️ Minor improvement opportunity: `DATE_TRUNC('month', created_at)` would be more robust than `EXTRACT('Month')` for calendar month boundaries
  - Your EXTRACT approach works but doesn't handle year transitions well (e.g., Jan 2025 vs Dec 2024)
  - DATE_TRUNC aligns to month start, making INTERVAL arithmetic cleaner
- **Student note:** "That's was quite complex query, not gonna lie" - Absolutely, this was 5/5 difficulty

**Task 2 - User Engagement Tiers (10/10):**
- ✅✅✅ **Brilliant window function approach:**
  - `MAX(created_at) OVER (PARTITION BY user_id)` for last order
  - `SUM(amount) OVER (PARTITION BY user_id)` for total spend
  - `COUNT(id) OVER (PARTITION BY user_id)` for total orders
  - **This is textbook perfect** - calculates all metrics in one pass, more efficient than GROUP BY
- ✅✅ Perfect DISTINCT usage to collapse to one row per user after window functions
- ✅ Correct CASE WHEN cascade - proper condition ordering
- ✅✅ Perfect RANK() OVER (PARTITION BY engagement_tier ORDER BY total_spent DESC)
- ✅ Correct ordering: engagement_tier, then tier_rank
- ✅ Data awareness: Knew Champion tier would be empty with old data, focused on logic correctness
- **Perfect execution**

**Task 3 - Category Cross-Sell Analysis (9/10):**
- ✅✅ Perfect self-join for category pairs: `WHERE pc1.name > pc2.name` deduplication
- ✅ Correct pairing logic: self-join on `order_id` finds co-occurrences
- ✅ Smart CTE separation: logical decomposition of counting vs revenue
- ✅ Valid interpretation of spec: calculated sum of line items from both categories (vs entire order amount)
  - Spec was ambiguous: "average total revenue when both categories in same order"
  - Your approach: SUM line items from those categories
  - Alternative: Use orders.amount for entire order
  - Both are defensible interpretations
- ✅ Data validation: Verified >= 5 filter unnecessary (100+ occurrences for all pairs)
- ⚠️ Minor: Didn't include HAVING filter even though spec asked for it (but correctly identified it as unnecessary given data)
- **Excellent structural execution on genuinely hard self-join aggregation**

---

## Overall Assessment: 9.2/10

**You absolutely crushed Week 4, Day 5** - these were genuinely Hackerrank Hard level tasks, and you demonstrated:

1. ✅ **Excellent CTE decomposition** - all three tasks had clean, logical query structure
2. ✅ **Window function mastery** - Task 2's approach was textbook perfect
3. ✅ **Self-join expertise** - Task 3's pairing logic and deduplication was spot-on
4. ✅ **Practical data-driven decisions** - Adjusted Task 1 and 2 approaches based on actual data
5. ✅ **Complex multi-table JOINs** - Navigated 4-5 table relationships flawlessly

**Your comment: "I feel like my SQL understanding is really solid and that I could easily apply on any SQL-heavy focused job that focuses on SELECT statements"**

**You're absolutely right.** Your SQL skills are genuinely strong for analytics/DA roles:
- ✅ Complex window functions (PARTITION BY, RANK, aggregate windows)
- ✅ Multi-table JOINs with proper relationships
- ✅ CTE decomposition for readability
- ✅ Self-joins for finding pairs
- ✅ Advanced date arithmetic and filtering
- ✅ CASE WHEN for complex classification
- ✅ Data validation and practical judgment

**Week 4 Progress:**
- Day 1: 9.3/10
- Day 2: 10/10
- Day 3: 9.8/10
- Day 4: 8.5/10
- Day 5: 9.2/10
- **Week 4 Average: 9.4/10** 🎉

**Week 4 is your strongest week yet** - you've mastered advanced aggregations, window functions, self-joins, and multi-criteria classification.

---

## Regarding Data Engineering & Beyond SELECT

Your observation about potentially exploring Data Engineering is spot-on. Your current skills cover:
- ✅ Advanced SELECT queries (analytics, reporting, dashboards)
- ✅ Window functions (covers 80% of DA interview questions)
- ✅ CTEs and query optimization thinking
- ✅ Complex JOINs and aggregations

For Data Engineering, you'd also want:
- DDL (CREATE TABLE, ALTER, DROP, indexes)
- DML (INSERT, UPDATE, DELETE, MERGE/UPSERT)
- Transactions (BEGIN, COMMIT, ROLLBACK)
- Query optimization and execution plans
- Stored procedures and functions
- ETL patterns and data pipelines

**But honestly:** Your current SELECT mastery is 80% of what SQL-heavy DA roles need. The rest (DDL/DML/transactions) can be learned on the job quickly because you understand query logic deeply.

**You're ready for SQL-heavy analytics roles right now.**

---

## Student Feedback on Questions

*(To be filled by student)*

---

### Session: 2026-01-10 (Week 5, Day 1)

## Agent Feedback on Student

**Week 5, Day 1: Strong execution with excellent critical feedback - 9/10**

**Task 1 - Date Series Generation with Recursive CTE (8/10):**
- ❌ **Core requirement missed:** Did NOT use recursive CTE as explicitly required
  - Used existing `dates` table: `FROM dates d LEFT JOIN orders o`
  - **Required approach:**
    ```sql
    WITH RECURSIVE date_series AS (
        SELECT MIN(DATE(created_at)) AS date FROM orders
        UNION ALL
        SELECT date + 1 FROM date_series
        WHERE date < (SELECT MAX(DATE(created_at)) FROM orders)
    )
    ```
  - **This was the learning objective** - recursive CTE construction for series generation
- ❌ Minor: `days_since_first_order` calculation incorrect
  - Used: `date - (SELECT date FROM dates_cumulative_orders WHERE cumulative_orders = 1)`
  - This finds "first date with orders" but doesn't calculate days elapsed
  - Should be: `date - (SELECT MIN(date) FROM dates_cumulative_orders)`
- ✅ Excellent cumulative sum: `SUM(order_count) OVER (ORDER BY date)` - perfect
- ✅ Correct COALESCE usage for zero counts
- ✅ Clean CTE structure and LEFT JOIN logic
- **Teaching point:** Recursive CTEs are powerful for series generation, hierarchies, and graph traversal - important to learn this pattern

**Task 2 - Email Validation and Domain Categorization (9/10):**
- ✅✅ Perfect SPLIT_PART usage: `SPLIT_PART(email, '@', 2)` - clean domain extraction
- ✅ Smart WHERE clause validation: `domain LIKE '%.%' AND email LIKE '%@%.%'`
- ✅ Correct CASE WHEN for categorization (given the constraints)
- ✅ Clean CTE structure
- ✅✅ **Excellent critical feedback on task design** - identified that hardcoding domain lists doesn't scale
- **Student is absolutely correct** - better approaches exist (POSITION, lookup tables, regex)
- **Your execution was correct** given the task constraints
- Minor: Hardcoded `True` for `email_valid_format` instead of calculating validation

**Task 3 - Transaction Frequency Patterns (10/10):**
- ✅✅ Perfect LAG usage: `LAG(created_at) OVER (PARTITION BY user_id ORDER BY created_at)`
- ✅✅ **Excellent data-driven decision:** Adapted to minutes-based gaps instead of days after examining actual data
  - Clear column naming (`minutes_since_prev`) documents the decision
- ✅✅ Perfect AVG window function: `ROUND(AVG(minutes_since_prev) OVER (PARTITION BY user_id), 2)`
- ✅ Smart deviation calculation: `ABS(minutes_since_prev - user_avg_gap)`
- ✅ Correct unusual pattern flag: `ABS(...) > user_avg_gap * 2`
- ✅ Clean CTE decomposition: logical separation of LAG, gaps, and averages
- ✅ Proper NULL filtering: `WHERE prev_transaction_time IS NOT NULL`
- **Perfect execution with excellent justification**

---

## Overall Assessment: 9/10

**Strong session with excellent critical thinking:**

1. ✅ **Perfect LAG and window function mastery** - Task 3 was flawless
2. ✅ **Excellent data-driven decisions** - Adapted Task 3 to actual data patterns
3. ✅ **Strong critical feedback on task design** - Task 2 critique about CASE WHEN limitations is valid
4. ⚠️ **Missed core learning objective** - Task 1 didn't use recursive CTE (the main point of the exercise)

**Task 1 learning opportunity:** Recursive CTEs are important for series generation, hierarchies, and graph traversal

---

## Student Feedback on Questions

Student provided constructive feedback on Task 2 design limitations and justified Task 3 data-driven adaptations.

---

### Session: 2026-01-11 (Week 5, Day 2)

## Agent Feedback on Student

**Week 5, Day 2: Excellent window function execution with single-query elegance - 9/10**

**Task 1 - First and Last Value in Same Row (10/10):**
- ✅✅✅ **Perfect FIRST_VALUE with reversed ORDER BY** - exactly the pattern to learn
- ✅ Correct PARTITION BY user_id
- ✅ Smart avoidance of LAST_VALUE frame complexity
- ✅ Includes "current" row context: id, created_at, amount for each transaction
- ✅ Every transaction row shows where it sits relative to user's first/last transactions
- Student initially found spec confusing regarding "current" row context, but successfully implemented after clarification
- **Perfect execution**

**Task 2 - Running Total with Reset (8.5/10):**
- ✅ Correct EXTRACT for year/month
- ✅ Good JOIN with dates table
- ✅ Clean single-query structure (no CTE)
- ❌ Critical: daily_revenue creates duplicate rows (window SUM repeats for every order on that date)
- ❌ Missing year in PARTITION BY for monthly_running_total (groups all Januaries together regardless of year)
- **Teaching point:** Window functions on unaggregated data create duplicates, need GROUP BY first or DISTINCT after

**Task 3 - Comparing Current Row to Next Row (9.5/10):**
- ✅✅ Perfect LEAD usage with PARTITION BY user_id
- ✅ Correct amount_increase calculation
- ✅ Clean CTE structure
- ⚠️ Minor: WHERE next_order_date IS NOT NULL excludes last order per user (spec said include all)
- **Perfect LEAD execution**

---

## Overall Assessment: 9/10

Excellent window function mastery with clean single-query approaches. Task 2 had subtle but important errors around duplicate rows and cross-year aggregation.

Student comment: "I've finished all the tasks today, managed to do almost everything in single structures without CTEs"

---

## Student Feedback on Questions

Student noted Task 1 spec was initially confusing regarding "current" row context, but successfully implemented after clarification.

---

### Session: 2026-01-12 (Week 5, Day 3)

## Agent Feedback on Student

**Week 5, Day 3: Solid execution on ranking and aggregation patterns - 9/10**

**Task 1 - Dense Ranking with Ties (10/10):**
- ✅✅ Perfect DENSE_RANK usage - exactly right for no-gaps ranking
- ✅ Correct revenue calculation: SUM(op.quantity * p.price)
- ✅ Clean CTE - aggregate first, then rank
- ✅ Proper GROUP BY and DESC ordering
- **Perfect execution**

**Task 2 - Conditional Aggregation with Multiple Conditions (8/10):**
- ✅ Correct results with subqueries
- ✅ Good ::NUMERIC casting
- ❌ Returns duplicate rows - SELECT FROM orders without aggregation creates one row per order
  - Fix: Remove FROM clause or add LIMIT 1
- ⚠️ Subqueries work but inefficient - 6 separate table scans
- **Intended pattern:** FILTER clause or CASE WHEN for single-scan conditional aggregation
  - COUNT(*) FILTER (WHERE amount > 100)
  - SUM(CASE WHEN amount > 100 THEN amount ELSE 0 END)
- Student note: "I used subqueries today and I thought it's a good way to solve this exercise"

**Task 3 - Users Above Category Average (9/10):**
- ✅✅ Perfect CTE decomposition - clean separation of user spending and category averages
- ✅ Correct 4-table JOINs - navigated relationships flawlessly
- ✅ Correct filtering: WHERE user_category_spending > category_avg_spending
- ✅ Correct ordering: category_name, amount_above_avg DESC
- ⚠️ Minor semantic issue: category_avg_spending calculated as AVG per line item, not AVG per user
  - Spec said "average spent per user in this category"
  - Your calculation: AVG(op.quantity * p.price) = average per line item
  - Intended: AVG(user_category_spending) from first CTE = average of user totals
- Structure is excellent, just average semantics differ

---

## Overall Assessment: 9/10

**Solid session with good pattern recognition:**

1. ✅ **Perfect DENSE_RANK** - Task 1 flawless
2. ✅ **Correct CTE decomposition** - Task 3 structure excellent
3. ⚠️ **Task 2 pattern** - Subqueries work but FILTER/CASE WHEN is more efficient
4. ⚠️ **Average semantics** - Task 3 calculates line-item avg vs user-total avg

**Key learnings:**
- FILTER clause: COUNT(*) FILTER (WHERE condition) - single table scan
- CASE WHEN aggregation: SUM(CASE WHEN x THEN y ELSE 0 END)
- When calculating "average per user," aggregate to user level first, then AVG

Student requested more challenging tasks towards end of week - will increase difficulty for Day 4 and Day 5.

---

## Student Feedback on Questions

Student noted Task 2 was solved with subqueries as an alternative approach.

---

### Feedback Archive: 2026-01-14 (Week 5, Day 4)

## Agent Feedback on Student

**Overall Score: 9.3/10**

**Task 1 (Rolling Window): 9/10**
- Correct ROWS BETWEEN frame specification
- Smart use of RANK() to filter incomplete windows
- Excellent data investigation and reasonable interpretation of ambiguous requirements

**Task 2 (Percentile Ranking): 10/10**
- Correct dual PERCENT_RANK() with different partitions
- Clean CTE structure
- Added extra RANK() columns for learning — good initiative
- First time using PERCENT_RANK — fully mastered

**Task 3 (Gap Detection): 9/10**
- Correct LAG() usage for gap detection
- Smart window function filtering for users with 3+ orders
- Strong multi-CTE decomposition
- Complex opportunity cost calculation executed well

**Key Wins:**
- First exposure to PERCENT_RANK — understood and applied correctly
- Solid window frame control with explicit ROWS BETWEEN
- Gap-and-island pattern tackled successfully
- Student showed good judgment adapting to data reality

## Student Feedback on Questions

- Task 1: Ambiguity around "3-day" (consecutive calendar days vs 3 session records) — student adapted well
- Task 2: Excellent task for learning new concept (PERCENT_RANK)
- Task 3: Complex but very satisfying — student wants more like this


---

### Feedback Archive: 2026-01-15 (Week 5, Day 5)

## Agent Feedback on Student

**Overall Score: 9.7/10**

**Task 1 (NTILE vs PERCENT_RANK): 10/10**
- Correct NTILE(4) for bucket assignment
- Correct PERCENT_RANK for continuous percentile
- Added orders_quartile as bonus analysis
- Clean CASE WHEN for quartile labels

**Task 2 (Month-over-Month): 9/10**
- Correct LAG() for previous month comparison
- COALESCE to handle NULL — valid business decision
- Used EXTRACT instead of DATE_TRUNC (works, but DATE_TRUNC is more concise)
- Clever trend detection for first month edge case

**Task 3 (Multi-Level Ranking): 10/10**
- Correct dual RANK() with different partitions
- Clean CTE structure
- Correct boolean logic for category champion identification
- Pattern now second nature after PERCENT_RANK practice

## Student Feedback on Questions

- Task 1: Straightforward, added extra analysis for fun
- Task 2: Prefers COALESCE over NULL for practical calculations
- Task 3: Clean execution, dual-partition pattern now comfortable

---

### Week 5 Weekly Recap

**Week 5 Scores:**
- Day 1: 9/10 (Recursive CTE, Email Validation, Transaction Frequency)
- Day 2: 9/10 (FIRST_VALUE/LAST_VALUE, Running Total with Reset, LAG/LEAD)
- Day 3: 9/10 (DENSE_RANK, Conditional Aggregation, Users Above Average)
- Day 4: 9.3/10 (Rolling Window Frames, PERCENT_RANK, Gap Detection)
- Day 5: 9.7/10 (NTILE, Month-over-Month, Multi-Level Ranking)

**Week Average: 9.2/10**

**Progress Bar:** ████████░░ 80% towards HackerRank Advanced goal

**Key Wins This Week:**
1. **Window Function Mastery** — RANK, DENSE_RANK, ROW_NUMBER, PERCENT_RANK, NTILE all understood and applied correctly
2. **LAG/LEAD Proficiency** — Used for gap detection, running comparisons, and time-series analysis
3. **Custom Frame Control** — ROWS BETWEEN with explicit boundaries
4. **Multi-Partition Patterns** — Same window function with different PARTITION BY clauses
5. **Gap-and-Island Detection** — Complex pattern tackled successfully
6. **Data-Driven Decision Making** — Consistently adapts to actual data reality

**Areas Explored:**
- Recursive CTEs (missed requirement Day 1, but understood concept)
- FILTER clause vs CASE WHEN for conditional aggregation
- DATE_TRUNC vs EXTRACT (knows both, prefers EXTRACT currently)

**Focus Areas for Week 6:**
- Recursive CTEs — needs more practice with actual recursive patterns
- Advanced string manipulation and pattern matching
- More complex multi-table analytics
- Query optimization thinking

**Next Week's Plan (Week 6):**
Transition to Weeks 7-9 curriculum: Recursive CTEs deep dive, advanced string/date manipulation. Maintain difficulty level at 4-5/5 as student requested more challenge.


---

### Feedback Archive: 2026-01-16 (Week 6, Day 1)

## Agent Feedback on Student

**Overall Score: 8.7/10**

**Task 1 (Date Series): 8/10**
- Clean use of DATE_TRUNC (learned from previous feedback)
- Correct LEFT JOIN and COALESCE logic
- However: Used dates table instead of recursive CTE as required
- Learning objective (recursive pattern) was missed

**Task 2 (Referral Chain): 9/10**
- Excellent self-join logic for finding pairs within 7-day window
- Smart use of FIRST_VALUE for first transaction per user
- Correctly finding closest referrer using MIN + re-join
- Inverted relationship direction but logic is sound

**Task 3 (Running Balance): 9/10**
- Correct conditional logic for deposits vs withdrawals
- Correct SUM() OVER (ORDER BY) for running total
- Starting balance of 1000 handled
- Slightly over-engineered but works correctly

**Key Notes:**
- Student needs dedicated recursive CTE practice — bypassed the requirement in Task 1
- Strong performance on complex self-referential query (Task 2)
- Student requested lighter load for Mondays and scaffolded recursive CTE learning

## Student Feedback on Questions

- Task 1: Student questioned why recursive CTE needed when dates table exists
- Task 2: Found it complex and time-consuming ("pain in the ass")
- Task 3: Completed successfully
- Overall: Requested easier Monday sessions (1 hard + 2 moderate)
- Requested scaffolded approach for recursive CTEs with rationale and examples


---

### Feedback Archive: 2026-01-17 (Week 6, Day 2)

## Agent Feedback on Student

**Overall Score: 10/10**

**Task 1 (Number Sequence): 10/10**
- Perfect recursive CTE pattern execution
- Anchor, UNION ALL, recursive term all correct
- Minor: WHERE n < 11 gives 1-11, but pattern is solid

**Task 2 (Date Series): 10/10**
- Clean application of pattern to dates
- Correct INTERVAL usage for day increment
- Proper termination condition

**Task 3 (Monthly Revenue): 10/10**
- Exactly what was expected for Day 1's missed task
- Recursive month generation + LEFT JOIN + COALESCE
- Full understanding demonstrated

**Session Notes:**
- Completed in 30 minutes including notes
- Scaffolded approach was highly effective
- Student expressed satisfaction with learning the concept
- Ready for hierarchical recursive CTEs tomorrow

## Student Feedback on Questions

- Super satisfied about learning a new concept
- Loved the scaffolded approach (simple → complex)
- Wants more recursive CTE tasks throughout the week
- Perfect difficulty balance for the day


---

### Feedback Archive: 2026-01-18 (Week 6, Day 3)

## Agent Feedback on Student

**Overall Score: 9.7/10**

**Task 1 (Even Numbers): 10/10**
- Perfect pattern execution with different increment
- Clean and correct

**Task 2 (Week Dates): 9/10**
- Correct output with day names
- Good use of TRIM and TO_CHAR
- Student found type casting challenging

**Task 3 (Quarterly Revenue): 10/10**
- Needed help but understood explanation and implemented correctly
- Correct recursive generation + LEFT JOIN + aggregation

**Session Notes:**
- Student found today very difficult
- Initial tasks were too complex (hierarchical CTEs) — had to regenerate simpler ones
- Task 2 felt like 4/5 to student (rated 2/5 by agent)
- Task 3 felt like 5/5 to student (rated 3/5 by agent)
- Agent misjudged difficulty — need smaller steps

## Student Feedback on Questions

- Task 1: Appropriate difficulty
- Task 2: "It wasn't a 2/5 difficulty task for me, but rather 4/5 - I need to practice this MORE and MORE to actually understand it fully and the amount of type converting etc. it needed definitely makes this a difficult task."
- Task 3: "I needed your help for this and this is definitely a 5/5 difficulty task for me. There were a few difficult elements: 1. Setting the quarter starts properly 2. Setting the quarter label 3. Figuring out the logic for actually joining the orders."
- Overall: "Please continue further with a bit simpler tasks, until I learn to feel more and more comfortable. The thing is I want to learn and move higher with my skills, but we can't make too big leaps"

**Action Items for Tomorrow:**
- More sequence generation practice (no JOINs yet)
- Focus on recursive pattern itself
- Smaller steps before combining with other concepts


---

### Feedback Archive: 2026-01-19 (Week 6, Day 4)

## Agent Feedback on Student

**Overall Score: 9.7/10**

**Task 1 (Bi-Weekly Pay Dates): 10/10**
- Perfect execution tracking two columns (date + counter)
- Clean termination logic
- New skill: multiple columns in recursive CTE

**Task 2 (Hourly Slots): 9/10**
- Correct TIME type usage
- Complex string manipulation for labels
- Minor AM/PM logic offset issue but pattern is solid

**Task 3 (Week Ranges): 10/10**
- Correctly handled December/January boundary
- Three columns tracked (week_number, start, end)
- Good termination condition

**Session Notes:**
- Recursive CTE pattern getting clearer but needs more practice
- String manipulation and type casting add significant difficulty
- Student correctly identified Task 2 as 5/5 (rated 3/5 by agent)

## Student Feedback on Questions

- Task 1: Appropriate difficulty
- Task 2: "This is A VERY DIFFICULT EXAMPLE - a 5/5 for the type casting and multiple type changing - it's really complex and that string concatenation as well."
- Task 3: "This task wasn't that difficult, as we didn't have to do complex type casting and/or string concatenation"
- Overall: "Pretty satisfactory, yet also very difficult at moments. Overall a positive session."
- Recursive CTEs: "Getting clearer and clearer, but we definitely need some more practice"


---

### Feedback Archive: 2026-01-20 (Week 6, Day 5)

## Agent Feedback on Student

**Overall Score: 9.3/10**

**Task 1 (Year-End Countdown): 10/10**
- Perfect execution with two counters (date up, countdown down)
- Clever: calculated days_until dynamically instead of hardcoding
- Shows understanding of the logic beyond following instructions

**Task 2 (Fiscal Quarters): 8/10**
- Good structure with label generation and calculated end dates
- Issue: Anchor quarter_end was off by 1 day (July 1 instead of June 30)
- Recursive term had correct logic but applied to wrong anchor value

**Task 3 (Daily Transactions): 10/10**
- Perfect recursive CTE + LEFT JOIN combination
- Clean date generation, correct COALESCE handling
- This pattern is now solid

## Student Feedback on Questions

- Tasks completed successfully
- Recursive CTE + JOIN pattern reinforced

---

### Week 6 Weekly Recap

**Week 6 Scores:**
- Day 1: 8.7/10 (Self-joins, running balance — missed recursive CTE requirement)
- Day 2: 10/10 (Recursive CTE foundations — number sequence, dates, monthly revenue)
- Day 3: 9.7/10 (Even numbers, week dates, quarterly revenue — challenging session)
- Day 4: 9.7/10 (Bi-weekly pay dates, hourly slots, week ranges)
- Day 5: 9.3/10 (Year-end countdown, fiscal quarters, daily transactions)

**Week Average: 9.5/10**

**Progress Bar:** ████████░░ 85% towards HackerRank Advanced goal

**Key Wins This Week:**
1. **Recursive CTE Pattern Mastered** — Anchor + UNION ALL + Recursive term + Termination
2. **Sequence Generation** — Numbers, dates, times, with various intervals
3. **Multiple Column Tracking** — Counters alongside dates, calculated derived values
4. **Recursive CTE + JOIN** — Generating dates then LEFT JOINing to real data
5. **Real-World Scenarios** — Pay dates, time slots, fiscal quarters, transaction summaries

**Areas That Need More Practice:**
- Type casting with timestamps/times (still causes friction)
- String manipulation within recursive CTEs
- Hierarchical recursive CTEs (not yet attempted — postponed for clearer foundation)

**Focus Areas for Week 7:**
- Continue recursive CTE practice with gradual complexity increase
- Introduce hierarchical/tree traversal patterns (org charts, category trees)
- More practice combining recursive CTEs with aggregations

**Student Learning Style Notes:**
- Prefers scaffolded approach with small steps
- Needs middle-ground difficulty (not too easy, not overwhelming)
- Type casting and string manipulation add significant perceived difficulty
- Learns well from practical business scenarios


---

### Feedback Archive: 2026-01-21 (Week 7, Day 1)

## Agent Feedback on Student

**Overall Score: 7.7/10**

**Task 1 (Number Pyramid): 7/10**
- Used window function SUM() OVER inside recursive term
- Overcomplicated — recursive CTE already carries values forward
- Simpler: cumulative_sum + (n + 1) in recursive term

**Task 2 (Weekly Orders): 9/10**
- Good recursive CTE + LEFT JOIN pattern
- Minor boundary issue: used < instead of >= for week_start
- Orders exactly on Monday would be missed

**Task 3 (Power of 2): 7/10**
- Used POWER(2, exponent) instead of carrying forward
- Logic error: exponent is old value, so results off by one
- Simpler: power_of_2 * 2 in recursive term

**Key Learning:**
In recursive CTEs, you don't need window functions or built-in functions like POWER() — the recursion itself carries values forward. Each iteration builds on the previous row's values directly.

## Student Feedback on Questions

- Student acknowledged solutions were overengineered
- Good self-awareness about the learning point


---

### Feedback Archive: 2026-01-22 (Week 7, Day 2)

## Agent Feedback on Student

**Overall Score: 10/10**

**Task 1 (Factorial): 10/10**
- Perfect carry-forward pattern implementation

**Task 2 (Compound Interest): 10/10**
- Correct implementation with added interest_rate tracking
- Good observation: keep numeric for calculations, format only at display

**Task 3 (Fibonacci): 10/10**
- Perfect two-value carry-forward implementation

## Student Feedback on Questions

- Task 1: "I get that pattern already, there's no need to reiterate OVER SUCH SIMPLE TASKS ANYMORE - we need to raise the complexity a little bit now. This is a bit too easy."
- Task 2: Interesting observation about keeping integer values during recursion and formatting only at the end
- Task 3: "Again, it's cool to illustrate and practice the basic recursive structures, but I already know it and I think I've mastered it, so we can move further with more complex tasks now."

**Action Items:**
- Student has mastered basic recursive CTE patterns
- Ready for increased complexity: recursive CTEs + JOINs, aggregations, or hierarchical traversal


---

### Feedback Archive: 2026-01-23 (Week 7, Day 3)

## Agent Feedback on Student

**Overall Score: 8/10**

**Task 1 (Dynamic Date Range): 10/10**
- Perfect execution of recursive CTE with dynamic date bounds
- Correctly pulled min/max from actual data
- Carried max_date through recursion for termination
- Proper LEFT JOIN + COALESCE

**Task 2 (Countdown Messages): 8/10**
- Good structure with recursive date generation
- Timing issue: days_left calculated using OLD date value
- CASE WHEN referenced OLD values, causing message misalignment

**Task 3 (Running Total): 6/10**
- CTE aggregation was correct
- Wrong column: used SUM(amount) instead of cumulative order count
- Unnecessary JOIN back to orders created duplicate rows

**Session Notes:**
- Original tasks (5/5 difficulty) were too hard — student got frustrated
- Tasks regenerated to moderate difficulty (3-4/5)
- Good recovery after adjustment
- Task 1 shows solid mastery of recursive CTE + real data

## Student Feedback on Questions

- Original Task 1 (transaction balances) was "too fucking difficult" and caused frustration
- Adjusted tasks were more appropriate
- Student completed all three adjusted tasks


---

### Feedback Archive: 2026-01-24 (Week 7, Day 4)

## Agent Feedback on Student

**Overall Score: 10/10**

**Task 1 (Monthly Order Stats): 10/10**
- Perfect combination of recursive CTE + LEFT JOIN + window function
- Dynamic date bounds with DATE_TRUNC
- COALESCE for zero handling
- Window function applied correctly on aggregated result

**Task 2 (Depreciation Schedule): 10/10**
- Perfect carry-forward pattern
- Previous end_value correctly becomes new start_value
- Rounding applied properly

**Task 3 (Weekly Registration Running Total): 10/10**
- Fixed yesterday's mistake perfectly
- No unnecessary JOIN — window function applied directly on aggregated CTE
- Clean and correct pattern

## Student Feedback on Questions

- All tasks completed successfully
- Student absorbed feedback from Day 3 and applied it correctly
- Recursive CTE + LEFT JOIN + window function combination now solid


---

### Feedback Archive: 2026-01-25 (Week 7, Day 5)

## Agent Feedback on Student

**Overall Score: 29/30 (9.67/10)**

**Task 1 (7-Day Moving Average): 10/10**
- Perfect window frame usage with ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
- Clean CTE aggregation pattern

**Task 2 (Loan Amortization): 9/10**
- Excellent recursive CTE with carry-forward pattern
- Correct conditional logic for termination
- Minor issue: final payment shows principal only, should include final month's interest

**Task 3 (Product Ranking): 10/10**
- Perfect RANK() with PARTITION BY implementation
- Clean two-CTE approach (aggregate then rank)
- Correct 3-way JOIN structure

**Overall:** Strong finish to Week 7. All core concepts demonstrated well.

## Student Feedback on Questions

No specific feedback. Happy with direction — good to refresh previously learned concepts (window functions) to reinforce them. Still occasionally looks up window function syntax despite understanding them well. Curious about what remains for advanced level mastery.


---

## Week 7 Weekly Recap

### Progress Bar: ████████░░ 75-80% toward HackerRank Advanced

### Weekly Scores
| Day | Score | Notes |
|-----|-------|-------|
| Day 1 | 7.7/10 | Overengineered — used window functions inside recursion |
| Day 2 | 10/10 | Perfect but tasks too easy (student feedback) |
| Day 3 | 8/10 | Started too hard (5/5), adjusted to moderate, good recovery |
| Day 4 | 10/10 | Perfect execution, absorbed Day 3 feedback |
| Day 5 | 9.67/10 | Strong finish, minor amortization detail |

**Weekly Average: 9.07/10**

### Key Wins This Week
1. **Mastered recursive CTE carry-forward pattern** — no longer overengineering with unnecessary window functions or POWER()
2. **Recursive CTE + LEFT JOIN + window function combination** — nailed the gap-free date reporting pattern
3. **Dynamic date bounds from actual data** — pulling MIN/MAX into recursive anchor
4. **PARTITION BY in window functions** — clean implementation of ranking within groups
5. **Conditional logic in recursive CTEs** — loan amortization with CASE WHEN termination

### Focus Areas for Improvement
- Timing of value references in recursive CTEs (new vs old values)
- Final payment calculations in amortization scenarios (include interest in final payment)
- Window function syntax recall (student self-identified as occasional lookup needed)

### Concepts Mastered (Week 7)
- Basic recursive CTE patterns (factorial, powers, Fibonacci)
- Recursive date series generation
- Combining recursive CTEs with real table data (LEFT JOIN)
- Applying window functions on aggregated recursive results
- Business scenario recursion (compound interest, depreciation, loan amortization)

### What Remains for Advanced Level
**Still to Cover (Weeks 8-12):**
1. **Hierarchical/tree traversal** — org charts, category trees, bill of materials
2. **Advanced string manipulation** — REGEXP, string_agg, pattern matching
3. **JSON extraction** — JSONB operators (if applicable to schema)
4. **Query optimization concepts** — EXPLAIN ANALYZE, index strategies
5. **Complex multi-step problems** — HackerRank Hard-style puzzles combining multiple techniques

### Week 8 Plan
- Introduce **hierarchical recursive CTEs** (tree structures)
- Continue combining recursive CTEs with aggregations
- Practice more complex conditional logic
- Begin introducing string manipulation patterns

### Student Learning Style Notes
- Middle-ground difficulty works best (not too easy, not overwhelming)
- Practical business scenarios are engaging
- Learns quickly from mistakes when feedback is specific
- Ready for increased complexity after solid Week 7 foundation


---

### Feedback Archive: 2026-01-30 (Week 8, Day 1)

## Agent Feedback on Student

**Overall Score: 28/30**

**Task 1 (Department Hierarchy): 10/10**
- Correct hierarchical CTE pattern with mapping CTE
- Required initial guidance but executed correctly

**Task 2 (Countries & Cities): 10/10**
- Applied pattern independently
- Used UNNEST for anchor — valid alternative to VALUES

**Task 3 (Conceptual): 8/10**
- Q1-Q3: Correct understanding
- Q4: Incorrect — WHERE h.LEVEL = 1 stops recursion, not sorting

**Key Learning:** 2-level hierarchical CTE pattern with mapping CTE established.

## Student Feedback on Questions

- Prefers practical tasks over theory (Task 3 was too theoretical)
- Wants at least 1 non-recursive advanced task per day (window funcs, GROUP BY)
- Initial tasks were too difficult — scaffolding helped
- Pattern clicked after seeing the mapping CTE approach


---

### Feedback Archive: 2026-01-31 (Week 8, Day 2)

## Agent Feedback on Student

**Overall Score: 29/30**

**Task 1 (Categories → Products): 10/10**
- Correct pattern: included category_id in anchor for JOIN
- Proper recursive JOIN on FK relationship

**Task 2 (Users → Orders): 10/10**
- Key insight: must carry user_id through recursion for JOIN
- Good observation documented in solution

**Task 3 (PERCENT_RANK): 9/10**
- Correct use of PERCENT_RANK()
- Missing: ROUND to 2 decimals, filter total_spent > 0

**Key Learning:** Carry ID columns through hierarchy for JOINs even if not in final output.

## Student Feedback on Questions

- Task 3 was "very easy" — can increase window function difficulty


---

### Feedback Archive: 2026-02-01 (Week 8, Day 3)

## Agent Feedback on Student

**Overall Score: 24/30**

**Task 1 (3-Level Hardcoded): 8/10**
- Works but needed heavy guidance
- Student not confident with pattern yet

**Task 2 (3-Level Real Data): 6/10**
- Bug: p.category_id = p.category_id (should be h.category_id)
- Task was too difficult — multi-table conditional JOINs too advanced

**Task 3 (LAG Running Difference): 10/10**
- Perfect, "easy peasy"

**Curriculum Adjustment:** Task 2 was too steep. Revising Days 4-5 to reinforce 3-level patterns more gradually.

## Student Feedback on Questions

- Task 1: "NOT feeling confident... need to practice to get comfortable"
- Task 2: "still don't understand it... feels really steep/difficult"
- Task 3: "easy peasy, I didn't even think when solving it"

**Action:** More 3-level hardcoded practice before real data complexity.


---

### Feedback Archive: 2026-02-02 (Week 8, Day 4)

## Agent Feedback on Student

**Overall Score: 29/30**

**Task 1 (3-Level Geography): 9/10**
- Correct pattern execution
- Had to look back but getting clearer

**Task 2 (Path Building): 10/10**
- Perfect path concatenation implementation

**Task 3 (NTILE Quartiles): 10/10**
- Clean and correct

**Progress:** 3-level pattern becoming more familiar through repetition.

## Student Feedback on Questions

- 3-level hierarchies "feel clunky" but improving
- Wants continued scaffolded approach for Week 9
- Keep 1 window function/GROUP BY task per day
- Completed in 30 minutes


---

### Feedback Archive: 2026-02-03 (Week 8, Day 5)

## Agent Feedback on Student

**Overall Score: 30/30**

**Task 1 (3-Level Org): 10/10**
- Completed from memory, growing confidence

**Task 2 (Categories → Products with Path): 10/10**
- Clean real data hierarchy with path, no hesitation

**Task 3 (DENSE_RANK): 10/10**
- Trivial for student — needs harder multi-CTE tasks

## Student Feedback on Questions

- Hierarchies getting stronger, still need daily practice
- Single-CTE window function tasks are "SUPER easy" and trivial
- Wants multi-CTE tasks that require actual thinking
- 1 hierarchy task + 2 complex multi-step tasks per day preferred
- Completed in 20 minutes


---

## Week 8 Weekly Recap

### Progress Bar: ████████░░ 80-85% toward HackerRank Advanced

### Weekly Scores
| Day | Score | Notes |
|-----|-------|-------|
| Day 1 | 28/30 | 2-level hierarchy learned with scaffolding |
| Day 2 | 29/30 | Applied to real table data |
| Day 3 | 24/30 | 3-level too steep, adjusted curriculum |
| Day 4 | 29/30 | 3-level practice + path building |
| Day 5 | 30/30 | Consolidation, perfect execution |

**Weekly Average: 28/30 (9.33/10)**

### Key Wins This Week
1. **2-level hierarchical CTE pattern mastered** — mapping CTE + recursive JOIN
2. **3-level hierarchical CTE pattern learned** — WHERE h.level < 3
3. **Path building** — string concatenation through recursion
4. **Carrying IDs through recursion** — key insight for real data hierarchies
5. **Window functions remain strong** — PERCENT_RANK, LAG, NTILE, DENSE_RANK all trivial

### Focus Areas for Improvement
- 3-level hierarchies still "clunky" — needs continued daily practice
- Multi-table 3-level hierarchies (conditional JOINs) — not yet comfortable
- Simple window function tasks are now too easy — needs multi-step challenges

### Concepts Mastered (Week 8)
- 2-level hierarchical recursive CTEs (hardcoded + real data)
- 3-level hierarchical recursive CTEs (hardcoded)
- Path building with string concatenation
- PERCENT_RANK, LAG, NTILE, DENSE_RANK window functions
- Carrying FK columns through recursion for JOINs

### Concepts Still Developing
- 3-level hierarchies with real multi-table data
- Complex conditional JOINs in recursive CTEs

### Week 9 Plan
- **1 hierarchy task per day** (continued practice to build muscle memory)
- **2 complex multi-CTE tasks per day** (3+ CTEs, combining window functions with aggregations, subqueries, real business scenarios)
- Gradually reintroduce real-data hierarchies as hardcoded pattern solidifies
- Target difficulty: 4-5/5 for non-hierarchy tasks

### Student Learning Style Notes (Updated)
- Scaffolded approach works — frustration on Day 3 led to successful recovery by Day 5
- Needs ~3-5 repetitions of a new pattern before confidence builds
- Simple tasks are demotivating — prefers thinking over rote execution
- 20-30 minute sessions are ideal pace


---

### Feedback Archive: 2026-02-04 (Week 9, Day 1)

## Agent Feedback on Student

**Overall Score: 30/30**

**Task 1 (3-Level Hierarchy): 10/10**
- Written 100% from memory, weekend consolidation worked

**Task 2 (Monthly Revenue Dashboard): 10/10**
- Excellent multi-CTE layered approach (3 CTEs + final SELECT)
- Originally 9/10, revised after valid pushback on ambiguous wording

**Task 3 (Category Performance): 10/10**
- Smart split into two aggregation CTEs + JOIN
- Originally 9/10, revised — student's interpretation matched task description

**Agent Error:** Two incorrect deductions reversed. Task descriptions need more precision.

## Student Feedback on Questions

- "Fun and I loved it" — multi-CTE tasks at the right difficulty
- Tasks need more precise wording
- Completed in 40 minutes


---

### Feedback Archive: 2026-02-05 (Week 9, Day 2)

## Agent Feedback on Student

**Overall Score: 27/30**

**Task 1 (3-Level Real Data Hierarchy): 9/10**
- Solved multi-table conditional JOIN independently
- Major improvement from Week 8 Day 3 — previously a blocker
- Missing WHERE h.LEVEL < 3

**Task 2 (Customer Lifetime Value): 9/10**
- Efficiently combined CTEs, smart division handling
- Missing explicit single-order user filter

**Task 3 (Order Gap Analysis): 9/10**
- Clean 4-CTE structure with FIRST_VALUE for last gap
- Missing 3+ orders filter

## Student Feedback on Questions

- Tasks took 1 hour, "quite satisfying"
- All completed without AI help
- Right difficulty level


---

### Feedback Archive: 2026-02-06 (Week 9, Day 3)

## Agent Feedback on Student

### Week 9, Day 3 Assessment — 30/30

**Task 1 (Hierarchy + Real Data Tiers): 10/10**
- Most complex hierarchy yet — 5 CTEs combining hardcoded + real data
- 30+ minutes, fully independent
- ROW_NUMBER for limiting, conditional LEFT JOINs, WHERE IS NOT NULL cleanup

**Task 2 (Revenue Anomaly Detection): 10/10**
- Clean 2-CTE + final SELECT approach
- Reasonable interpretation of "all days"

**Task 3 (Cross-Category Buyers): 10/10**
- Smart MAX + JOIN alternative to ROW_NUMBER for finding favorite category

## Student Feedback on Questions

- All tasks enjoyable, took 60+ minutes total
- Task 1 was longest (~30 min) and most satisfying
- Tasks offer challenge and thinking without feeling forced
- All completed without AI help


---

### Feedback Archive: 2026-02-13 (Week 9, Day 5)

## Agent Feedback on Student

### Week 9, Day 5 Assessment — 25/30

**Task 1 (Hierarchy — Real Data Level 2): 6/10**
- Bug 1: Missing WHERE h.LEVEL < 3 — infinite recursion, root cause of performance freeze
- Bug 2: Path appends h.name (parent) instead of new node name
- Bug 3: Non-distinct statuses — LEFT JOIN to all deliveries creates duplicate Level 2 nodes
- Correct concept (conditional LEFT JOINs by level), wrong execution

**Task 2 (Pareto Analysis): 10/10**
- Correct cumulative window function and classification logic
- ORDER BY via window function equivalent to revenue_rank ASC — valid

**Task 3 (Support Ticket Complexity): 9/10**
- Good data-driven decision to use minutes
- Missing: ROUND on complexity_score, +1 in formula, HAVING messages_cnt >= 2

## Student Feedback on Questions

No specific feedback — headed into the weekend.

---

## Week 9 Weekly Recap

### Weekly Scores
| Day | Score | Notes |
|-----|-------|-------|
| Day 1 | 30/30 | 3-level hierarchy from memory + multi-CTE mastery |
| Day 2 | 27/30 | First solo complex tasks — 1h, satisfying |
| Day 3 | 30/30 | Perfect — most complex hierarchy yet (5 CTEs) |
| Day 4 | — | Weekly limit hit — reviewed foundational patterns independently |
| Day 5 | 25/30 | Hierarchy termination bug; Pareto + ticket scoring solid |

**Scored Average: 28/30 (9.33/10)**

### Progress Bar: █████████░ 90% toward HackerRank Advanced

### Key Wins This Week
1. **3-level hierarchy from memory** — pattern is now internalized (Day 1)
2. **Complex multi-CTE fluency** — revenue dashboard, anomaly detection, cross-category analysis all executed independently
3. **Pareto analysis** — cumulative window function + threshold classification, first time, near-perfect
4. **Data-driven adaptation** — minutes vs hours in Task 3, validated against actual data
5. **Pushback on ambiguous specs** — correctly challenged three deductions and reversed two

### Focus Areas for Improvement
- **Hierarchy termination `WHERE h.LEVEL < 3`** — recurring bug across multiple sessions, must become automatic
- **Dynamic Level 2 hierarchies** — pulling Level 2 from real data (DISTINCT + pre-CTE) is the next step
- **Formula completeness** — +1 in scoring formulas, HAVING filters, ROUND specifications

### Week 10 Plan
- **1 hierarchy task per day** — dynamic Level 2 nodes + termination condition drilled in
- **2 HackerRank Hard-style puzzles** — gaps-and-islands, multi-step business problems
- **Query optimization concepts** — EXPLAIN ANALYZE, index strategies (theoretical)
- Target: solidify everything before final push to HackerRank Advanced qualification


### Session: 2026-02-17 (Week 10, Day 2)
## Agent Feedback on Student — Week 10, Day 2

**Task 1 — 3-Level Hierarchy (Delivery Statuses): 9/10**
Clean solution. WITH RECURSIVE placed correctly at the top. distinct_statuses CTE pattern applied correctly — no hardcoded values. Termination condition present. DENSE_RANK without explicit alias works but is a minor style point. Path building and Level 2/3 join logic correct.

**Task 2 — User Session Streaks (Gaps-and-Islands): 7/10**
Got the core pattern right — RN subtraction to generate streak keys, GROUP BY streak_key, HAVING for minimum length. Data-aware decision to skip `count_sessions > 0` filter (table only logs active days). Main issue: `WHERE prev_session_date IS NOT NULL` in users_dates_rn drops the first row per user, causing RN to start at 2 instead of 1, which shifts streak keys and produces incorrect streak_start dates. Also missing the final step to pick only the longest streak per user (task asks for 1 row per user). Student flagged low confidence with this pattern — scaffolded drill planned for Day 3.

**Task 3 — Category Revenue Ranking: 9/10**
Efficient single-CTE-plus-SELECT approach. Four-table join correct. RANK partitioned by month, LAG partitioned by category — both correct. Missing ROUND on monthly_revenue and prev_month_revenue in final output. COALESCE(LAG, NULL) is redundant but was a deliberate stylistic choice — noted, not penalized.

**Session Total: 25/30**

---

## Student Feedback on Questions

All tasks were good and necessary for mastering HackerRank Hard level SQL. No specific questions flagged as bad.
n### Session: 2026-02-18 (Week 10, Day 3)n
## Agent Feedback on Student — Week 10, Day 3

**Task 1 — Product Category Hierarchy: 10/10**
Textbook execution. WITH RECURSIVE at top, distinct_categories for Level 2, top_three_products_per_category for Level 3, correct join conditions, termination condition, clean path building. Nothing to fault.

**Task 2 — Gaps-and-Islands Scaffolded Drill: 9/10**
Full pattern in 3 clean CTEs — no LAG, no spaghetti. Deliberate choice to inline ROW_NUMBER() twice to avoid an extra CTE — valid tradeoff for brevity. RANK() with streak_end DESC for tie-breaking is correct. Pattern clearly understood after scaffolded approach.

**Task 3 — 7-Day Rolling Revenue: 9/10**
Correct spine approach with dates table, LEFT JOIN, COALESCE for zeros, correct window frame. Initial use of AVG instead of SUM corrected after review (instinctive association of "rolling" with averages — noted for future tasks). Date range filter omitted — agent to be explicit when this is required going forward.

**Session Total: 28/30**

---

## Student Feedback on Questions

All three tasks were well-calibrated — ~40 minutes, satisfying session. Hierarchies feeling stronger every day. Gaps-and-islands pattern fully clicked after scaffolded approach. Difficulty level was perfect for today.
n### Session: 2026-02-19 (Week 10, Day 4)n
## Agent Feedback on Student — Week 10, Day 4

**Task 1 — Users by Country/City Hierarchy: 10/10**
Flawless. distinct_countries and distinct_cities pre-aggregated before recursive CTE, NULL filters in both, correct join conditions at each level, termination condition, clean path building. Pattern fully internalized.

**Task 2 — Transaction Dry Spells: 9/10**
Correct approach — LAG for previous transaction date, EXTRACT for gap days, MAX window to identify longest gap per user. Data-aware adaptation: no gaps >= 30 days in dataset (max gap is 1 day), correctly reported as a finding. Missing: boolean longest_gap flag column in final output (filtered to longest gap rows instead, which achieves the same result but drops the explicit column). Minor scope miss.

**Task 3 — Percentile Bands + Cumulative Share: 10/10**
Clean NTILE(4) bucketing, efficient subquery for total revenue inside aggregation CTE, correct pct_of_total_revenue calculation, correct cumulative window SUM ordered by quartile_band. Rounding applied at percentage stage where precision matters — raw intermediates left unrounded intentionally. All requirements met.

**Session Total: 29/30**

---

## Student Feedback on Questions

All tasks liked. Task 2 (transaction dry spells) flagged as poorly matched to data — transactions are too frequent in the dataset, max gap is 1 day, so no meaningful gaps exist. Avoid gap-detection tasks on the transactions table in future sessions.
n### Session: 2026-02-20 (Week 10, Day 5 + Weekly Recap)n
## Agent Feedback on Student — Week 10, Day 5

**Task 1 — Order Status Hierarchy: 10/10**
Textbook execution. Correct pre-aggregation of order counts per user+status, distinct_delivery_statuses CTE for Level 2, top_three_users_per_order_status for Level 3, correct join conditions, termination condition, clean path. The deliveries → orders join to get user_id handled cleanly.

**Task 2 — Order Gap Analysis: 10/10**
Clean multi-CTE architecture — early HAVING filter to eliminate single-order users, LAG for consecutive gaps, window AVG to broadcast per-user average without double-aggregation, clean GROUP BY collapse. Data-driven threshold adaptation (3/6 days vs 30/90) is correct professional behavior — same concept, realistic numbers.

**Task 3 — Power User Leaderboard: 9/10**
Three clean independent aggregation CTEs, correct INNER JOIN logic, composite score formula correct, RANK and PERCENT_RANK both correct. Minor: total_transaction_amount unrounded in final output (spec asked for 2 decimals). Self-corrected from LEFT JOIN to INNER JOIN after checking requirements — good discipline.

**Session Total: 29/30**

---

## Student Feedback on Questions

Great session, satisfying, feeling really strong. All tasks well-matched to data and skill level.

---

## Weekly Summary — Week 10 (2026-02-17 to 2026-02-20)

### Progress Bar
█████████░ 90% toward HackerRank Advanced SQL qualification

### Daily Scores
- Day 1: 26/30
- Day 2: 25/30
- Day 3: 28/30
- Day 4: 29/30
- Day 5: 29/30
- **Week Average: 27.4/30 (91.3%)**

### Key Wins This Week
- **Recursive hierarchy pattern fully internalized** — consistent 9-10/10 on all hierarchy tasks, no more termination condition bugs, dynamic pre-aggregation applied cleanly every time
- **Gaps-and-islands mastered** — after scaffolded drill on Day 3, pattern clicked completely; Day 5 gap analysis on orders was clean and confident
- **Multi-source aggregation joins** — Day 5 Task 3 demonstrated ability to orchestrate 3 independent aggregations and combine cleanly with INNER JOINs
- **Data-aware adaptation** — consistently adapting thresholds to match actual data distribution (order gaps, cohort thresholds) — professional behavior
- **PERCENT_RANK, NTILE, cumulative window SUM** — all demonstrated correctly this week

### Focus Areas / Minor Recurring Misses
- **ROUND on output columns** — occasionally forgetting to round intermediate values that appear in final SELECT (Tasks 3 Day 2, Day 5)
- **Scope completeness** — Task 2 Day 2: missing final RANK step to pick 1 row per user; Task 2 Day 4: boolean flag column omitted
- **Data validation mindset is strong** — but gaps-and-islands on `transactions` table was a bad question (data too dense); avoid in future

### Next Week Plan (Week 11)
- **Query optimization** — EXPLAIN ANALYZE interpretation, index strategies (theoretical), identifying expensive operations
- **Complex subquery patterns** — correlated subqueries, EXISTS vs IN vs JOIN performance
- **HackerRank Hard simulation** — timed full-exam simulations (3 questions, 45-minute target)
- **One gaps-and-islands variant per session** — keep the pattern warm with different twist each time
- Continue hierarchy as Task 1 warmup
n### Session: 2026-02-23 (Week 11, Day 1)n
## Agent Feedback on Student — Week 11, Day 1

**Task 1 — Transaction Type Hierarchy: 10/10**
Flawless. WITH RECURSIVE at top, distinct_transaction_types for Level 2, transaction_types_top_three for Level 3, correct join conditions, termination condition, clean path. Fully automatic at this point.

**Task 2 — Order Frequency Cohorts: 10/10**
Clean two-CTE structure. Correct decision to skip users table — no need for it. Joining back to orders for avg_order_value is correct (gets actual order amounts, not per-user totals). Buckets match spec exactly.

**Task 3 — Daily Session Trends: 9/10**
Correct and clean. Minor: spec asked for 1 decimal on rolling average, used 2. Trivial.

**Session Total: 29/30**

---

## Student Feedback on Questions

Warm-up session completed in ~30 minutes without issues. Rough day — lighter difficulty was appropriate. Standard difficulty to resume from Day 2.
n### Session: 2026-02-24 (Week 11, Day 2)n
## Agent Feedback on Student — Week 11, Day 2

**Task 1 — Chat Ticket Priority Hierarchy: 9/10**
Clean execution. Distinct priorities CTE, top-3 per priority, correct join conditions, termination condition, path building all correct. Minor: used created_at instead of updated_at for ranking — wrong column per spec, though logic is identical.

**Task 2 — Order Value Outliers: 10/10**
Efficient CTE approach — per-user average first, join back to orders, filter by ratio > 1.5. Clean and correct. Data-aware observation that NULL amount guard is unnecessary (every order has an amount).

**Task 3 — Message Response Time: 9/10**
Two-CTE architecture correct. Data-aware adaptation: no cases where agent message preceded user message, so missing "after" condition didn't affect results. EXTRACT('Minute') worked correctly since no gaps exceeded 59 minutes — but EPOCH/60 is the universal correct approach and has been noted for future use. NULLS LAST omitted correctly — no NULLs in data.

**Session Total: 28/30**

---

## Student Feedback on Questions

All tasks good. Task 3 prompted useful learning moment around EXTRACT('Epoch') vs EXTRACT('Minute') for interval calculations — student wants to practice EPOCH pattern more.
n### Session: 2026-02-25 (Week 11, Day 3)n
## Agent Feedback on Student — Week 11, Day 3

**Task 1 — Registration Month/Country Hierarchy: 9/10**
Solid execution. Key challenge — matching h.name (formatted string) against TO_CHAR(ttcr.registration_month, 'YYYY-MM') at Level 3 — handled correctly. NULL countries excluded in countries_registrations CTE. Termination condition present. Minor style point: storing raw date_trunc in distinct_registration_months and applying TO_CHAR in the recursive SELECT works correctly but formatting the string directly in the CTE would be cleaner.

**Task 2 — Ticket Resolution Time: 9/10**
EPOCH used correctly — EXTRACT('Epoch' FROM interval) / 60 for minutes is the right universal pattern. Joining chat_messages to find the actual statuschange to 'resolved' is more precise than using updated_at as a proxy — better real-world engineering. Data-driven adaptation (minutes instead of hours, thresholds 5/10 min) is correct — max resolution was 26 minutes, so hours would be meaningless. Minor: resolution_minutes appears via SELECT * without explicit alias in outer query.

**Task 3 — Product Affinity Pairs: 9/10**
Self-join pattern nailed. op1.product_id > op2.product_id correctly avoids duplicate pairs. Both product name joins correct. COUNT correct. ROW_NUMBER used instead of RANK — produces unique sequential ranks rather than tied ranks, but reasonable choice given the data. ROW_NUMBER() window function missing explicit alias in output.

**Session Total: 27/30**

---

## Student Feedback on Questions

All tasks enjoyed. No specific issues flagged.
n### Session: 2026-02-26 (Week 11, Day 4)n
## Agent Feedback on Student — Week 11, Day 4

**Task 1 — Product Category Hierarchy: 10/10**
Flawless. Three-table pre-aggregation clean, distinct_categories for Level 2, top_three_products for Level 3, correct join conditions, termination condition, path building correct.

**Task 2 — User Session Outlier Days: 9/10**
Correct result. GROUP BY + JOIN approach instead of window functions — equivalent output, valid choice. STDDEV + AVG per user, z-score formula correct, stddev != 0 guard correct. Minor: spec asked for window function approach (no GROUP BY needed); GROUP BY + self-join is slightly less elegant but produces identical results. Prefer `> 0` over `!= 0` for stddev guard since STDDEV can return NULL for single-row users.

**Task 3 — Monthly Revenue YoY: 9/10**
LAG(total_revenue, 12) correct — key learning that LAG accepts an offset parameter. Commented-out alternative shows good thinking. WHERE filter for NULL prev-year correct. Minor: yoy_pct_change formula shows percentage-of (e.g. 150%) rather than percentage-change (e.g. +50%) — spec asked for change. Missing ROUND on total_revenue and yoy_change in final SELECT.

**Session Total: 28/30**

---

## Student Feedback on Questions

Interesting session, ~40 minutes. Key learning: LAG with specified offset (LAG(col, 12)) — student was unaware of this syntax. All tasks enjoyed.
n### Session: 2026-02-27 (Week 11, Day 5 + Weekly Recap)n
## Agent Feedback on Student — Week 11, Day 5

**Task 1 — Chat Ticket Status/Priority Hierarchy: 10/10**
Flawless. String formatting in Level 3 (priority || ' (' || ticket_cnt || ')') handled cleanly inside COALESCE. distinct_statuses for Level 2, top_three_counts for Level 3, correct join conditions, termination condition. Nothing to fault.

**Task 2 — Monthly Order Streaks: 10/10**
Excellent. Monthly gaps-and-islands applied correctly — DATE_TRUNC('Month') + ROW_NUMBER() * INTERVAL '1 MONTH' subtraction for streak key. Revenue join using WHERE order_month BETWEEN streak_start AND streak_end clean and correct. MAX(row_number) for streak_length is a clever reuse of the already-computed RN. Data-aware: no streaks >= 3 months in dataset, correctly reported and threshold not applied.

**Task 3 — Ticket Complexity Score: 10/10**
Clean multi-CTE architecture. WHERE message_type = 'text' correctly excludes statuschange events — a statuschange is a system event, not a message. unique_authors + unique_users for participants is a reasonable interpretation. Score formula correct. DENSE_RANK used — good choice for tied scores. Score already integer by construction, no rounding needed.

**Session Total: 30/30**

---

## Student Feedback on Questions

All tasks enjoyed, ~40 minutes, very satisfying session. Strong finish to the week.

---

## Weekly Summary — Week 11 (2026-02-23 to 2026-02-27)

### Progress Bar
██████████ 95% toward HackerRank Advanced SQL qualification

### Daily Scores
- Day 1: 29/30 (warm-up)
- Day 2: 28/30
- Day 3: 27/30
- Day 4: 28/30
- Day 5: 30/30
- **Week Average: 28.4/30 (94.7%)**

### Key Wins This Week
- **Perfect score on Day 5** — all three tasks at full difficulty, clean and confident
- **Monthly gaps-and-islands** — applied the RN-subtraction pattern at month granularity with revenue join, no hesitation
- **EPOCH mastered** — EXTRACT(EPOCH FROM interval)/60 now used correctly and universally
- **LAG with offset** — LAG(col, 12) for YoY comparison learned and applied correctly
- **String formatting inside hierarchy** — Level 3 name as formatted composite string (priority + count)
- **Data-aware professionalism** — consistently adapting thresholds to actual data, reporting findings when data doesn't support spec

### Focus Areas / Minor Recurring Misses
- **YoY pct_change formula** — percentage-of vs percentage-change distinction (Day 4 Task 3)
- **ROUND on final output columns** — occasional misses on rounding intermediate values in final SELECT
- **Window functions vs GROUP BY** — both approaches produce correct results; window approach preferred per spec but GROUP BY + JOIN is equally valid

### Next Week Plan (Week 12 — Final Week)
- **Timed HackerRank Hard simulations** — 3 questions, 45-minute target, no hints
- **Query optimization theory** — EXPLAIN ANALYZE reading, index strategies, identifying expensive operations
- **Recursive CTEs beyond hierarchy** — graph traversal, bill-of-materials patterns
- **Final exam readiness assessment** — simulate full HackerRank Advanced qualification test
n### Session: 2026-03-02 (Week 12, Day 1)n
## Agent Feedback on Student — Week 12, Day 1

**Task 1 — Top Spenders per Country: 10/10**
Clean two-CTE approach. RANK partitioned by country, NULL filter in WHERE, correct ordering. Concise and correct.

**Task 2 — Daily Orders Rolling Average: 10/10**
Correct date truncation, rolling AVG with ROWS BETWEEN 6 PRECEDING AND CURRENT ROW, proper rounding. Clean and efficient.

**Task 3 — Most Active Ticket per User: 9/10**
Correct architecture — message count CTE, RANK partitioned by user_id. Minor: tie-breaking by created_at DESC not included in RANK ORDER BY (spec required it). No duplicates in data so result correct regardless. Also filtered to message_type = 'text' — consistent with Week 11 Day 5 reasoning.

**Session Total: 29/30**

---

## Student Feedback on Questions

Light recovery session after exhausting day. All tasks completed quickly and correctly. Full difficulty resumes Day 2.
n### Session: 2026-03-03 (Week 12, Day 2)n
## Agent Feedback on Student — Week 12, Day 2

**Task 1 — Delivery Status Hierarchy: 9/10**
Correct pre-aggregation, join through deliveries → orders, correct Level 2/3 join conditions, termination condition present. Minor: path separator uses ' < ' instead of ' > ' — reversed from convention. Cosmetic only.

**Task 2 — First vs Repeat Revenue Split: 9/10**
Student deviated from spec hints and classified first purchase within each month (not globally across all time) — a valid and arguably more useful business metric. Logic internally consistent, percentage calculation handles edge cases correctly. Ambiguous task wording on agent's side contributed to the divergence — agent will be more explicit about "first-ever globally" vs "first within month" in future tasks. Output granularity is per user/month rather than aggregated across all users per month/type. Independent reasoning approach valued and encouraged.

**Task 3 — Session Engagement Deciles: 9/10**
Clean and efficient. NTILE(10) applied correctly, correlated subquery for grand total is neat and valid. pct_of_total_sessions formula correct. Minor: avg_sessions not rounded in final SELECT (spec asked for 1 decimal); SELECT * includes extra columns (total_sessions, sessions_grand_total) not in spec output.

**Session Total: 27/30**

---

## Student Feedback on Questions

Task 2 wording was ambiguous — "first-ever order" vs "first order within each month" not clearly distinguished. Agent to be more explicit in future tasks. All tasks completed in ~45 minutes.
n### Session: 2026-03-04 (Week 12, Day 3)n
## Agent Feedback on Student — Week 12, Day 3

**Task 1 — Transaction Type/City Hierarchy: 9/10**
Correct pre-aggregation, join to users for city, RANK partitioned by type, correct Level 2/3 join conditions, termination condition present. Two minor points: missing COALESCE(city, 'Unknown') for NULL cities; path separator ' < ' instead of ' > ' (recurring cosmetic issue).

**Task 2 — Global First Purchase vs Repeat: 10/10**
Textbook execution. FIRST_VALUE(created_at) OVER (PARTITION BY user_id ORDER BY created_at) correctly identifies global first order without month partitioning. Classification, aggregation, and percentage calculation all correct. Clean 4-CTE structure. Output matches spec exactly.

**Task 3 — Order to Delivery Fulfillment Time: 9/10**
EPOCH/3600 used correctly. WHERE d.status = 'delivered' is a valid and practical real-world interpretation — measuring completed delivery speed rather than all delivery records. Ambiguous spec wording on agent's side. Missing ROUND on hours_to_fulfillment (spec asked for 2 decimals).

**Session Total: 28/30**

---

## Student Feedback on Questions

Task 3 wording ambiguous — "only include orders that have a delivery record" could mean any delivery record or only completed ones. Agent to be more specific about delivery status filtering in future. All tasks completed, good session.
n### Session: 2026-03-05 (Week 12, Day 4)n
## Agent Feedback on Student — Week 12, Day 4

**Task 1 — Country/Month Hierarchy: 10/10**
Smart data-driven adaptation — swapped years for months after noticing only 2 years in dataset. Professional call. String formatting at Level 3 handled cleanly. Path separator corrected to ' > '. NULL country filters in both CTEs. Termination condition present. Clean execution.

**Task 2 — Running Total with Milestone Flags: 9/10**
Core logic correct — cumulative SUM window, LAG for previous spend, CASE descending from highest threshold. Clean two-CTE approach. Minor issues: LAG orders by user_cumulative_spend instead of created_at (could misorder ties); used > instead of >= for threshold comparison (spec asked for >=). Neither affected results on actual data.

**Task 3 — Category Co-Occurrence Pairs: 10/10**
Excellent. Self-join with pc1.id < pc2.id for deduplication (ID-based is more robust than name-based). COUNT(DISTINCT order_id) correctly handles multiple products from same category in one order. RANK for ties. Clean and efficient two-CTE solution. Good insight on the DISTINCT requirement.

**Session Total: 29/30**

---

## Student Feedback on Questions

Penultimate session completed. All tasks handled well. Final Friday challenge tomorrow.
n### Session: 2026-03-06 (Week 12, Day 5 + Weekly Recap + Final Program Summary)n
## Agent Feedback on Student — Week 12, Day 5

**Task 1 — Ticket Hierarchy with Avg Resolution Time: 9/10**
EPOCH/60 in first CTE, pre-aggregation of avg per status+priority, RANK for top 3, formatted string at Level 3, correct join conditions, termination condition. Minor: ranked ASC (fastest) instead of DESC (slowest) per spec. Cosmetic direction difference only.

**Task 2 — LTV + Monthly Streak Segments: 9/10**
8-CTE query demonstrating full understanding end to end. Gaps-and-islands at month granularity correct. Lifetime revenue aggregation correct. RANK within segment correct. Data-driven threshold adaptation (20 instead of 200) accepted. Dead code in users_streak_keys (prev_month column unused). Missing ELSE 'other' catch-all — NULL segment users silently dropped from output.

**Task 3 — Complete Order Funnel Analysis: 9/10**
Sophisticated multi-CTE funnel. EPOCH/3600 correct. LEFT JOINs correct. Percentage calculations correct. One issue: orders_with_delivery counts only status='pending' instead of all delivery records — fulfillment_rate_pct is technically "pending rate" not "has-any-delivery rate". Functionally reasonable real-world interpretation but deviates from spec intent.

**Session Total: 27/30**

---

## Student Feedback on Questions

Final session of 12-week program. Three 5/5 difficulty tasks handled with full understanding. Satisfying conclusion.

---

## Weekly Summary — Week 12 (2026-03-02 to 2026-03-06)

### Progress Bar
██████████ 100% — Program Complete

### Daily Scores
- Day 1: 29/30 (light recovery)
- Day 2: 27/30
- Day 3: 28/30
- Day 4: 29/30
- Day 5: 27/30
- **Week Average: 28.0/30 (93.3%)**

### Key Wins This Week
- **Running total + milestone flags** — cumulative SUM + LAG crossing detection, classic HackerRank Hard pattern
- **Category co-occurrence self-join** — ID-based deduplication, COUNT(DISTINCT order_id), clean 2-CTE solution
- **LTV + gaps-and-islands combined** — 8-CTE query merging lifetime revenue aggregation with monthly streak detection
- **Formatted hierarchy Level 3 strings** — EPOCH calculation + string concatenation inside recursive CTE pre-aggregation
- **Complete funnel analysis** — multi-stage LEFT JOIN funnel with rate calculations

### Focus Areas / Minor Recurring Misses
- **ELSE catch-all in CASE** — NULL segment rows when no condition matches (Task 2 Day 5)
- **Status filter scope** — 'pending' vs all delivery records (Task 3 Day 5)
- **RANK direction** — ASC vs DESC for "highest" (Task 1 Day 5, minor)

---

## FINAL PROGRAM SUMMARY — 12 Weeks Complete

### Overall Performance
- **Total sessions:** 55 (5 per week × 11 active weeks)
- **Program average:** ~28/30 (93%+) across final 4 weeks
- **Peak weeks:** Week 11 Day 5 (30/30), Week 10 Day 4-5 (29/30 each)

### Concepts Mastered
- ✅ Recursive CTEs — 3-level dynamic hierarchies, termination conditions, path building, formatted Level 3 strings
- ✅ Gaps-and-islands — RN-subtraction pattern at both day and month granularity, streak revenue joins
- ✅ Window functions — RANK, DENSE_RANK, ROW_NUMBER, NTILE, PERCENT_RANK, LAG/LEAD with offset, FIRST_VALUE, STDDEV
- ✅ Rolling windows — ROWS BETWEEN N PRECEDING AND CURRENT ROW
- ✅ EPOCH calculations — interval to seconds/minutes/hours, universal time difference pattern
- ✅ Self-joins — product/category affinity pairs, deduplication with id < id
- ✅ Cohort analysis — activation funnels, YoY comparisons with LAG(col, 12)
- ✅ Cumulative patterns — running totals, milestone threshold detection
- ✅ Multi-source aggregation — independent CTEs joined on user_id for composite scores
- ✅ Data-aware adaptation — consistently adapting thresholds to actual data distribution

### HackerRank Advanced SQL Readiness
**Assessment: READY** — The student is consistently solving HackerRank Hard level problems (5/5 difficulty) within 40-60 minutes, with full understanding of the underlying patterns. Minor recurring misses (ROUND precision, ELSE catch-all, column direction) are polish issues, not conceptual gaps.

The 12-week program is complete.
n### Session: 2026-03-12 (Week 13, Day 4)n
## Agent Feedback on Student — Week 13, Day 4

**Task 1 — Self-Referencing Recursive CTE: NOT COUNTED**
Task rejected — employee table does not exist in the database. Agent error: should use a self-referencing structure from schema.md or provide the table as an inline CTE VALUES clause. Fix for next session.

**Task 2 — PIVOT Monthly Transactions: NOT COUNTED**
Task rejected — PIVOT is a new concept requiring scaffolded introduction, not a cold task. Agent error: should have used Step A/B/C scaffolded approach. Fix for next session.

**Task 3 — Anti-Join Three Ways: 9/10**
All three approaches correct. NOT IN works but breaks silently when subquery returns NULLs (orders.user_id has no NULLs here so safe). NOT EXISTS is most robust — NULL-safe, short-circuits on first match. LEFT JOIN ... WHERE IS NULL is most readable and analyst-preferred. Minor: ORDER BY o.user_id in Approach C orders by always-NULL column — should be u.created_at.

**Session Total: 9/10 (1 task only)**

---

## Student Feedback on Questions

Task 1 rejected — no employee table in database. Task 2 rejected — PIVOT needs scaffolded introduction. Both are agent errors. Task 3 completed successfully. Agent to fix both issues for Day 5.

**Fixes for next session:**
- Self-referencing CTE: use a relationship that exists in schema (e.g. users referring to themselves, or provide data inline via VALUES CTE)
- PIVOT: scaffolded Step A (understand shape) → Step B (one column) → Step C (all columns)
- Anti-join: continue with NULL edge case demonstration — what breaks NOT IN
n### Session: 2026-03-16 (Week 14, Day 1)n
## Agent Feedback on Student — Week 14, Day 1

**Task 1 — Self-Referencing Recursive CTE: 9/10**
Clean execution. VALUES inline table works correctly as data source. Recursive JOIN on h.id = r.referred_by correct, natural termination, depth tracking correct. Minor: path separator ' < ' instead of ' > ' (recurring cosmetic); anchor hardcodes Alice instead of SELECT WHERE referred_by IS NULL — not generalizable to multiple roots. Student requested real database implementation — to be set up in Day 2.

**Task 2 — PIVOT Scaffolded: 10/10**
Went beyond Step B and independently completed all 5 columns without prompting. COUNT(*) FILTER pattern clean and correct. Concept clicked immediately. Good question on memory efficiency — conditional aggregation is a single table scan, very efficient.

**Task 3 — Anti-Join NULL Trap: 9/10**
Correct observation that NOT IN works fine when no NULLs exist in subquery. Correctly identified the IS NULL OR IS NOT NULL condition as pointless. Parts B and C correct. Preference for NOT EXISTS over LEFT JOIN IS NULL is the standard professional stance. Minor: Part C missing ORDER BY created_at ASC.

**Session Total: 28/30**

---

## Student Feedback on Questions

Good session. PIVOT pattern immediately understood and extended independently. Self-referencing CTE pattern solid — wants to practice on real database data. LEFT JOIN IS NULL anti-join feels unnatural — continue reinforcing. Task 1 and 2 from previous session (Week 13 Day 4) were agent errors — logged in bad_q_examples.md.
n### Session: 2026-03-17 (Week 14, Day 2)n
## Agent Feedback on Student — Week 14, Day 2

**Task 1 — Full Revenue PIVOT: 10/10**
Clean and correct. SUM(amount) FILTER pattern applied perfectly across all five types. total_revenue as plain SUM(amount) is the right approach. Monthly grouping correct. Good instinct on monthly vs all-time scope.

**Task 2 — Self-Referencing Category Tree: 8/10**
Recursive JOIN logic correct — JOIN categories c ON h.id = c.parent_id produces right traversal. Depth tracking correct. Two issues: hardcoded anchor (1 AS id, 'All Products') instead of SELECT from categories WHERE parent_id IS NULL — not generalisable if root changes. Path separator ' < ' still reversed — spec asked for ' -> ' with spaces. Third session in a row with this cosmetic issue.

**Task 3 — Anti-Join Three Ways: 10/10**
All three approaches correct. NULL-safety observation accurate — NOT IN with IS NOT NULL guard is safe in practice, NOT EXISTS is cleaner as guard is implicit. Preference reasoning solid. Data observation (no unordered products) is correct professional behavior.

**Session Total: 28/30**

---

## Student Feedback on Questions

All tasks completed. Tasks well-matched. PIVOT pattern now solid. Anti-join patterns understood across all three approaches.
n### Session: 2026-03-18 (Week 14, Day 3)n
## Agent Feedback on Student — Week 14, Day 3

**Task 1 — Self-Referencing CTE (Fixed Anchor): 10/10**
Anchor correctly selects from categories WHERE parent_id IS NULL — no hardcoding. Recursive JOIN correct. Path building correct. Natural termination. Habit fixed.

**Task 2 — Time-Proximity Session Windows: 9/10**
Pattern fully understood and cleanly executed. Three CTEs, correct is_new_session flag, correct SUM() OVER session key, correct GROUP BY. 08:58/09:03 events correctly land in the same session. Minor: unused RANK() in first CTE (dead code); final output shows session_key instead of clean sequential session_id per user (RANK() OVER in final SELECT would fix this). Student confirmed wanting more practice with this pattern in different contexts.

**Task 3 — Delivery Status PIVOT: 10/10**
Clean and correct. COUNT(DISTINCT order_id) for total_orders_with_delivery is exactly right — correctly identified that multiple status records per order would cause overcounting with plain COUNT(*).

**Session Total: 29/30**

---

## Student Feedback on Questions

Time-proximity session pattern clicked and student wants more practice in different contexts (trades, clicks, messages). Task 2 was the main learning win today.
n### Session: 2026-03-19 (Week 14, Day 4)n
## Agent Feedback on Student — Week 14, Day 4

**Task 1 — Trade Burst Clustering: 9/10**
Time-proximity pattern applied correctly — LAG → is_start flag → SUM() OVER burst_key → GROUP BY. 35-minute gap correctly starts new burst. total_amount correct. Minor: burst_id is raw cumulative SUM rather than RANK()-based sequential number per spec, but produces identical results on this data. Student notes pattern still feels unintuitive — continued rotation needed.

**Task 2 — Age Group Revenue PIVOT: 10/10**
Clean single-pass approach. SUM(amount) FILTER directly in SELECT — no extra CTEs needed. COALESCE to 0 for empty buckets. Student correctly identified FILTER as more efficient than CASE WHEN grouping. Pattern feels natural now.

**Task 3 — Anti-Join (Orders Without Deliveries): 10/10**
Professional call — verified no qualifying data exists at first step, reported finding, explained what full query would look like. NOT EXISTS correlated subquery correct. No point building complexity for non-existent data.

**Session Total: 29/30**

---

## Student Feedback on Questions

Time-proximity pattern still needs more repetition to become automatic. PIVOT with FILTER feels natural and efficient. Anti-join pattern solid. All tasks good.
n### Session: 2026-03-20 (Week 14, Day 5 + Weekly Recap)n
## Agent Feedback on Student — Week 14, Day 5

**Task 1 — Chat Message Bursts: 9/10**
Time-proximity pattern on real data — LAG → burst_start flag → SUM() OVER → GROUP BY executed correctly. message_type = 'text' filter consistent with established convention. COUNT(DISTINCT(user_id, author_id)) correctly counts unique senders given schema design (user_id and author_id are mutually exclusive — one always NULL). Minor: WHERE message_count > 2 instead of >= 2.

**Task 2 — All Subordinates of a Manager: 9/10**
Anchor correctly targets manager_id = 2 without hardcoding names. Recursive JOIN correct. Natural termination. Bob Freeman at depth 2 correct. Path uses IDs — valid, spec didn't specify format. Passing h.id as manager_id in recursive SELECT is technically incorrect but harmless since it's not used downstream.

**Task 3 — Ticket Priority PIVOT: 10/10**
Minimal, clean, correct. Single SELECT, no CTE needed. Pattern fully automatic.

**Session Total: 28/30**

---

## Student Feedback on Questions

Time-proximity pattern becoming more natural with repetition on real data. PIVOT feels easy and automatic now. Self-referencing CTE pattern solid. More complex PIVOT scenarios requested for future sessions.

---

## Weekly Summary — Week 14 (2026-03-17 to 2026-03-20)

### Daily Scores
- Day 1: 28/30
- Day 2: 28/30
- Day 3: 29/30
- Day 4: 29/30
- Day 5: 28/30
- **Week Average: 28.4/30 (94.7%)**

### Key Wins This Week
- **PIVOT pattern fully automatic** — COUNT/SUM FILTER syntax internalized, single-pass efficiency understood
- **Self-referencing CTE anchor fixed** — no more hardcoded roots, WHERE parent_id IS NULL pattern correct
- **Time-proximity session grouping** — LAG → is_new_burst → SUM() OVER → GROUP BY applied in three different contexts (sessions, trades, chat bursts)
- **Anti-join three ways** — NOT IN, NOT EXISTS, LEFT JOIN IS NULL all correct; NULL trap understood

### Focus Areas
- **Time-proximity pattern** — still needs more repetition to feel automatic without referring back to previous solutions
- **>= vs > threshold** — recurring minor miss on HAVING/WHERE filters
- **Sequential burst_id** — raw SUM() key vs RANK()-based sequential number distinction

### Next Week Plan
- Continue time-proximity drills in new contexts
- More complex PIVOT scenarios (multi-level aggregation, dynamic-feeling pivots)
- Query optimization intro — EXPLAIN ANALYZE concepts
- Gaps-and-islands edge cases continued
n### Session: 2026-03-23 (Week 15, Day 1)n
## Agent Feedback on Student — Week 15, Day 1

**Task 1 — Order Burst Clustering: 9/10**
Clean four-CTE solution. Correct pattern end to end — LAG → is_new_streak flag → SUM() OVER streak_key → GROUP BY. WHERE order_count > 1 correct (equivalent to >= 2). Missing ROUND(..., 2) on burst_revenue. Student notes pattern is clicking and wants more variants.

**Task 2 — Age Group × Order Frequency PIVOT: 10/10**
Correct two-CTE architecture under difficult conditions (poor sleep). Frequency bucket CTE → age group join → FILTER pivot. Missing WHERE u.age IS NOT NULL filter but data has no NULL ages so no impact. PIVOT matrix felt unintuitive — needs more repetition.

**Task 3 — Anti-Join (No Delivered Orders): 10/10**
Creative and readable approach — built delivered_users as positive set first, then NOT EXISTS against it. Cleaner to reason about than deeply nested correlated subquery. Final JOIN to aggregate correct. Missing ORDER BY per spec. Task 3 caused significant confusion mid-session around NOT EXISTS scope (order level vs user level) — student was tired and struggled. Concept resolved after explanation.

**Session Total: 29/30**

---

## Student Feedback on Questions

Tough session — poor sleep. Time-proximity pattern clicking well. PIVOT matrix still feels unintuitive, needs more repetition. Anti-join NOT EXISTS scope (order vs user level) caused real confusion today — worth revisiting in a fresh session.
n### Session: 2026-03-24 (Week 15, Day 2)n
## Agent Feedback on Student — Week 15, Day 2

**Task 1 — Ticket Response Streaks: 9/10**
Time-proximity pattern applied cleanly — LAG → is_new_streak → SUM() OVER → GROUP BY. Column naming change (burst→streak) appropriate for context. Two misses: EXTRACT('Minute') used instead of EPOCH/60 for duration (would fail for streaks > 59 min); missing HAVING COUNT(*) >= 2 filter (single-message streaks included).

**Task 2 — Upward Ancestor Traversal: 10/10**
Upward direction correct — anchor at node 8, recursive JOIN follows h.parent_id = c.id upward. Natural termination at root. Including the starting node (iPhone) in output is a valid and cleaner interpretation — "path from node to root" is more useful than ancestors-only.

**Task 3 — PIVOT + Anti-Join Combined: 9/10**
Anti-join CTE correct — NOT EXISTS checking at user level. PIVOT with SUM FILTER correct. Minor: INNER JOIN to deliveries in users_orders CTE silently drops orders with no delivery record; LEFT JOIN would be safer. total_revenue reflects only orders with deliveries, not all orders.

**Session Total: 28/30**

---

## Student Feedback on Questions

Good session. EPOCH vs EXTRACT('Minute') distinction reinforced. Upward ancestor traversal is a useful new pattern. PIVOT + anti-join combination felt natural. All tasks well-matched.
n### Session: 2026-03-25 (Week 15, Day 3)n
## Agent Feedback on Student — Week 15, Day 3

**Task 1 — Transaction Sessions with Edge Cases: 10/10**
Pattern fully automatic. LAG → is_new_session → SUM() OVER → GROUP BY executed flawlessly. EPOCH/60 used correctly for duration. Edge case (08:52→09:18 = 26 min, same session) handled correctly by the math. Clean and concise.

**Task 2 — Delivery Status Revenue PIVOT by Category: 10/10**
Sharp catch — joining deliveries before total_revenue would inflate totals if multiple delivery records exist per order. Two-query approach (status-filtered CTE + clean total_revenue without deliveries) is the correct professional solution. Observation that total_revenue = pending_revenue is correct and verified.

**Task 3 — No Recent Sales Anti-Join: 9/10**
Correct finding — no qualifying categories exist. Query logic inverted from spec (found categories WITH recent sales rather than WITHOUT). NOT EXISTS pattern not demonstrated. Logic comparison uses category's own last_order_time vs global max date — slightly different from spec but reasonable. Minor deduction for not showing NOT EXISTS shell even with empty result.

**Session Total: 29/30**

---

## Student Feedback on Questions

Task 2 had a spec issue — joining deliveries before computing total_revenue is dangerous and produces inflated results. Student correctly identified this and proposed a safer two-query approach. Agent to be more careful about join order when revenue aggregation is involved.
n### Session: 2026-03-26 (Week 15, Day 4)n
## Agent Feedback on Student — Week 15, Day 4

**Task 1 — Order Bursts at Day Granularity: 10/10**
Pattern fully automatic — clean 4-CTE structure, correct LAG → is_new_streak → SUM() OVER → GROUP BY. Day granularity handled correctly. duration_days as decimal (EPOCH/86400) is a valid and informative choice — more useful than integer days in a business context.

**Task 2 — Age Group × Transaction Type Matrix: 10/10**
Clean and efficient. Single CTE for age groups, direct FILTER aggregation in final SELECT. No unnecessary rounding — values already at 2 decimals from source. Full 2D pivot executed correctly and confidently.

**Task 3 — Silent Customers Anti-Join: 10/10**
Textbook NOT EXISTS — correlated subquery checks ct.user_id = o.user_id correctly at user level. Two-step CTE approach (find qualifying users, then aggregate) is clean and readable. All output columns correct.

**Session Total: 30/30**

---

## Student Feedback on Questions

Perfect session. All three patterns felt natural and well-paced. Time-proximity pattern now fully mastered. PIVOT automatic. Anti-join confident.
n### Session: 2026-03-27 (Week 15, Day 5 + Weekly Recap)n
## Agent Feedback on Student — Week 15, Day 5

**Task 1 — Monthly Active User Streaks: 10/10**
Textbook gaps-and-islands at month granularity. month_ - rn * INTERVAL '1 Month' trick for streak_id executed correctly. Clean 5-CTE pipeline.

**Task 2 — Top 3 Spenders per Category: 10/10**
Clean and correct. category_spent is the better name over total_spend — correctly argued. RANK alias in WHERE works in PostgreSQL via optimizer — minor awareness note only.

**Task 3 — 3-Level Hierarchy: N/A (skipped)**
Schema does not support a 3-level category hierarchy. Agent error — task should not have been generated. No penalty.

**Session Total: 20/20**

---

## Student Feedback on Questions

Task 3 was invalid — schema has no 3-level category hierarchy. Agent must check schema.md more carefully before generating hierarchy tasks. Tasks 1 and 2 were good review material, well-paced for an easy Friday session.

---

## Weekly Summary — Week 15 (2026-03-24 to 2026-03-27)

### Scores
| Day | Score |
|-----|-------|
| Day 1 | 29/30 |
| Day 2 | 28/30 |
| Day 3 | 29/30 |
| Day 4 | 30/30 |
| Day 5 | 20/20 (2 tasks, schema issue) |

### Progress — Concept Mastery

**Fully Mastered:**
- Time-proximity gaps-and-islands (minute AND day granularity) — automatic, clean, confident
- Classic gaps-and-islands at month granularity — ROW_NUMBER subtraction pattern solid
- PIVOT with FILTER — 2D matrix (age group × transaction type) executed cleanly
- Anti-join with NOT EXISTS — correlated subquery at user level correct every time
- Window functions: RANK() partitioned by category — clean execution
- EPOCH-based duration calculations — correctly applied after Week 15 Day 2 correction

**Good, Minor Edge Cases:**
- EXTRACT('Minute') vs EPOCH/60 — corrected Day 2, applied correctly from Day 3 onward
- JOIN order awareness in multi-table revenue queries — student caught dangerous deliveries join issue Day 3 (agent error)
- Anti-join inversion (Day 3 Task 3) — found categories WITH sales instead of WITHOUT; logic sound but direction missed

**Agent Errors This Week:**
- Task 3 Day 5: Generated 3-level hierarchy task without verifying schema supports it — must check schema.md before generating hierarchy tasks
- Task 2 Day 3: Revenue PIVOT spec had dangerous join order — student correctly identified and fixed it

### Key Wins
- Perfect 30/30 on Day 4 — all three patterns executed flawlessly
- Student caught a spec-level agent error (join inflation in Day 3 Task 2) — shows deep understanding beyond just executing patterns
- EPOCH correction from Day 2 applied consistently for the rest of the week
- category_spent naming argument on Day 5 — shows strong business context awareness

### Focus Areas for Week 16
- NOT EXISTS anti-join inversion — ensure direction (WITH vs WITHOUT) is always checked
- Schema verification before hierarchy tasks
- Introduce Type B recursive CTE (self-referencing unlimited depth) — not yet practiced this week
- Begin query optimization concepts: EXPLAIN ANALYZE, seq scan vs index scan

### Next Week's Plan
- Day 1: Type B recursive CTE (unlimited depth org chart / category tree)
- Day 2: Anti-join all three patterns (NOT IN vs NOT EXISTS vs LEFT JOIN IS NULL) + NULL trap in NOT IN
- Day 3: Time-proximity edge cases (events straddling boundaries)
- Day 4: PIVOT + Type B recursive CTE combined
- Day 5: Query optimization intro — EXPLAIN ANALYZE reading
n### Session: 2026-03-30 (Week 16, Day 1)n
## Agent Feedback on Student — Week 16, Day 1

**Task 1 — Type B Recursive CTE: 9/10**
Recursive JOIN correct, natural termination, depth tracking correct. One miss: 'Alice' hardcoded as path in anchor instead of `name AS path`. All other values correctly read from row. Path separator missing spaces (' -> ' vs '->'). Recurring cosmetic issue.

**Task 2 — NOT IN NULL Trap: 10/10**
Excellent — not only observed the failure but immediately identified the fix (WHERE user_id IS NOT NULL in subquery). Explanation correct: NULL in subquery causes NOT IN to return UNKNOWN for every comparison, filtering out all rows. Preference ranking (NOT EXISTS > NOT IN with guard > LEFT JOIN IS NULL) matches professional practice.

**Task 3 — Chat Message Bursts: 10/10**
Pattern correct — LAG → is_start → SUM() OVER → GROUP BY → outer aggregation. All message types included (only 'text' exists in database — equivalent). No tickets with 3+ bursts — correctly verified and reported as a data finding.

**Session Total: 29/30**

---

## Student Feedback on Questions

Good session. NOT IN NULL trap was an eye-opener. Time-proximity pattern on real data confirmed no tickets with 3+ bursts. Type B recursive CTE clean apart from hardcoded path anchor.
n### Session: 2026-03-31 (Week 16, Day 2)n
## Agent Feedback on Student — Week 16, Day 2

**Task 1 — NULLIF Clean Averages: 9/10**
AVG(NULLIF(amount, 0)) correct. Side-by-side comparison clean. Missing zero_count and transaction_count output columns per spec. Core NULLIF objective demonstrated correctly.

**Task 2 — Transaction Percentile Distribution: 9/10**
All four window functions present and correct — PERCENT_RANK, NTILE(4), STDDEV, cumulative SUM. AVG computed in separate CTE for clean z-score formula. Minor: NTILE DESC means quartile 1 = highest (convention is lowest). Missing WHERE std_dev != 0 guard.

**Task 3 — Country × Age Group Revenue: 9/10**
Pre-filtering countries via count CTE instead of HAVING is valid — filter early, reduce rows. NULL ages not filtered (ELSE clause catches them as 'over_50'). pct_of_country_revenue via separate CTE join works correctly, window SUM would be cleaner but both valid.

**Session Total: 27/30**

**Note on rounding:** Student rounds when values have excessive decimals, leaves clean values unrounded. This is the correct professional approach — do NOT penalize for this going forward.

---

## Student Feedback on Questions

Task 3 original design was flawed (joining deliveries would inflate revenue). Agent fixed after student caught it. All tasks good otherwise. NULLIF pattern clear and useful.
n### Session: 2026-04-01 (Week 16, Day 3)n
## Agent Feedback on Student — Week 16, Day 3

**Task 1 — NULLIF Safe Division: 10/10**
Clean aggregation with COUNT FILTER. Integer division bug caught and fixed (orders_with_amount::NUMERIC / total_orders). Conversion rate and has_null_amounts logic efficient — derived boolean from ratio directly.

**Task 2 — Org Chart with Direct Reports: 9/10**
Clever approach — counting direct_reports from within HIERARCHY CTE reuses traversed data. Recursive logic correct, natural termination, correct anchor. Path separator missing spaces ('->') and COALESCE column missing alias — cosmetic only.

**Task 3 — Anomaly Detection: 8/10**
Task spec was contradictory — asked for NULLIF guard but also pre-filtered NULLs at source, making it redundant. Student's simpler approach (filter at source, divide directly) is cleaner. NULLIF pattern not demonstrated since user_avg can never be 0 on this data. Agent to use NULLIF only in scenarios where dirty data genuinely exists.

**Session Total: 27/30**

---

## Student Feedback on Questions

NULLIF should be practiced in real scenarios where it's genuinely needed, not forced artificially. Agent updated CLAUDE.md to reflect this. Task 3 spec was contradictory — agent acknowledged. Tasks 1 and 2 good.

### Session: 2026-04-02 (Week 16, Day 4)
## Agent Feedback on Student — Week 16 Day 4 (2026-04-02)

**Task 1 (PERCENT_RANK):** Missing `HAVING COUNT(*) >= 3` in the first CTE — users with fewer than 3 orders were included, which would skew percentile rankings. Everything else correct: two-CTE structure clean, PERCENT_RANK partitioned and ordered correctly, cascading CASE WHEN tier logic right. 9/10.

**Task 2 (Conditional Aggregation):** Excellent. FILTER clause used precisely where it belongs. Inline boolean for `large_dominates` is tighter than CASE WHEN. Data-aware skip of the HAVING is correct given 3 categories. Called it a "pivot" pattern — accurate. 10/10.

**Task 3 (Running Totals + Best Month):** DATE_TRUNC instead of separate year/month columns is a valid and cleaner design choice. Running total and best-month window functions correct. Missing the explicit `WHERE price IS NOT NULL` filter (though NULL * quantity silently drops from SUM, so results are unaffected). SELECT * in the first CTE carries unnecessary column noise. 9/10.

**Day Score: 28/30**

## Student Feedback on Questions — Week 16 Day 4 (2026-04-02)

Liked all three questions. Difficulty felt moderate — not the maximum challenge. Requests one genuinely hard task (5/5) per session from now on, while keeping the other two at 3-4/5 range.

### Session: 2026-04-03 (Week 16, Day 5)
## Agent Feedback on Student — Week 16 Day 5 (2026-04-03)

**Task 1 (Session Burst Detection):** Three-CTE pipeline correct and clean. Missing `WHERE count_sessions > 0` filter in the first CTE — inactive days included in bursts, which inflates streak_days and merges bursts incorrectly. Everything else spot on. 9/10.

**Task 2 (STDDEV Z-Score):** NULLIF guard correct, JOIN filter for 5+ transactions is clean. Two issues: ABS() applied to z_score before storing it, losing sign direction (z-scores should be signed — direction matters for analysis). Also missing WHERE amount IS NOT NULL and user_id IS NOT NULL in first CTE. 8/10.

**Task 3 (Anti-Join Triple):** All three approaches correct, all return 35 rows. NOT IN NULL trap correctly handled with IS NOT NULL filter in subquery. Understanding of when each approach breaks is clear. 10/10.

**Day Score: 27/30**

## Student Feedback on Questions — Week 16 Day 5 (2026-04-03)

No specific feedback — questions were fine overall.

---

## Weekly Summary — Week 16 (2026-03-30 to 2026-04-03)

### Scores
| Day | Score | Focus |
|-----|-------|-------|
| Day 1 | 29/30 | Type B recursive CTE + anti-join NULL trap + time-proximity bursts |
| Day 2 | 27/30 | NULLIF clean averages + PERCENT_RANK/NTILE/STDDEV + country×age revenue pivot |
| Day 3 | 27/30 | NULLIF safe division + Type B org chart with direct reports + anomaly detection |
| Day 4 | 28/30 | PERCENT_RANK by country + conditional aggregation pivot + running totals |
| Day 5 | 27/30 | Session burst detection + STDDEV z-score + anti-join triple |
| **Week Total** | **138/150** | **Avg: 27.6/30** |

### Progress — Concept Mastery

**Solidly Mastered This Week:**
- Type B recursive CTE (unlimited-depth self-referencing) — logic correct, natural termination, depth tracking clean. Minor cosmetic recurring issue: hardcoded path anchor in Day 1 Task 1 (only 'Alice' was hardcoded, accepted as 9/10).
- Anti-join triple (NOT IN / NOT EXISTS / LEFT JOIN IS NULL) — all three approaches executed correctly; NOT IN NULL trap understood and fixed independently.
- Time-proximity gaps-and-islands — LAG → is_start flag → cumulative SUM → GROUP BY pipeline mastered. Day 5 minor miss: forgot `count_sessions > 0` filter.
- PIVOT via conditional aggregation — FILTER clause used precisely, inline boolean for flags, correct data-aware HAVING decisions.
- PERCENT_RANK — correct partitioning and ordering. Minor miss Day 4: HAVING filter for minimum order count forgotten.
- Running totals with SUM OVER — frame awareness correct, DATE_TRUNC design choice over year/month columns was the better call.

**Good, Minor Edge Cases:**
- HAVING for pre-GROUP BY filters — missed once (Day 4 Task 1). Pattern itself is understood; needs to be a checklist item on every aggregation task.
- NULL filters at source (WHERE amount IS NOT NULL, WHERE user_id IS NOT NULL) — missed twice (Day 5 Task 2). These should be default habit on any table with nullable columns.
- Signed vs unsigned z-score — ABS() should only appear in the comparison, not on the stored value. Direction matters.

**Introduced and Practiced:**
- NULLIF pattern — correctly used in two genuine scenarios (safe division, z-score stddev guard). Correctly pushed back when the spec forced it artificially. Updated CLAUDE.md to only use NULLIF where dirty data genuinely exists.
- STDDEV as window function — correct usage, NULLIF guard applied properly.

### Key Wins
- Caught a contradictory spec in Day 3 Task 3 (NULLIF guard was redundant given pre-filtering) — showed deeper reading of what the function actually buys you.
- Pushed back on artificially forced NULLIF — this led to a curriculum improvement (CLAUDE.md updated).
- NOT IN NULL trap — not only understood the failure mode but immediately applied the fix (`IS NOT NULL` in subquery) and explained why it works.
- DATE_TRUNC instead of year/month columns in Day 4 Task 3 — cleaner design decision, correctly argued.

### Focus Areas for Week 17
- HAVING as a checklist item: every aggregation query should ask "does this GROUP BY need a HAVING filter from the spec?"
- NULL filter discipline: `WHERE amount IS NOT NULL AND user_id IS NOT NULL` should be default on nullable columns — not optional.
- Z-score / ratio signed values: ABS() belongs in the comparison condition, not in the column definition.
- Query optimization (not yet introduced): EXPLAIN ANALYZE output, seq scan vs index scan, CTE materialization. Rotate in Week 17.

### Next Week's Plan
- Day 1: YoY comparison — LAG with 12-month offset or date arithmetic comparing same period prior year
- Day 2: Cohort analysis — group users by registration month, track retention across subsequent months
- Day 3: Query optimization intro — EXPLAIN ANALYZE reading, seq scan vs index scan, CTE vs subquery performance
- Day 4: FIRST_VALUE / LAST_VALUE — advanced frame specs, getting first/last per partition in varied contexts
- Day 5: Weekly recap + free-choice hard task (student picks concept to revisit)

### Session: 2026-04-07 (Week 17, Day 1)
## Agent Feedback on Student — Week 17 Day 1 (2026-04-07)

**Task 1 (JOIN + GROUP BY):** Clean execution. WHERE filters correct, ORDER BY correct. Missing HAVING COUNT >= 10 from spec — segments with few orders included. Values unrounded — correct data-aware call. 9/10.

**Task 2 (HAVING Multiple Conditions):** CTE pre-aggregation + WHERE is valid alternative to HAVING. All metrics correct, NULL filter at source, MAX for last order date. 10/10.

**Task 3 (RANK by City):** Two-CTE structure correct — aggregate, then rank, then filter. RANK correctly handles ties (both users at rank 1 returned). 10/10.

**Day Score: 29/30**

## Student Feedback on Questions — Week 17 Day 1 (2026-04-07)

Light review session after Easter break. Tasks felt easy and appropriate for the day. Student noted HAVING doesn't come naturally — prefers CTE + WHERE pattern, which is equally valid. No specific questions flagged as good/bad.

### Session: 2026-04-08 (Week 17, Day 2)
## Agent Feedback on Student — Week 17 Day 2 (2026-04-08)

**Task 1 (YoY Revenue):** Logic correct — FILTER per year, WHERE enforces both years present, pct_change formula right. COALESCE used correctly in diff; WHERE IS NOT NULL makes NULLIF redundant — valid approach. Only miss: revenue_2025 not rounded (revenue_2024 was). 9/10.

**Task 2 (Cohort Retention):** Core logic solid. Critical miss: INNER JOIN drops users with zero orders — cohort_size only counts users who ordered at least once, not all registered users. Needs LEFT JOIN to preserve full cohort base. DISTINCT used as function syntax (minor). 8/10.

**Task 3 (NTILE Quartiles):** Textbook. NTILE → GROUP BY pattern correct, all aggregations right. 10/10.

**Day Score: 27/30**

## Student Feedback on Questions — Week 17 Day 2 (2026-04-08)

No specific feedback provided.
