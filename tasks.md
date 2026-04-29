# Daily SQL Practice Tasks

**Generated:** 2026-04-29
**Week 20, Days 2+3 Focus:** FIRST_VALUE + custom window frames + anti-join NULL trap + LAG offset + YoY comparison + NULLIF in real division

---

## DAY 2

---

## Task 1: FIRST_VALUE — Most Recent Transaction per User

**Scenario:**
The finance team wants to see each transaction alongside the user's most recent transaction amount — for comparison purposes.

For every transaction, show:
- `transaction_id`
- `user_id`
- `amount`
- `created_at`
- `latest_amount` — the amount of the most recent transaction for that user (including the current row if it's the latest)

**Tables:** `crappy_data_db.transactions`

**Requirements:**
- Use `FIRST_VALUE(amount ORDER BY created_at DESC)` — not LAST_VALUE
- Exclude NULL amounts
- Order by `user_id ASC`, `created_at ASC`

**Difficulty Rating:** 3/5


SELECT
	id AS TRANSACTION_ID,
	user_id,
	amount,
	created_at,
	FIRST_VALUE(amount) OVER (PARTITION BY user_id ORDER BY created_at DESC) AS latest_amount
FROM crappy_data_db.transactions

2/10 difficulty

---

## Task 2: Cumulative SUM with Custom Frame — Rolling 3-Order Revenue

**Scenario:**
The finance team wants a running total of order revenue per user, but only looking at the current order and the 2 preceding ones — a rolling 3-order window.

For every order, show:
- `order_id`
- `user_id`
- `amount`
- `rolling_3_revenue` — sum of `amount` over the current row and 2 preceding rows within the user's order history (ordered by `created_at ASC`)

**Tables:** `crappy_data_db.orders`

**Requirements:**
- Use `SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)`
- Exclude NULL amounts
- Order by `user_id ASC`, `created_at ASC`

**Difficulty Rating:** 4/5

SELECT
	id AS order_id,
	user_id,
	amount,
	SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS rolling_3_revenue
FROM crappy_data_db.orders

The ROWS BETWEEN X PRECEDING AND CURRENT ROW it's totally worth remembering. Other than that, pretty easy.


---

## Task 3: Anti-Join NULL Trap — Products Never Ordered

**Scenario:**
The inventory team wants to find products that have never appeared in any order.

**Your tasks:**
1. Write the query using `LEFT JOIN ... WHERE IS NULL`
2. Write it again using `NOT EXISTS`
3. Write the `NOT IN` version — then in a comment explain: what happens to `NOT IN` if `orders_products.product_id` contained even one NULL, and why does it silently return 0 rows?

**Expected Output Columns:** `product_id`, `name`

**Tables:** `crappy_data_db.products`, `crappy_data_db.orders_products`

**Order by:** `product_id ASC`

**Difficulty Rating:** 5/5

THERE ARE NO SUCH PRODUCTS

SELECT * FROM crappy_data_db.orders_products op
LEFT JOIN crappy_data_db.products p ON op.product_id = p.id
WHERE p.id IS NULL


SELECT * FROM crappy_data_db.orders_products op
WHERE NOT EXISTS
(SELECT p.id FROM crappy_data_db.products p
WHERE op.product_id = p.id
AND p.id IS NOT NULL
)

SELECT * FROM crappy_data_db.orders_products op
WHERE OP.product_id NOT IN 
(SELECT p.id FROM crappy_data_db.products p
WHERE op.product_id = p.id
AND p.id IS NOT NULL
)


It doesn't matter if we add the NOT NULL condition honestly. I'd like us to settle ON ONE SOLUTION - maybe NOT EXISTS from now on, as it's pointless and confusing, and I sometimes confuse one iwth another as we mingle these three.




---

## DAY 3

---

## Task 4: LAG with Offset — Compare Order to 3 Orders Prior

**Scenario:**
The analytics team wants to see how each order's amount compares to the order placed 3 orders ago by the same user.

For every order, show:
- `order_id`
- `user_id`
- `amount`
- `amount_3_prior` — amount from 3 orders ago for this user (NULL if fewer than 3 prior orders exist)
- `diff_from_3_prior` — `amount - amount_3_prior` (NULL if no prior)

**Tables:** `crappy_data_db.orders`

**Requirements:**
- Use `LAG(amount, 3)` partitioned by user, ordered by `created_at ASC`
- Exclude NULL amounts
- Order by `user_id ASC`, `created_at ASC`

**Difficulty Rating:** 3/5

SELECT 
	*,
	LAG(amount, 3) OVER (PARTITION BY user_id ORDER BY created_at) AS amount_3_prior,
	amount - LAG(amount, 3) OVER (PARTITION BY user_id ORDER BY created_at) AS diff_from_3_prior
FROM crappy_data_db.orders o

Again, pretty easy once you know the trick with LAG(x, OFFSET). I could use abs() here, but we didn't compare anything anyway, so there was no point in doing it.



---

## Task 5: YoY — Monthly Transaction Revenue Comparison

**Scenario:**
The finance team wants a month-by-month transaction revenue table showing the same month's revenue from the prior year for direct comparison.

For each month, show:
- `month` (DATE_TRUNC to month)
- `revenue` — total transaction amount for this month
- `prev_year_revenue` — revenue for the same calendar month one year prior (NULL if no data)
- `yoy_pct_change` — percentage change vs prior year, rounded to 1 decimal, NULL if no prior year. Formula: `(revenue - prev_year_revenue) / prev_year_revenue * 100`

**Tables:** `crappy_data_db.transactions`

**Requirements:**
- Exclude NULL amounts
- Use `LAG(revenue, 12) OVER (ORDER BY month ASC)` for prior year
- Use `NULLIF` to avoid division by zero in the percentage calculation
- Only include months with at least 1 transaction
- Order by `month ASC`

**Difficulty Rating:** 4/5

WITH orders_months AS (
SELECT 
	*,
	DATE_TRUNC('Month', created_at) AS month_
FROM crappy_data_db.transactions t
),
monthly_revs AS (
SELECT
	month_,
	SUM(amount) AS monthly_revenue
FROM orders_months
GROUP BY month_
)
SELECT 
	*,
	LAG(monthly_revenue, 12) OVER (ORDER BY month_) AS prev_year_revenue,
	ROUND((monthly_revenue - LAG(monthly_revenue, 12) OVER (ORDER BY month_)) / LAG(monthly_revenue, 12) OVER (ORDER BY month_) * 100, 1) AS yoy_pct_change
FROM monthly_revs

---

## Task 6: NULLIF + COALESCE — Cleaning Dirty Aggregations

**Scenario:**
The analytics team is building a user quality report from `crappy_data_db.users`. Some users have empty string `''` in their `city` field instead of NULL, which skews city-based counts.

For each country, show:
- `country`
- `total_users` — total number of users in that country
- `users_with_city` — count of users who have a real city (not NULL and not empty string `''`)
- `pct_with_city` — percentage of users with a real city, rounded to 1 decimal. Use `NULLIF` on the denominator to avoid division by zero, and `COALESCE` to show `0.0` instead of NULL for countries with no users having a city.

**Tables:** `crappy_data_db.users`

**Requirements:**
- Use `COUNT(NULLIF(city, ''))` to exclude empty strings from the city count
- Use `NULLIF(total_users, 0)` in the division
- Use `COALESCE(..., 0.0)` to replace NULL pct with 0.0
- Exclude NULL countries
- Order by `total_users DESC`

**Difficulty Rating:** 5/5

WITH USERS_COUNTERS AS (
SELECT 
	country,
	COUNT(*) AS total_users,
	COUNT(NULLIF(city, '')) AS users_with_city
FROM crappy_data_db.users u
WHERE COUNTRY IS NOT null
GROUP BY country
)
SELECT 
	country,
	total_users,
	users_with_city,
	round(users_with_city / total_users::NUMERIC, 3) * 100 AS pct_with_city
FROM users_counters
ORDER BY total_users DESC


Frankly it's a useless task for this db, since all countries have 100% users with cities. There's no such issue.

---

## Submission Instructions

**Day 2:**
1. Task 1 — FIRST_VALUE latest transaction amount per user (3/5)
2. Task 2 — Rolling 3-order SUM with custom frame (4/5)
3. Task 3 — Anti-join three ways + NOT IN NULL trap explanation (5/5)

**Day 3:**
4. Task 4 — LAG(amount, 3) offset comparison (3/5)
5. Task 5 — YoY monthly transaction revenue with NULLIF division (4/5)
6. Task 6 — NULLIF + COALESCE dirty city data cleanup (5/5)
