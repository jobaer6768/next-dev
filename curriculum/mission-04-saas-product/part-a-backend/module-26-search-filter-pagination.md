# Module 26: Search, Filtering, Sorting & Pagination

> "An endpoint that returns the whole table is a denial-of-service attack you haven't noticed yet."

---

## 1. Why this matters

The single most common performance bug in early-stage SaaS is "the list page got slow." You started with `SELECT *` and no `LIMIT`. Six months later you have a million rows and the page takes nine seconds. By then, fixing it is a quarter-long project because every endpoint has the same shape.

This module is about not painting yourself into that corner. The right patterns are:

- **Index the queries the product actually runs**, not the ones you imagine it might run.
- **Project only the columns you need.** Don't ship `description` (a 10 KB text field) on a list view.
- **Paginate every list endpoint.** No exceptions.
- **Prefer cursor pagination** for "infinite scroll" surfaces; reserve offset for "page 47 of 412" admin views.

Search and filtering should also be **bounded** — never let the client send arbitrary SQL. A strict allow-list of sortable columns and a typed schema for filter values keeps your database safe and your codebase sane.

---

## 2. Learning objectives

By the end of this module you can:

1. Build a typed, Zod-validated query schema for search/filter/sort/pagination.
2. Implement case-insensitive search across multiple text fields using `ILIKE` and a `tsvector` index.
3. Build a reusable `paginate()` helper supporting both offset and cursor modes.
4. Implement cursor pagination using `(createdAt, id) < (cursorCreatedAt, cursorId)` tuple comparison.
5. Use `EXPLAIN ANALYZE` to verify a query is using its index.
6. Add the right indexes for the product's hot queries.
7. Recognize and prevent N+1 queries with eager loading via Drizzle's `with`.

---

## 3. Conceptual model

### 3.1 The query contract: Zod schema first

The schema is the source of truth. The route validates, the service consumes the typed object.

```ts
// src/schemas/listTasks.ts
import { z } from 'zod';

const SORTABLE = ['createdAt', 'updatedAt', 'dueAt', 'title', 'status'] as const;
type Sortable = typeof SORTABLE[number];

export const listTasksQuery = z.object({
  q: z.string().trim().min(1).max(100).optional(),                    // search query
  status: z.enum(['todo', 'in_progress', 'done', 'archived']).optional(),
  projectId: z.string().uuid().optional(),
  assigneeId: z.string().uuid().optional(),
  dueBefore: z.coerce.date().optional(),
  dueAfter: z.coerce.date().optional(),
  sortBy: z.enum(SORTABLE).default('createdAt'),
  sortDir: z.enum(['asc', 'desc']).default('desc'),
  cursor: z.string().optional(),                                      // for cursor pagination
  limit: z.coerce.number().int().min(1).max(100).default(25),
  page: z.coerce.number().int().min(1).optional(),                    // for offset pagination
});

export type ListTasksQuery = z.infer<typeof listTasksQuery>;
```

`SORTABLE` is the **allow-list**. Anything not in it is rejected by Zod, before the query ever runs.

### 3.2 The reusable paginate() helper

```ts
// src/lib/paginate.ts
import { and, asc, desc, eq, lt, gt, sql, type SQL } from 'drizzle-orm';
import type { PgColumn } from 'drizzle-orm/pg-core';

export type PaginateInput<T extends PgColumn> = {
  table: any;
  where: SQL | undefined;
  sortBy: T;
  sortDir: 'asc' | 'desc';
  tieBreaker: PgColumn;                              // typically `id`
  limit: number;
  cursor?: string;
  page?: number;                                     // offset mode
};

export type PaginateResult<Row> = {
  rows: Row[];
  pageInfo:
    | { mode: 'cursor'; nextCursor: string | null; hasMore: boolean }
    | { mode: 'offset'; page: number; pageSize: number; total: number; totalPages: number };
};

const encodeCursor = (sortValue: any, id: string) =>
  Buffer.from(JSON.stringify([sortValue instanceof Date ? sortValue.toISOString() : sortValue, id])).toString('base64url');

const decodeCursor = (raw: string): [any, string] => {
  const [a, b] = JSON.parse(Buffer.from(raw, 'base64url').toString('utf8'));
  return [a, b];
};

/**
 * One paginate() to rule them all.
 *  - If `cursor` is provided, runs in cursor mode and returns nextCursor.
 *  - Else if `page` is provided, runs in offset mode and returns totals.
 *  - Else defaults to cursor mode at page 1.
 */
export async function paginate<Row, S extends PgColumn>(input: PaginateInput<S>): Promise<PaginateResult<Row>> {
  const { table, where, sortBy, sortDir, tieBreaker, limit, cursor, page } = input;
  const dir = sortDir === 'asc' ? asc : desc;

  // --- Cursor mode ---
  if (cursor || page === undefined) {
    const cursorWhere = cursor
      ? (() => {
          const [sv, id] = decodeCursor(cursor);
          const cmp = sortDir === 'asc' ? gt : lt;
          // (sortBy, id) <op> (sv, id)  — the standard tuple trick
          return sql`(${table[sortBy.name]} < ${sv} OR (${table[sortBy.name]} = ${sv} AND ${table[tieBreaker.name]} < ${id}))`;
        })()
      : undefined;
    const finalWhere = and(where, cursorWhere);

    const rows = await db.select().from(table).where(finalWhere)
      .orderBy(dir(table[sortBy.name]), dir(table[tieBreaker.name]))
      .limit(limit + 1);

    const hasMore = rows.length > limit;
    const trimmed = hasMore ? rows.slice(0, limit) : rows;
    const last = trimmed[trimmed.length - 1] as any;
    const nextCursor = hasMore && last ? encodeCursor(last[sortBy.name], last[tieBreaker.name]) : null;

    return { rows: trimmed as Row[], pageInfo: { mode: 'cursor', nextCursor, hasMore } };
  }

  // --- Offset mode ---
  const offset = (page - 1) * limit;
  const [{ total }] = await db.execute(sql`SELECT COUNT(*)::int AS total FROM ${table} WHERE ${where ?? sql`TRUE`}`);
  const rows = await db.select().from(table).where(where)
    .orderBy(dir(table[sortBy.name]), dir(table[tieBreaker.name]))
    .limit(limit).offset(offset);

  return {
    rows: rows as Row[],
    pageInfo: { mode: 'offset', page, pageSize: limit, total, totalPages: Math.ceil(total / limit) },
  };
}
```

