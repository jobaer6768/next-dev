# Module 14 — PostgreSQL Fundamentals & Advanced Data Manipulation

> From "I can write SELECT * FROM users" to "I can read EXPLAIN ANALYZE and tune a query."

---

## Why this module matters

Mission 4 will have a multi-tenant SaaS with 30+ tables, hundreds of endpoints, and queries that look like the example at the end of this module. If you can't read, write, and reason about that SQL, the rest of the course breaks down.

This module is dense with patterns. Skim it once, then come back as a reference while you build.

---

## Learning objectives

- Use `ALTER` to evolve a schema in place.
- Write `SELECT` queries with aliases, ordering, and distinct.
- Filter with `WHERE`, logical operators, and comparisons.
- Use scalar and aggregate functions.
- Use `NULL` correctly (and the `IS NULL` / `IS NOT NULL` distinction).
- Combine conditions with `IN`, `BETWEEN`, `LIKE`, `ILIKE`.
- Paginate with `LIMIT` / `OFFSET`.
- Update with `UPDATE ... FROM`.

---

## 1. The starting schema

```sql
-- module-14-setup.sql
CREATE TABLE employees (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name       TEXT NOT NULL,
  dept       TEXT NOT NULL,
  salary     NUMERIC(10,2) NOT NULL,
  hired_at   DATE NOT NULL,
  manager_id UUID REFERENCES employees(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 2. `ALTER` — modifying tables

```sql
-- Add a column
ALTER TABLE employees ADD COLUMN email TEXT;

-- Add a NOT NULL with a default (Postgres 11+ is fast)
ALTER TABLE employees ADD COLUMN active BOOLEAN NOT NULL DEFAULT true;

-- Add a UNIQUE constraint
ALTER TABLE employees ADD CONSTRAINT employees_email_unique UNIQUE (email);

-- Change a column type
ALTER TABLE employees ALTER COLUMN salary TYPE BIGINT USING (salary * 100)::BIGINT;  -- dollars → cents

-- Rename a column
ALTER TABLE employees RENAME COLUMN dept TO department;

-- Add a CHECK
ALTER TABLE employees ADD CONSTRAINT salary_positive CHECK (salary >= 0);

-- Remove a constraint
ALTER TABLE employees DROP CONSTRAINT salary_positive;

-- Set/drop a default
ALTER TABLE employees ALTER COLUMN active SET DEFAULT false;
ALTER TABLE employees ALTER COLUMN active DROP DEFAULT;
```

### `ALTER` in a transaction

In production, always wrap DDL changes in a transaction:

```sql
BEGIN;
ALTER TABLE employees ADD COLUMN email TEXT;
-- verify
COMMIT;  -- or ROLLBACK
```

DDL in Postgres *is* transactional. If the migration is wrong, `ROLLBACK` reverts it cleanly.

---

## 3. `SELECT` fundamentals

```sql
SELECT * FROM employees;
SELECT id, name, salary FROM employees;
SELECT id, name, salary * 12 AS annual_salary FROM employees;
```

### Aliases

```sql
SELECT
  e.name            AS employee,
  e.salary          AS monthly,
  e.salary * 12     AS annual,
  d.name            AS department_name      -- d is an alias for departments
FROM employees e
JOIN departments d ON d.id = e.department_id;
```

### `DISTINCT`

```sql
SELECT DISTINCT department FROM employees;
SELECT DISTINCT ON (department) department, name
  FROM employees
 ORDER BY department, hired_at DESC;     -- one row per department, newest hire
