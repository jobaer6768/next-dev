# Module 16 — Advanced Query Techniques and Database Objects

> Subqueries, CTEs, views, functions, stored procedures, triggers, and indexes.

---

## Why this module matters

For a CRUD app, you can survive on the patterns of Modules 13–15. For a multi-tenant SaaS with dashboards, exports, and AI features, you will need:

- **CTEs** to organize complex queries.
- **Views** to hide query complexity from the app.
- **Functions** for reusable business logic.
- **Triggers** for audit logs and denormalized counters.
- **Indexes** to make all of the above fast.

This module gives you the toolkit. We will use it heavily in Missions 4 and 6.

---

## Learning objectives

- Write and read correlated and non-correlated subqueries.
- Use `WITH` (CTEs) to compose complex queries.
- Create and use views, materialized views, and updatable views.
- Define SQL functions and stored procedures.
- Implement triggers for audit and derived data.
- Use `EXPLAIN` to choose the right index.

---

## 1. Subqueries — three patterns

### Pattern A: scalar subquery in `SELECT`

```sql
SELECT
  b.id,
  b.title,
  (SELECT COUNT(*) FROM loans l WHERE l.book_id = b.id) AS loan_count
FROM books b;
```

A scalar subquery returns one value per outer row. Useful for denormalized counts in a list.

### Pattern B: subquery in `WHERE`

```sql
SELECT * FROM members m
 WHERE m.id IN (SELECT member_id FROM loans WHERE returned_at IS NULL);
```

The subquery returns a set; the outer query filters by membership.

### Pattern C: subquery in `FROM` (derived table)

```sql
SELECT avg_per_member.total_loans
  FROM (SELECT member_id, COUNT(*) AS total_loans FROM loans GROUP BY member_id) AS avg_per_member
 WHERE avg_per_member.total_loans > 5;
```

Useful when you want to aggregate the aggregation.

### Correlated vs non-correlated

- **Non-correlated** — the subquery can run independently. Fast.
- **Correlated** — the subquery references the outer query. Runs once per outer row. Often slow; rewrite as a `JOIN` if possible.

---

## 2. `EXISTS` — the right way to ask "is there any?"

```sql
SELECT m.*
  FROM members m
 WHERE EXISTS (SELECT 1 FROM loans l WHERE l.member_id = m.id AND l.returned_at IS NULL);
```

`EXISTS` short-circuits on the first match. Faster than `IN` for large sets. Use it.

---

## 3. CTEs — `WITH`

A Common Table Expression is a named subquery you can reference in the main query.

```sql
WITH
  active_members AS (
    SELECT id, name FROM members WHERE status = 'active'
  ),
  recent_loans AS (
    SELECT member_id, COUNT(*) AS n
      FROM loans
     WHERE loaned_at >= now() - interval '30 days'
     GROUP BY member_id
  )
SELECT am.name, COALESCE(rl.n, 0) AS loans_last_30
  FROM active_members am
  LEFT JOIN recent_loans rl ON rl.member_id = am.id
 ORDER BY loans_last_30 DESC;
```

CTEs make complex queries **readable**. Use them aggressively.

### Recursive CTEs

```sql
WITH RECURSIVE subordinates AS (
  SELECT id, name, manager_id FROM employees WHERE id = 'top-boss-id'
  UNION ALL
  SELECT e.id, e.name, e.manager_id
    FROM employees e
    JOIN subordinates s ON e.manager_id = s.id
)
SELECT * FROM subordinates;
```

Recursive CTEs are how you walk trees (org charts, comment threads, category hierarchies). Module 49 will use a recursive CTE in pgvector's HNSW index metadata.

---

## 4. Views

```sql
CREATE VIEW v_active_members AS
SELECT id, name, email
  FROM members
 WHERE status = 'active';
```

A view is a stored query. Treat it like a table:

```sql
SELECT * FROM v_active_members WHERE name ILIKE 'r%';
```

### Updatable views

A simple view (one table, no aggregates) is automatically updatable:

```sql
UPDATE v_active_members SET name = 'Rumi A.' WHERE id = '...';
```

### Materialized views — when read perf matters

```sql
CREATE MATERIALIZED VIEW mv_daily_signups AS
SELECT date_trunc('day', created_at)::date AS day, COUNT(*) AS n
  FROM users
 GROUP BY 1
 ORDER BY 1;

CREATE UNIQUE INDEX ON mv_daily_signups (day);
```

