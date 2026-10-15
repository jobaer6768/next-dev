# Module 15 — Dates, Relationships & Understanding JOIN

> The two skills that turn SQL from "list of rows" into "answers to business questions."

---

## Why this module matters

Most production SQL is **joins and aggregations**. A list endpoint joins 3-4 tables. A dashboard endpoint does a `GROUP BY` across them. A monthly report combines the two.

If you can't read a `LEFT JOIN` in 10 seconds or write a `GROUP BY` with a `HAVING` clause, you will spend Mission 4 staring at Stack Overflow.

---

## Learning objectives

- Use Postgres date functions.
- Use `GROUP BY` and `HAVING` correctly.
- Model 1:N and M:N with foreign keys.
- Use all four JOIN types: `INNER`, `LEFT`, `RIGHT`, `FULL`.
- Use `CROSS` and `NATURAL` joins (and know when not to).
- Read query execution plans.

---

## 1. Date functions

```sql
SELECT now();                                     -- current timestamp
SELECT current_date;                              -- date only
SELECT now() + interval '7 days';
SELECT now() - interval '1 month';
SELECT date_trunc('month', now());                -- 2026-10-01 00:00:00
SELECT date_trunc('day', hired_at);
SELECT extract(year FROM hired_at);
SELECT extract(month FROM hired_at);
SELECT age(now(), hired_at);                      -- interval
SELECT to_char(now(), 'YYYY-MM-DD');              -- formatted text
SELECT to_char(now(), 'YYYY-MM');                 -- for month buckets
```

### Common date patterns

```sql
-- "Today"
WHERE created_at >= current_date
  AND created_at <  current_date + 1

-- "This month"
WHERE created_at >= date_trunc('month', now())
  AND created_at <  date_trunc('month', now()) + interval '1 month'

-- "Last 7 days"
WHERE created_at >= now() - interval '7 days'

-- "Last 30 days, inclusive of today"
WHERE created_at >= now() - interval '30 days'
  AND created_at <  now() + interval '1 day'
```

---

## 2. `GROUP BY` and `HAVING`

`GROUP BY` collapses rows that share the same grouping column. `HAVING` filters *after* aggregation (whereas `WHERE` filters *before*).

```sql
SELECT department, COUNT(*) AS headcount, AVG(salary) AS avg_salary
  FROM employees
 WHERE active = true                  -- pre-aggregation filter
 GROUP BY department
HAVING COUNT(*) > 5                   -- post-aggregation filter
 ORDER BY headcount DESC;
```

You can group by multiple columns:

```sql
SELECT department, date_trunc('month', hired_at) AS month, COUNT(*)
  FROM employees
 GROUP BY department, month
 ORDER BY department, month;
```

`GROUPING SETS`, `ROLLUP`, and `CUBE` produce multiple groupings in one query:

```sql
SELECT department, date_trunc('year', hired_at) AS year, COUNT(*)
  FROM employees
 GROUP BY ROLLUP (department, year);
```

For most apps, plain `GROUP BY` is enough.

---

## 3. Modeling relationships with foreign keys

We've seen this in earlier modules; here's the canonical library example again:

```sql
CREATE TABLE authors (
  id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL
);

CREATE TABLE books (
  id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title   TEXT NOT NULL,
  author_id UUID NOT NULL REFERENCES authors(id) ON DELETE CASCADE
);

CREATE TABLE members (
  id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name  TEXT NOT NULL
);

CREATE TABLE loans (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  book_id     UUID NOT NULL REFERENCES books(id)    ON DELETE RESTRICT,
  member_id   UUID NOT NULL REFERENCES members(id)  ON DELETE CASCADE,
  loaned_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  due_at      TIMESTAMPTZ NOT NULL,
  returned_at TIMESTAMPTZ
);
```

`ON DELETE` options recap:
- `CASCADE` — delete children when parent is deleted.
- `RESTRICT` — block parent delete if children exist.
- `SET NULL` — nullify the FK.
- `NO ACTION` — like `RESTRICT` but checked at end of statement.

---

## 4. `INNER JOIN` — the default

```sql
SELECT b.title, a.name
  FROM books b
  INNER JOIN authors a ON a.id = b.author_id;
```

Returns only rows that have matches in both tables.

Aliases (`b`, `a`) are essential when you join 3+ tables. Be consistent.

---

## 5. `LEFT JOIN` — keep all left rows

```sql
SELECT b.title, a.name
  FROM books b
  LEFT JOIN authors a ON a.id = b.author_id;
```

Returns every book, even if it has no author (the `a.name` will be `NULL`).

`LEFT JOIN` is the most common join in production code. **"Show me all the X, and the related Y if there is one."**

To find "X with no Y":

```sql
SELECT b.*
  FROM books b
  LEFT JOIN loans l ON l.book_id = b.id
 WHERE l.id IS NULL;                     -- never loaned
```

---

## 6. `RIGHT JOIN` — keep all right rows

```sql
SELECT b.title, a.name
  FROM books b
 RIGHT JOIN authors a ON a.id = b.author_id;
```

`RIGHT JOIN` is rare. Most teams prefer to swap the order and use `LEFT JOIN` for consistency. Pick a style; be consistent.

---

## 7. `FULL OUTER JOIN` — keep everything

