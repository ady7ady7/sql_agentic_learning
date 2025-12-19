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