```

### `ORDER BY`

```sql
SELECT id, name, hired_at FROM employees ORDER BY hired_at DESC;
SELECT id, name FROM employees ORDER BY department ASC, salary DESC;
```

You can order by expressions and by column position (avoid column numbers in production code).

---

## 4. `WHERE` — filtering rows

```sql
SELECT * FROM employees WHERE department = 'engineering';
SELECT * FROM employees WHERE salary > 100000;
SELECT * FROM employees WHERE department = 'engineering' AND active = true;
SELECT * FROM employees WHERE department = 'engineering' OR department = 'design';
```

### Comparison operators

| Op | Meaning |
|----|---------|
| `=` | equal |
| `<>` or `!=` | not equal |
| `<`, `>`, `<=`, `>=` | ordering |
| `IS NULL` / `IS NOT NULL` | null check |
| `BETWEEN a AND b` | inclusive range |
| `IN (a, b, c)` | membership |
| `LIKE` / `ILIKE` | pattern match (case-sensitive / insensitive) |
| `~` / `~*` | regex (case-sensitive / insensitive) |

### `NULL` is not a value — it's the *absence* of a value

```sql
SELECT * FROM employees WHERE manager_id = NULL;   -- always returns 0 rows
SELECT * FROM employees WHERE manager_id IS NULL;   -- correct
```

This is the #1 source of "why isn't my filter working" in SQL.

### Three-valued logic

A boolean expression in SQL is `TRUE`, `FALSE`, or `UNKNOWN`. `UNKNOWN` is the result of any comparison with `NULL`. `WHERE` keeps only rows where the predicate is `TRUE`. This is why `column = NULL` is always false.

The `NULL`-coalescing operator:

```sql
COALESCE(manager_id, '00000000-0000-0000-0000-000000000000')
COALESCE(salary, 0)
COALESCE(NULLIF(a, b), 'default')   -- NULLIF returns NULL if a = b
```

---

## 5. Logical operators

```sql
WHERE department = 'engineering' AND (salary > 100000 OR hired_at > '2024-01-01')
WHERE NOT (department = 'engineering')
WHERE department IN ('engineering', 'design', 'product')
WHERE salary BETWEEN 80000 AND 120000
WHERE name LIKE 'A%'                    -- starts with A
WHERE name LIKE '%son%'                 -- contains 'son'
WHERE name ILIKE 'a%'                   -- case-insensitive
WHERE email ~* '^[a-z0-9._%+-]+@'       -- regex
```

`LIKE` patterns:
- `%` — any string
- `_` — any single character
- `\%` / `\_` — escaped literal

For full-text search, use `tsvector` (Module 16) — `ILIKE '%term%'` doesn't use indexes.

---

## 6. Scalar functions

```sql
SELECT
  upper(name),
  lower(email),
  length(name),
  trim(name),
  substring(name from 1 for 3),
  now(),
  current_date,
  date_trunc('month', hired_at),
  age(now(), hired_at),
  extract(year FROM hired_at) AS hire_year
FROM employees;
```

---

## 7. Aggregate functions

```sql
SELECT COUNT(*) FROM employees;
SELECT COUNT(DISTINCT department) FROM employees;
SELECT AVG(salary), MIN(salary), MAX(salary), SUM(salary) FROM employees;
```

`COUNT(*)` counts rows. `COUNT(column)` counts non-null values of `column`. They are different.

### `GROUP BY` (preview — full treatment in Module 15)

```sql
SELECT department, COUNT(*) AS headcount, AVG(salary) AS avg_salary
  FROM employees
 GROUP BY department
 ORDER BY headcount DESC;
```

`GROUP BY` collapses rows that share the same value in the grouped column. Every selected column must be either grouped or aggregated.

---

## 8. `LIMIT` and `OFFSET` — pagination

```sql
SELECT * FROM employees ORDER BY hired_at DESC LIMIT 20 OFFSET 0;   -- page 1
SELECT * FROM employees ORDER BY hired_at DESC LIMIT 20 OFFSET 20;  -- page 2
```

`OFFSET` is fine for small pages but degrades for deep pagination. For large datasets, use **cursor pagination** with `WHERE (hired_at, id) < ($cursor_hired_at, $cursor_id)` (Module 16).

---

## 9. `UPDATE` — modifying rows

```sql
UPDATE employees SET salary = salary * 1.05 WHERE department = 'engineering';
UPDATE employees SET active = false WHERE hired_at < '2020-01-01';
```

### `UPDATE ... FROM`

Update one table based on another:

```sql
UPDATE employees e
   SET salary = s.new_salary
  FROM salary_bumps s
 WHERE e.id = s.employee_id;