Materialized views store the result. They're fast but **stale** — refresh them:

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_signups;  -- needs UNIQUE index
```

Use for: dashboards, reports, anything read-heavy that doesn't need second-by-second freshness.

---

## 5. Functions

```sql
CREATE OR REPLACE FUNCTION member_active_loans(p_member UUID)
RETURNS BIGINT
LANGUAGE SQL
STABLE
AS $$
  SELECT COUNT(*) FROM loans WHERE member_id = p_member AND returned_at IS NULL;
$$;

SELECT member_active_loans('00000000-0000-0000-0000-000000000000');
```

### PL/pgSQL — for procedural logic

```sql
CREATE OR REPLACE FUNCTION mark_overdue()
RETURNS INTEGER
LANGUAGE plpgsql
AS $$
DECLARE
  affected INTEGER;
BEGIN
  UPDATE loans SET status = 'overdue'
   WHERE returned_at IS NULL
     AND due_at < now();
  GET DIAGNOSTICS affected = ROW_COUNT;
  RETURN affected;
END;
$$;
```

Use PL/pgSQL when you need:
- Loops.
- Conditionals.
- Multiple statements.
- Error handling (`RAISE EXCEPTION`).

---

## 6. Stored procedures

Postgres 11+ has stored procedures with transaction control (functions couldn't `COMMIT`).

```sql
CREATE OR REPLACE PROCEDURE archive_old_loans(cutoff DATE)
LANGUAGE plpgsql
AS $$
BEGIN
  INSERT INTO loan_archive
    SELECT * FROM loans WHERE returned_at < cutoff;
  DELETE FROM loans WHERE returned_at < cutoff;
  COMMIT;
END;
$$;

CALL archive_old_loans('2024-01-01');
```

Use sparingly. **Most business logic belongs in your application**, not the database. Use procedures for: nightly batch jobs, integrity-critical multi-statement operations.

---

## 7. Triggers

A trigger is a function that the database calls automatically when an event happens.

```sql
-- 1. The function
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$;

-- 2. The trigger
CREATE TRIGGER trg_users_updated_at
BEFORE UPDATE ON users
FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

Now every `UPDATE users SET ...` automatically refreshes `updated_at`.

### Audit log trigger

```sql
CREATE TABLE audit_log (
  id          BIGSERIAL PRIMARY KEY,
  table_name  TEXT NOT NULL,
  row_id      UUID NOT NULL,
  operation   TEXT NOT NULL CHECK (operation IN ('INSERT','UPDATE','DELETE')),
  changed_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  changed_by  UUID,
  old_data    JSONB,
  new_data    JSONB
);

CREATE OR REPLACE FUNCTION audit_trigger()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    INSERT INTO audit_log (table_name, row_id, operation, new_data)
    VALUES (TG_TABLE_NAME, NEW.id, 'INSERT', to_jsonb(NEW));
    RETURN NEW;
  ELSIF TG_OP = 'UPDATE' THEN
    INSERT INTO audit_log (table_name, row_id, operation, old_data, new_data)
    VALUES (TG_TABLE_NAME, NEW.id, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW));
    RETURN NEW;
  ELSIF TG_OP = 'DELETE' THEN
    INSERT INTO audit_log (table_name, row_id, operation, old_data)
    VALUES (TG_TABLE_NAME, OLD.id, 'DELETE', to_jsonb(OLD));
    RETURN OLD;
  END IF;
END;
$$;

CREATE TRIGGER trg_tasks_audit
AFTER INSERT OR UPDATE OR DELETE ON tasks
FOR EACH ROW EXECUTE FUNCTION audit_trigger();
```

### Triggers: when to use, when to avoid

- **Use for:** `updated_at` timestamps, audit logs, denormalized counters, integrity constraints the schema can't express.
- **Avoid for:** business workflows ("when an order is paid, send a Slack message"). That's application logic; do it in code.

---

## 8. Indexes — the most important section

An index is a data structure that lets the database find rows without scanning the whole table. Default type: **B-tree** (good for equality and range).

```sql
-- Single column
CREATE INDEX users_email_idx ON users (email);

-- Composite
CREATE INDEX tasks_project_status_idx ON tasks (project_id, status);

-- Partial
CREATE INDEX tasks_open_idx ON tasks (project_id) WHERE status NOT IN ('done', 'archived');

-- Unique
CREATE UNIQUE INDEX users_email_unique ON users (email);

-- For text search
CREATE INDEX books_title_trgm ON books USING GIN (title gin_trgm_ops);
-- requires: CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- For JSONB
CREATE INDEX users_settings_brand_idx ON users USING GIN (settings jsonb_path_ops);
```