```sql
SELECT a.name AS author, b.title AS book
  FROM authors a
  FULL OUTER JOIN books b ON b.author_id = a.id;
```

Returns all authors (even without books) and all books (even without authors). Useful for diffs and reconciliation.

---

## 8. `CROSS JOIN` — every combination

```sql
SELECT d.name AS department, p.name AS project
  FROM departments d
 CROSS JOIN projects p;
```

Cartesian product. Used for:
- Generating a "skeleton" of dates or categories for charts.
- Filling in zeros for missing buckets in a report.

Almost never used in CRUD code.

---

## 9. `NATURAL JOIN` — never

```sql
SELECT * FROM books NATURAL JOIN authors;
```

Joins on every column with the same name. Brittle. Don't use it in production code.

---

## 10. The realistic query — "Top books by loan count this month"

```sql
SELECT
  b.id,
  b.title,
  a.name AS author,
  COUNT(l.id) AS loan_count
FROM books b
JOIN authors a     ON a.id = b.author_id
LEFT JOIN loans l  ON l.book_id = b.id
                  AND l.loaned_at >= date_trunc('month', now())
GROUP BY b.id, b.title, a.name
ORDER BY loan_count DESC, b.title ASC
LIMIT 10;
```

Notice the `LEFT JOIN` with a date predicate in the `ON` clause, not the `WHERE`. This is how you keep books that have zero loans *this month* but had loans in the past.

---

## 11. The realistic aggregation — "Daily signups for the last 30 days, including zero days"

```sql
WITH days AS (
  SELECT generate_series(
    (current_date - interval '29 days')::date,
    current_date,
    '1 day'::interval
  )::date AS day
)
SELECT
  d.day,
  COUNT(u.id) AS signups
FROM days d
LEFT JOIN users u
  ON date_trunc('day', u.created_at)::date = d.day
GROUP BY d.day
ORDER BY d.day;
```

This is the canonical "fill in the missing days" pattern. `generate_series` is your friend.

---

## 12. Reading `EXPLAIN ANALYZE`

```sql
EXPLAIN ANALYZE
SELECT b.title, COUNT(l.id)
  FROM books b
  LEFT JOIN loans l ON l.book_id = b.id
 GROUP BY b.id;
```

Output (simplified):

```
HashAggregate  (cost=20.50..22.50 rows=200 width=40) (actual time=0.5..0.6 rows=10 loops=1)
  Group Key: b.id
  ->  Hash Left Join  (cost=10.00..18.00 rows=200 width=36) (actual time=0.2..0.4 rows=200 loops=1)
        Hash Cond: (l.book_id = b.id)
        ->  Seq Scan on books b  (cost=0.00..3.00 rows=100 width=32) (actual time=0.01..0.01 rows=10 loops=1)
        ->  Hash  (cost=8.00..8.00 rows=200 width=20) (actual time=0.1..0.1 rows=200 loops=1)
              ->  Seq Scan on loans l  (cost=0.00..8.00 rows=200 width=20) (actual time=0.05..0.08 rows=200 loops=1)
```

What to look for:

- **Seq Scan** — full table scan. Fine for small tables; bad for large ones.
- **Index Scan** / **Index Only Scan** — using an index. Good.
- **Nested Loop**, **Hash Join**, **Merge Join** — how two inputs are combined.
- **actual time** — how long it actually took (in ms).
- **rows** — estimated vs actual. Big difference = stale statistics (`ANALYZE`).

`EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` gives you even more. Use it whenever a query is slow.

---

## 13. Common mistakes

1. **`JOIN` with the predicate in `WHERE` instead of `ON`.** For `LEFT JOIN`, the result differs — predicates in `WHERE` filter after the join (turning it into an inner join).
2. **Aggregating without `GROUP BY` for non-aggregated columns.** Postgres errors; many other DBs do something surprising.
3. **Counting `*` when you wanted distinct users.** `COUNT(DISTINCT user_id)`.
4. **Forgetting `date_trunc`.** Comparing `created_at` to `'2026-01-01'` without a range will miss records from that day at any time other than midnight.

---

## Exercise

Using the library case study, write SQL for:

1. List all members and the books they currently have on loan (use `LEFT JOIN` for members with no loans).
2. For each author, the number of books they have written (use `GROUP BY`).
3. For each month of 2025, the total number of loans created (use `date_trunc` and `generate_series`).
4. The 3 members with the most overdue loans.
5. The books that have *never* been loaned.
6. Run `EXPLAIN ANALYZE` on each. Add an index to make at least one of them faster.

---

## AI Industry Note (2026)

- **AI assistants write SQL joins wrong in two predictable ways:** (a) putting `LEFT JOIN` predicates in `WHERE`, silently turning the join into inner; (b) using the wrong join direction (they reach for `INNER JOIN` when they should be using `LEFT JOIN`). Always review the join logic by hand.
- **The `EXPLAIN ANALYZE` skill is senior-level and AI doesn't have it.** When you ask an AI to "optimize this query," it will suggest indexes without running the query. The senior move is to read the plan yourself and form a hypothesis before adding any index. The query plan is a thinking tool, not a checkmark.
- **What to read next:** the Postgres [`EXPLAIN` docs](https://www.postgresql.org/docs/current/using-explain.html). The page is dense but it is the single biggest leverage point in your SQL career. Read it once a year.
