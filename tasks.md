# Daily SQL Practice Tasks

**Generated:** 2026-05-15
**Week 22, Day 5 Focus:** Light Friday — simple GROUP BY + LAG + NOT EXISTS refresh

---

## Task 1: Offer Count by Seniority and Work Type

**Scenario:**
Quick breakdown — how many offers exist for each seniority + work type combination?

Show:
- `seniority` — nazwa from seniority
- `typ`
- `offer_count`

Exclude NULL seniority_id and NULL typ. Order by `seniority ASC`, `offer_count DESC`.

**Tables:** `job_db.oferty`, `job_db.seniority`

**Difficulty Rating:** 2/5

SELECT 
	s.nazwa,
	o.typ,
	COUNT(*) AS offer_count
FROM job_db.oferty o
JOIN job_db.seniority s ON o.seniority_id = s.id
WHERE s.nazwa IS NOT NULL AND o.typ IS NOT NULL
GROUP BY s.nazwa, o.typ 
ORDER BY offer_count DESC



---

## Task 2: LAG — Each Transaction With Previous Amount

**Scenario:**
For every transaction, show the previous transaction amount for the same user.

Show:
- `transaction_id`
- `user_id`
- `amount`
- `prev_amount` — amount from the previous transaction for that user, ordered by `created_at ASC` (NULL if first)

**Tables:** `crappy_data_db.transactions`

**Requirements:**
- Exclude NULL amounts
- Order by `user_id ASC`, `created_at ASC`

**Difficulty Rating:** 3/5

SELECT 
	id AS transaction_id,
	user_id,
	amount,
	LAG(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_amount
FROM crappy_data_db.transactions t 

It auto orders itself due to the window func

---

## Task 3: NOT EXISTS — Seniority Levels With No 2025 Listings

**Scenario:**
Which seniority levels had no job offers listed in 2025?

Use `NOT EXISTS`. Start from `job_db.seniority`, check against `job_db.oferty`.

**Expected Output Columns:** `seniority_id`, `seniority`

**Tables:** `job_db.seniority`, `job_db.oferty`

**Order by:** `seniority_id ASC`

**Difficulty Rating:** 3/5

SELECT * FROM job_db.seniority s
WHERE NOT EXISTS
(SELECT o.seniority_id FROM job_db.oferty o
WHERE o.seniority_id IS NOT NULL AND o.seniority_id = s.id
AND EXTRACT('Year' FROM o.data_wystawienia) = '2025'
)



There were a few cases surprisingly


---

## Submission Instructions

1. Task 1 — Offer count by seniority + work type (2/5)
2. Task 2 — LAG previous transaction amount per user (3/5)
3. Task 3 — NOT EXISTS: seniority with no 2025 listings (3/5)