Why the `tieBreaker`? Without it, two rows with the same `createdAt` would shuffle between pages and the same row could appear twice. `(sortBy, id)` is a stable sort key.

### 3.3 Search with case-insensitive matching

For simple substring search, `ILIKE` works. For real text search, a `tsvector` column with a GIN index is dramatically faster at scale.

```ts
// In the schema:
import { index, customType } from 'drizzle-orm/pg-core';

const tsvector = customType<{ data: string }>({ dataType: () => 'tsvector' });

export const tasks = pgTable('tasks', {
  // ... existing columns ...
  search: tsvector('search').generatedAlwaysAs(
    sql`to_tsvector('english', coalesce(title,'') || ' ' || coalesce(description,''))`,
    { mode: 'stored' }
  ),
}, (t) => ({
  searchIdx: index('tasks_search_idx').using('gin', t.search),
}));
```

The query:

```ts
// src/services/taskService.ts
import { and, eq, ilike, or, sql } from 'drizzle-orm';
import { tasks } from '../db/schema/tasks';
import { paginate } from '../lib/paginate';

export async function listTasks(workspaceId: string, q: ListTasksQuery) {
  const filters = [eq(tasks.workspaceId, workspaceId), isNull(tasks.deletedAt)];

  if (q.status) filters.push(eq(tasks.status, q.status));
  if (q.projectId) filters.push(eq(tasks.projectId, q.projectId));
  if (q.assigneeId) filters.push(eq(tasks.assigneeId, q.assigneeId));
  if (q.dueBefore) filters.push(lt(tasks.dueAt, q.dueBefore));
  if (q.dueAfter) filters.push(gt(tasks.dueAt, q.dueAfter));

  if (q.q) {
    const like = `%${q.q.replace(/[%_]/g, '\\$&')}%`;
    // ILIKE for ad-hoc; websearch_to_tsquery for ranked full-text.
    filters.push(sql`(${ilike(tasks.title, like)} OR ${ilike(tasks.description, like)} OR ${tasks.search} @@ websearch_to_tsquery('english', ${q.q}))`);
  }

  return paginate({
    table: tasks,
    where: and(...filters),
    sortBy: tasks[q.sortBy],
    sortDir: q.sortDir,
    tieBreaker: tasks.id,
    limit: q.limit,
    cursor: q.cursor,
    page: q.page,
  });
}
```

Notice: `sortBy: tasks[q.sortBy]` — TypeScript happily indexes the table by the enum value, so the allow-list is also a type guard.

### 3.4 Eager loading (kill the N+1)

A list endpoint that loads tasks, then for each task loads the assignee, fires 1 + N queries. Use Drizzle's `with`:

```ts
const rows = await db.query.tasks.findMany({
  where: ...,
  with: {
    assignee: { columns: { id: true, name: true, avatarUrl: true } },
    project:   { columns: { id: true, name: true } },
  },
});
```

For deeply nested data, build relations in the schema:

```ts
// src/db/schema/tasks.ts
import { relations } from 'drizzle-orm';
import { users } from './users';
import { projects } from './projects';

export const tasksRelations = relations(tasks, ({ one, many }) => ({
  assignee: one(users, { fields: [tasks.assigneeId], references: [users.id] }),
  project:  one(projects, { fields: [tasks.projectId], references: [projects.id] }),
  comments: many(comments),
}));
```

### 3.5 Verifying the index with `EXPLAIN ANALYZE`

```sql
EXPLAIN ANALYZE
SELECT id, title, status FROM tasks
WHERE workspace_id = '00000000-0000-0000-0000-000000000000'
  AND deleted_at IS NULL
  AND status = 'todo'
ORDER BY created_at DESC, id DESC
LIMIT 26;
```

