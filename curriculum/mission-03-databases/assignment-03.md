# Assignment 3 — Design & Build a Production Database

> Two conceptual sessions anchor this assignment: **"Normalize this live: turning a messy spreadsheet into 3NF"** and **"Index or not? Reading EXPLAIN ANALYZE together."**

---

## Why this assignment matters

This is the schema that your Mission 4 SaaS will be built on. If it's wrong here, every line of application code you write in Mission 4 will be fighting the database.

You will be graded on:

- **Correctness** — does the schema enforce the brief's invariants?
- **Normalization** — is every table in 3NF (or a documented, justified denormalization)?
- **Performance** — does `EXPLAIN ANALYZE` show the indexes you added are actually used?
- **Documentation** — can a teammate read the ERD and the schema docs in 10 minutes and understand the model?

---

## Submission rules

- A single GitHub repo `nl-m3-assignment-3`.
- `pnpm install` should not be required (this is a database project, no code). Add a `Makefile` or `package.json` with scripts for setup.
- A `README.md` with the brief, the ERD, the design notes, and instructions to run.
- All SQL in `migrations/` and a `seed.sql` to populate test data.
- A `docs/SCHEMA.md` describing every table.

---

## The brief (use this exact one)

> "Build the database for a multi-tenant project management SaaS. Each user can be a member of multiple workspaces. Each workspace has many projects, each project has many tasks. A task has an assignee (a user), a status, a priority, an optional due date, and zero or more comments. Tasks can have file attachments. Workspaces have a single active subscription tied to a billing plan. Plans have a price and a feature set (stored as JSONB). Every entity has `created_at` and `updated_at`. The system must support soft delete for tasks (so they can be restored)."

---

## Deliverables

### 1. ER diagram

A Mermaid `erDiagram` in the `README.md`. Every entity, every relationship, every cardinality.

### 2. Migration scripts

```
migrations/
├── 001_extensions.sql       -- pgcrypto, pg_trgm, etc.
├── 002_users.sql
├── 003_workspaces.sql
├── 004_memberships.sql
├── 005_projects.sql
├── 006_tasks.sql
├── 007_comments.sql
├── 008_attachments.sql
├── 009_plans.sql
├── 010_subscriptions.sql
├── 011_indexes.sql          -- all CREATE INDEX
├── 012_triggers.sql         -- updated_at, etc.
└── ...
```

Each file is runnable on its own. The runner script is in `scripts/migrate.sh` (or `db/migrate.ts` if you prefer Node).

### 3. Seed data

A `seed.sql` that creates:

- 5 users
- 3 workspaces
- 3 plans (`free`, `pro`, `business`)
- 2 subscriptions
- 5 projects
- 20 tasks (mix of statuses, priorities, due dates)
- 10 comments
- 5 attachments (file URLs)

### 4. Required queries (in `queries/` or a `docs/QUERIES.md`)

For each query, include the SQL **and** the `EXPLAIN ANALYZE` output (before and after indexes).

1. List all open tasks in any workspace a user is a member of, sorted by priority then due date.
2. Count tasks per project, returning projects with zero tasks as `0`.
3. The top 10 most active workspaces by task count in the last 30 days.
4. For a given workspace, monthly task creation counts for the last 12 months (fill in missing months with zero).
5. List all overdue tasks (due_at < now, status not in done/archived) with assignee email and project name.
6. Move all "done" tasks in a project to "archived" if they were updated more than 30 days ago. Return the list of affected IDs.
7. A "task detail" query: one task with its assignee, project, workspace, comments (newest first), and attachments.
8. The most-commented tasks this month, top 5.

### 5. Triggers and functions

- `set_updated_at` trigger on every table with `updated_at`.
- An audit log trigger for `tasks` (writes to `task_audit`).
- A function `workspace_member_count(workspace UUID)` that returns the number of active members.

### 6. Indexes — justified

Add an `INDEXES.md` that lists every index, the query it supports, and a snippet of `EXPLAIN ANALYZE` proving the planner uses it.

At minimum:

- Indexes on every foreign key.
- Composite index on `tasks(project_id, status, due_at)`.
- Partial index on `tasks(assignee_id) WHERE status NOT IN ('done','archived')`.
- GIN index on `plans(features jsonb_path_ops)`.
- Index on `memberships(user_id) WHERE status = 'active'`.

### 7. Schema documentation

`docs/SCHEMA.md` — for every table:

- Purpose (one sentence).
- Columns with types, nullability, defaults, and constraints.
- Foreign keys and `ON DELETE` behavior.
- Indexes.
- Example queries.

---

## Conceptual session 1 — "Normalize this live"

In your `README.md`, write a 1–2 page essay that walks through:

1. The brief above.
2. The first-cut "single table" version that would fail normalization.
3. The anomalies (update / insertion / deletion) that version would produce.
4. The 3NF tables you actually built, with a sentence on what each table protects.

Include a small ASCII or Mermaid diagram of the spreadsheet, with arrows to the 3NF tables.

## Conceptual session 2 — "Index or not? Reading EXPLAIN ANALYZE together"

In your `README.md`, pick **one** of the queries from Deliverable 4. Show:

1. The query.
2. `EXPLAIN ANALYZE` *before* you added the index that supports it.
3. The index you added.
4. `EXPLAIN ANALYZE` *after*.

Explain in plain English what changed and why the planner made a different choice.

---

## Grading rubric (out of 100)

| Area | Points |
|------|--------|
| Schema enforces the brief's invariants (NOT NULL, CHECK, FK ON DELETE) | 15 |
| 3NF + every M:N has a junction table | 10 |
| Migrations runnable end to end on a fresh database | 10 |
| Seed data is realistic and exercises all features | 5 |
| Required queries produce correct results | 15 |
| `EXPLAIN ANALYZE` evidence of index usage | 10 |
| Triggers and functions implemented | 5 |
| ERD + SCHEMA.md + README docs are clear | 15 |
| Conceptual essays (normalization, indexing) | 15 |

---

## Deadline and how to submit

- **Deadline:** 7 days from the start of Mission 3.
- Push to GitHub; open a PR titled `[NL-M3-A3] <your-name>`.
- The grader will run your migrations on a fresh database, load your seed, and execute your queries.

---

## AI Industry Note (2026)

- **This is the assignment most candidates fail in real system design interviews.** "Design a database for X" is the single most common senior backend question in 2026. The grading rubric mirrors what a staff engineer would check in 30 minutes. **Practice this exact assignment 3 times with different briefs.**
- **The conceptual essays are the senior signal.** Anyone can produce a schema with AI. Almost no one can explain *why* the schema is in 3NF or *why* a particular index is used. Those explanations are what separate senior from staff.
- **What to do after you submit:** put the schema in a `nl-saas-schema` repo with an `ARCHITECTURE.md`, the ERD, and a 5-minute Loom walking through the design. That is the artifact that gets you the interview.