```

### `UPDATE` with a `RETURNING` clause

```sql
UPDATE employees SET salary = salary * 1.10
 WHERE id = '...'
 RETURNING id, name, salary;
```

The same trick as `INSERT ... RETURNING`. Use it from your Node code to fetch the updated row in one round-trip.

---

## 10. The realistic query — a SaaS "my open tasks" endpoint

```sql
SELECT
  t.id,
  t.title,
  t.status,
  t.priority,
  t.due_at,
  p.id   AS project_id,
  p.name AS project_name,
  w.id   AS workspace_id,
  w.name AS workspace_name,
  a.id   AS assignee_id,
  a.name AS assignee_name
FROM tasks t
JOIN projects p      ON p.id = t.project_id
JOIN workspaces w    ON w.id = p.workspace_id
LEFT JOIN users a    ON a.id = t.assignee_id
WHERE t.assignee_id = $1
  AND t.status NOT IN ('done', 'archived')
  AND w.id = ANY($2::uuid[])
ORDER BY
  CASE t.priority
    WHEN 'high'   THEN 1
    WHEN 'medium' THEN 2
    WHEN 'low'    THEN 3
  END,
  t.due_at NULLS LAST
LIMIT 50;
```

This single query:

- Joins four tables.
- Filters by assignee and a list of workspaces (the user's memberships).
- Excludes terminal statuses.
- Sorts with a custom priority order, then by due date, with `NULLS LAST` so tasks without a due date don't bubble to the top.
- Caps the result for the UI.

This is the shape of every "list" endpoint in Mission 4.

---

## 11. The realistic update — bulk status change

```sql
UPDATE tasks
   SET status = 'archived',
       updated_at = now()
 WHERE project_id = $1
   AND status = 'done'
   AND updated_at < now() - interval '30 days'
RETURNING id;
```

Returns the IDs of everything that was archived.

---

## 12. Common mistakes

1. **`= NULL`.** Always `IS NULL`.
2. **`COUNT(column)`** when you meant `COUNT(*)`.
3. **`SELECT *`** in production code. Always name the columns you need.
4. **Updates without `WHERE`.** Always have a `WHERE` clause. Add a test that asserts the count.
5. **Trusting the application to enforce uniqueness.** Use a `UNIQUE` constraint; let the DB be the source of truth.
6. **`OFFSET 100000`** for deep pagination. Use keyset/cursor pagination.

---

## Exercise

Use the library case study from Module 13. Write SQL for:

1. List the top 5 most-borrowed books this month.
2. Find members who have overdue loans (due date passed, not returned).
3. Update a single book: add a `genre` and a `total_copies` of 5.
4. Increase the salary of every author who has at least 3 books by 10%. (Add a `salary` column to `authors` for the exercise.)
5. Use `EXPLAIN ANALYZE` on query 1; identify the slowest step. Add an index to make it faster.

---

## AI Industry Note (2026)

- **AI assistants are good at SQL but bad at the `NULL` gotchas.** When prompting an AI to write SQL, *always* ask: *"Avoid using `= NULL`. Use `IS NULL` / `IS NOT NULL` exclusively."* Otherwise you will get back queries that silently return zero rows.
- **Postgres 16+ has `MERGE`, `RETURNING` in DML, and many other features that AI assistants trained on older data may not use.** When in doubt, write the SQL yourself and ask the AI to *review* it, not generate it from scratch.
- **The senior-engineer SQL test in 2026:** given a one-sentence business question, write a single SQL statement that answers it correctly with proper JOINs, NULL handling, and a reasonable index strategy. This module is your training ground. Modules 15 and 16 add the harder patterns (joins, aggregations, CTEs, subqueries, indexes).