You want to see `Index Scan using tasks_by_workspace` followed by a tiny `Limit`. If you see `Seq Scan`, the index is missing or the planner doesn't think it's selective enough. Two fixes:

- Add a **composite index** matching the predicate order: `(workspace_id, deleted_at, status, created_at DESC, id DESC)`.
- Run `ANALYZE tasks;` so the planner has fresh statistics.

For the search case:

```sql
EXPLAIN ANALYZE
SELECT id FROM tasks
WHERE workspace_id = $1 AND search @@ websearch_to_tsquery('english', $2)
ORDER BY created_at DESC LIMIT 26;
```

This should use the GIN index for the `@@` predicate and then re-sort. If it's slow, add an index on `(workspace_id, created_at DESC)` and force the planner to use it.

### 3.6 Indexing the queries the product actually runs

Don't index speculatively. The discipline is:

1. Enable slow-query logging in Postgres (`log_min_duration_statement = 500`).
2. Watch the top offenders in your APM for two weeks.
3. For each, write the actual query and run `EXPLAIN`.
4. Add the smallest index that makes it fast.

Useful index shapes you'll write repeatedly:

- `INDEX (workspace_id, deleted_at)` on every tenant table.
- `INDEX (workspace_id, project_id, created_at DESC)` on `tasks` for the project page.
- `GIN (search)` on `tasks` for full-text.
- `INDEX (workspace_id, assignee_id, status)` for "my open tasks" views.

### 3.7 Performance hygiene

- **Projection.** `SELECT id, title, status, due_at, created_at FROM tasks` — never `SELECT *` for a list view.
- **No N+1.** If you write `for (const t of tasks) { await loadAssignee(t.assigneeId) }`, refactor.
- **No count(*)** on unbounded queries. If you must, do it during off-peak or cache it.
- **Skip the offset when the user wants "everything from here on".** Cursor is faster for deep scrolls.

---

## 4. Common mistakes

1. **Sorting by an unindexed column.** The user clicks "Sort by description" and the page dies. The allow-list is your friend — only expose columns that are indexed.

2. **`OFFSET 100000`** for "load the next page." Deep offsets are O(N) in Postgres. Use a cursor for infinite scroll, and reserve offset for shallow page numbers.

3. **Storing the cursor in a JWT or in localStorage without versioning.** A user with a stale cursor from before a schema change gets nothing. Document the cursor format and tolerate failures (treat invalid cursor as "no cursor").

4. **`ILIKE '%term%'` with no index.** That's a sequential scan. For anything but tiny tables, use `tsvector` or a dedicated search engine.

5. **Returning `description` on the list endpoint** because "it might be useful." A 10 KB text field on 100 rows is a 1 MB response. Project aggressively.

6. **PaginatING before filtering.** Always filter first, then paginate. Otherwise `LIMIT 25` applies to "the first 25 unfiltered rows" and the user sees the same page every time.

7. **Forgetting the tie-breaker.** Two rows with the same `createdAt` and the same `id` (impossible, but two rows with the same `createdAt` is not) cause duplicates and skips. The `(sortBy, id)` tuple fixes this.

---

## 5. Exercise

1. Build the `listTasksQuery` Zod schema and the `paginate()` helper. Cover both cursor and offset modes.
2. Wire it into `GET /workspaces/:workspaceId/tasks`. Return `{ rows, pageInfo }`.
3. Add a `tsvector` column and GIN index to `tasks`. Run a migration.
4. Use `EXPLAIN ANALYZE` before and after the index. Record the timings.
5. Add an `assignee` eager-load to the response. Compare `EXPLAIN ANALYZE` row counts before and after — you should see one query, not 1 + N.
6. **Stretch:** Add a "saved views" feature — a user can save their current filter/sort/pagination state and recall it by name.

**Acceptance criteria:**
- `?status=todo&sortBy=dueAt&sortDir=asc` returns the right rows in the right order.
- A cursor from the first response loads the next page without overlap.
- Full-text search for "deploy" matches the word "deployment" (via stemming).
- No query in the list endpoint is a sequential scan on `tasks` for workspaces with > 1k rows.

---

## 6. AI Industry Note (2026)

By 2026, "AI search" has bifurcated. On one side, classical `tsvector`+GIN still wins for small datasets and structured filters. On the other, vector databases (pgvector, Turbopuff, Weaviate) have become the default for "semantic" search — "find me tasks *like* this one." Most production stacks now run *both*: a structured filter pipeline (this module) and a vector-recall stage, fused with a re-ranker.

A subtle point: vector search is *recall-oriented*; it returns plausible candidates that are sometimes wrong. Structured search is *precision-oriented*; it returns what the user asked for. The right UX is structured-first, semantic-fallback. The lesson for you is the same one this module has been teaching: explicit, typed, allow-listed contracts for what the API can do. The moment you let the client say "give me everything like this" without a `workspace_id`, the model will, eventually, leak across tenants.

A final note: the "AI pair programmer" of 2026 will happily write a `SELECT *` for you. Don't accept it. Make it write the projection, write the `LIMIT`, write the cursor. Be the senior engineer in the loop.
