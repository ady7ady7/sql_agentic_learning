# Bad Question Examples

This file stores examples of questions that the Student found unhelpful, confusing, or poorly designed.

---

## Purpose
- Questions here represent what to AVOID when generating new tasks
- Learn from these mistakes to improve question quality
- Add new entries when Student explicitly flags a question as "poor" or "unclear"

---

## Examples

### Week 2, Day 5 - Task 1: Redundant Column Requirements

**Issue:** Required both `activity_type` and `source_table` columns with identical information

**What was wrong:**
```
Expected Output Columns:
- activity_type (varchar) — 'order' or 'transaction'
- source_table (varchar) — 'orders' or 'transactions'
```

**Why it's bad:**
- Redundant: If `source_table = 'orders'`, then obviously `activity_type = 'order'`
- Verbose: Requires students to write the same literal value twice with slightly different text
- No learning value: Just busywork duplication
- Confusing: Makes students wonder if there's a subtle difference they're missing

**Better approach:**
Choose ONE column that conveys the information:
```
Expected Output Columns:
- source_table (varchar) — 'orders' or 'transactions'
```

**Student feedback:** "asking for activity_type when we already have source_table is retarded. We know it's from orders/transactions, and getting to know if it's an order/transaction is just dumb - everybody already knows what it is, and it's just rewriting the same info under a different column name."

---

### Week 3, Day 2 - Multiple Tasks: Overly Prescriptive Requirements (Giving Away the Solution)

**Issue:** Requirements directly specified the exact SQL functions and window syntax to use, eliminating critical thinking

**What was wrong:**

**Task 1 requirement:**
```
- Use DENSE_RANK() OVER (PARTITION BY category_id ORDER BY total_revenue DESC)
```

**Task 2 requirement:**
```
- Use window function with PARTITION BY year, month and ORDER BY date
- Use appropriate frame: ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

**Task 3 requirement:**
```
- Use NTILE(4) OVER (ORDER BY total_transaction_amount DESC) for quartile assignment
```

**Why it's bad:**
- **Removes critical thinking**: Student doesn't need to decide WHICH ranking function (DENSE_RANK vs RANK vs ROW_NUMBER)
- **Gives away the solution**: Literally provides the exact window function syntax
- **No learning value**: Student just copies the provided syntax instead of reasoning about the right approach
- **Eliminates decision-making**: Should student use GROUP BY or window function? Should they use DENSE_RANK or RANK? These are valuable decisions to make
- **Redundant frame clause**: `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` is the **default behavior** when ORDER BY is used in window functions, making it confusing and unnecessary
- **No exploration**: Student doesn't get to practice evaluating trade-offs between different approaches

**Better approach:**

**Task 1 - Better requirement:**
```
Requirements:
- Use products, product_categories, orders_products tables
- Calculate total revenue per product (price × quantity across all orders)
- Rank products within each category by revenue
- Show only the top 3 products from each category
- Handle ties appropriately (products with same revenue should get same rank)
```
Let the student decide: DENSE_RANK vs RANK vs ROW_NUMBER, and how to filter results.

**Task 2 - Better requirement:**
```
Requirements:
- Calculate daily revenue totals
- Show cumulative revenue that resets each month
- Order by date ascending
```
Let the student decide: Window function vs self-join, which PARTITION BY columns, frame specification.

**Task 3 - Better requirement:**
```
Requirements:
- Segment users into 4 equal groups based on total transaction amount
- Assign quartile labels (1 = lowest 25%, 4 = highest 25%)
- Show transaction statistics for each user
```
Let the student discover NTILE or figure out an alternative approach.

**Student feedback:** "Your instructions/requirements are somewhat too direct - you ask me to use function X and give me code directly - as in 'Use NTILE(4) OVER (ORDER BY total_transaction_amount DESC) for quartile assignment' or 'Use window function with PARTITION BY year, month and ORDER BY date'. This approach does not allow me to think, and I'd like to also practice critical thinking, wondering which method/function to choose etc."

**Additional technical issue:** "Your instruction to use ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW does not make sense here" - this is the default frame when ORDER BY is present, so explicitly stating it is redundant and confusing.

**Key Learning:**
- Describe WHAT the output should be, not HOW to achieve it
- Let student choose the right tool/function
- Focus on business requirements and expected results
- Allow student to practice decision-making between equivalent approaches
- Don't include redundant or unnecessary syntax specifications

---

### Week 10, Day 4 - Task 2: Gap Detection on Dense Transaction Data

**Issue:** Asked for users with gaps >= 30 days between transactions, but the transactions table has extremely dense data — max gap between consecutive transactions per user is 1 day.

**Why it's bad:**
- The business question (find long inactivity periods) is completely unsupported by the data
- Student cannot demonstrate the concept meaningfully — only adaptation (reducing threshold to 1 day) is possible
- Wastes time on a task that produces no interesting results

**Tables to avoid for gap-detection tasks:**
- `transactions` — too frequent, gaps rarely exceed 1 day

**Better approach:**
- Use `orders` table for gap detection (less frequent, more realistic gaps)
- Or use `user_sessions_daily` with date gaps (already validated as having meaningful gaps)
- Always verify data density before designing gap-detection tasks
