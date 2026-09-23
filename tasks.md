# SQL Tasks — 2026-09-23 (Week 39, Day 3)

**Dataset:** job_db · crappy_data_db  
**Focus:** DISTINCT ON with dirty-data parsing · NTH_VALUE with explicit frame

---

## Task 1 — Highest-Paying Offer per Platform (DISTINCT ON + Salary Parsing)

**Difficulty: 4/5**

**Business question:**  
The `zarobki` column is text, e.g. `"14 400 - 17 600 PLN/month"` or `"Undisclosed Salary"`. For each platform, find the single offer with the highest MAXIMUM salary value in its range.

You'll need to extract a numeric value from `zarobki` before you can compare or order by it — think about which part of the string is usable (the upper bound of the range) and how to strip out spaces, currency text, and the sentinel value.

Exclude offers where `zarobki` has no usable numeric value.

**Expected output columns:**  
`platform_name, pozycja, zarobki, max_salary`

Order by `platform_name`.


WITH oferty_zarobki AS (
SELECT 
	*,
	replace(substring(zarobki FROM '[\-\–\—]\s*([\d\s]+?)\s*(?=[a-zA-Z]|$)'), ' ', '') AS druga_kwota
FROM job_db.oferty o
)
SELECT DISTINCT ON (platform_name)
	p.nazwa AS platform_name,
	o.pozycja,
	o.zarobki,
	o.druga_kwota AS max_salary
FROM oferty_zarobki o
JOIN job_db.platforma p ON o.platforma_id = p.id
WHERE o.zarobki IS NOT NULL AND o.druga_kwota IS NOT NULL
ORDER BY platform_name, max_salary DESC, zarobki


---

## Task 2 — Third-Largest Transaction per User (NTH_VALUE, Explicit Frame)

**Difficulty: 4/5**

**Business question:**  
For each user, find the amount of their third-largest transaction. Use `NTH_VALUE` with an explicit frame that makes the window function see the entire partition for every row (not just rows up to the current one).

Only include users with at least 3 transactions.

**Expected output columns:**  
`user_id, third_largest_amount`

One row per user. Order by `user_id`.



WITH users_thirds AS (
SELECT 
	user_id,
	NTH_VALUE(amount, 3) OVER (PARTITION BY user_id ORDER BY amount DESC) AS third_largest_amount
FROM crappy_data_db.transactions t
)
SELECT 
	user_id,
	third_largest_amount 
FROM users_thirds
WHERE third_largest_amount IS NOT NULL
GROUP BY user_id, third_largest_amount
ORDER BY user_id

excluding nulls autofilters users below 3 transactions





---

## Submission Instructions

Paste your queries below each task.