### The composite index rule

For a query `WHERE a = ? AND b = ? ORDER BY c`, the ideal index is `(a, b, c)`. The order matters.

### Partial indexes — the secret weapon

For "open tasks in a project," a partial index on `(project_id) WHERE status = 'open'` is *much* smaller and faster than a full index.

### When to add an index

- Any column you `WHERE` on frequently.
- Any column you `ORDER BY` on a lot (for small result sets).
- Any foreign key (Postgres doesn't add them by default; add them yourself).

### When *not* to add an index

- Small tables (under a few thousand rows — seq scan is fine).
- Columns with very low cardinality (e.g. a `boolean`).
- Tables with heavy write traffic where the index maintenance cost outweighs read benefit.

### Reading `EXPLAIN` to validate

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'rumi@x.com';
```

Before index: `Seq Scan on users`. After index: `Index Scan using users_email_idx`. The actual time drops dramatically.

---

## 9. Putting it together — a "task search" feature

The application wants: "Find tasks in any of my workspaces, status `open`, with title containing the search term, sort by due date, paginate by cursor."

```sql
WITH my_workspaces AS (
  SELECT workspace_id FROM memberships
   WHERE user_id = $1 AND status = 'active'
)
SELECT t.id, t.title, t.due_at, p.name AS project_name
  FROM tasks t
  JOIN projects p ON p.id = t.project_id
 WHERE p.workspace_id IN (SELECT workspace_id FROM my_workspaces)
   AND t.status = 'open'
   AND t.title ILIKE '%' || $2 || '%'
   AND ($3::timestamptz IS NULL OR (t.due_at, t.id) > ($3, $4))
 ORDER BY t.due_at NULLS LAST, t.id
 LIMIT 20;
```

With indexes:

```sql
CREATE INDEX tasks_project_status_due_idx ON tasks (project_id, status, due_at);
CREATE INDEX projects_workspace_idx        ON projects (workspace_id);
CREATE INDEX memberships_user_status_idx   ON memberships (user_id, status) WHERE status = 'active';
```

This is the kind of query the AI assistant can now generate for you, **because you have the indexes, the CTE shape, and the cursor pagination pattern as habits**.

---

## 10. Common mistakes

1. **Function with side effects called in a `SELECT`.** That function runs for every row and can wreck performance.
2. **Trigger that does network I/O** (Slack, email). It's slow, it can fail mid-transaction, and it ties DB availability to your notification provider. Move side effects to the app.
3. **Indexing every column.** Indexes cost writes and storage. Index what you query.
4. **Stale materialized view.** Set up a `pg_cron` or application-level refresh.
5. **Triggers that recurse.** A trigger that writes to the same table can loop forever. Postgres protects you with depth limits, but don't test them.

---

## Exercise

Using the library case study:

1. Write a CTE that lists the top 5 authors by total loans, including authors with zero loans.
2. Create a view `v_book_loan_history` that joins books, members, and loans for a single round-trip from the app.
3. Create a `set_updated_at` trigger on `loans` and verify it works.
4. Create an audit-log trigger for `loans`.
5. Create indexes on `(book_id, loaned_at)` and `(member_id, returned_at)`. Run `EXPLAIN ANALYZE` on a query that uses them and confirm the plan changes.
6. Write a stored procedure that, given a member ID, returns their currently overdue loans as a JSONB array.

---

## AI Industry Note (2026)

- **The "AI writes a query, you write the index" division of labor is the new normal.** AI assistants are good at producing SQL that answers a question. They are bad at predicting which indexes the query will need at scale. The senior move is to use the AI to draft, then read the `EXPLAIN` and add the index.
- **Triggers and stored procedures are the AI's blind spot.** They don't appear in `git grep` of the application code, so AI assistants trained on application code will miss them. Always grep for `CREATE TRIGGER` and `CREATE FUNCTION` when refactoring a database.
- **What to read next:** the [Postgres docs on indexes](https://www.postgresql.org/docs/current/indexes.html) (short), and Markus Winand's [Use The Index, Luke](https://use-the-index-luke.com/) (free, the single best resource on SQL performance).
