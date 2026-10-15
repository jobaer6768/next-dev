# Module 19 — Relations & Advanced Query Techniques (Drizzle)

> Joins that read like SQL, a relational query API that erases N+1, and the migration path from `pg.query` to Drizzle.

---

## Why this module matters

In Module 18 you wrote a basic CRUD layer with Drizzle. That gets you a single table. The SaaS in Assignment 4 has **nine tables** that reference each other. You need:

- A way to *declare* those references in TypeScript, so the database and your types can never drift.
- A way to write `SELECT users.*, posts.* FROM users JOIN posts ...` without hand-rolling a `JOIN` for every screen.
- A way to paginate, aggregate, and run multi-statement work in a transaction — all in Drizzle.

This module is the second in Mission 4. By the end, you will have replaced the raw `pg.query` repository from Mission 2 with a Drizzle repository, and you will be ready to build the SaaS schema for real in Module 20.

---

## Learning objectives

- Define **1:1**, **1:N**, and **M:N** relations with Drizzle's `relations()` helper.
- Write **relational queries** with `db.query.<table>.findMany({ with: {...} })`.
- Use the **SQL-like join API**: `.leftJoin`, `.innerJoin`, `.rightJoin`, partial selects.
- Compose predicates: `eq`, `and`, `or`, `inArray`, `notInArray`, `between`, `like`, `ilike`, `isNull`, `isNotNull`, `gt`, `gte`, `lt`, `lte`.
- Drop down to `sql` for raw fragments.
- Sort with `asc()` / `desc()`, paginate **offset** and **cursor**.
- Aggregate: `.groupBy`, `.having`, `count()`, `sum()`, `avg()`, `min()`, `max()`.
- Run multi-statement work in `db.transaction(...)`.
- Migrate a Mission 2 repository from raw `pg` to Drizzle.

---

## 1. The relations helper

`relations()` is a Drizzle-side declaration. It does **not** add anything to the SQL schema — it teaches the relational query API about the foreign keys you already declared with `.references(...)`.

```ts
// src/db/schema/relations.ts
import { relations } from "drizzle-orm";
import { users } from "./users.js";
import { workspaces } from "./workspaces.js";
import { memberships } from "./memberships.js";
import { projects } from "./projects.js";
import { tasks } from "./tasks.js";
import { comments } from "./comments.js";

// 1:N — a user owns many workspaces
export const usersRelations = relations(users, ({ many, one }) => ({
  ownedWorkspaces: many(workspaces, { relationName: "owner" }),
  memberships:      many(memberships),
}));

// 1:N — a workspace has many projects, many memberships
export const workspacesRelations = relations(workspaces, ({ one, many }) => ({
  owner:      one(users, {
    fields:    [workspaces.ownerId],
    references: [users.id],
    relationName: "owner",
  }),
  projects:   many(projects),
  memberships: many(memberships),
}));

// M:N — users <-> workspaces, through memberships
export const membershipsRelations = relations(memberships, ({ one }) => ({
  user:      one(users,      { fields: [memberships.userId],      references: [users.id] }),
  workspace: one(workspaces, { fields: [memberships.workspaceId], references: [workspaces.id] }),
}));

// 1:N — a project has many tasks; tasks have one assignee
export const projectsRelations = relations(projects, ({ one, many }) => ({
  workspace: one(workspaces, { fields: [projects.workspaceId], references: [workspaces.id] }),
  creator:   one(users,      { fields: [projects.createdBy],   references: [users.id] }),
  tasks:     many(tasks),
}));

export const tasksRelations = relations(tasks, ({ one, many }) => ({
  project:  one(projects, { fields: [tasks.projectId],  references: [projects.id] }),
  assignee: one(users,    { fields: [tasks.assigneeId], references: [users.id] }),
  comments: many(comments),
}));
```

### The three relation shapes

```ts
// 1:1
export const usersRelations = relations(users, ({ one }) => ({
  profile: one(profiles, { fields: [users.id], references: [profiles.userId] }),
}));

// 1:N
export const authorsRelations = relations(authors, ({ many }) => ({
  posts: many(posts),
}));

// M:N (through a join table)
export const usersRelations = relations(users, ({ many }) => ({
  roles: many(usersRoles),   // usersRoles has userId + roleId
}));

export const usersRolesRelations = relations(usersRoles, ({ one }) => ({
  user: one(users, { fields: [usersRoles.userId], references: [users.id] }),
  role: one(roles, { fields: [usersRoles.roleId], references: [roles.id] }),
}));
```

If two tables have **two** foreign keys between them (e.g. `workspaces.ownerId` *and* `tasks.assigneeId` both point to `users`), use `relationName` to disambiguate.

---

## 2. Relational queries — the `db.query` API

Because the `db` client is constructed with `{ schema }` (see Module 18), it gets a typed `query` namespace.

```ts
const db = drizzle(pool, { schema });
// db.query.users, db.query.workspaces, db.query.tasks — all typed
```

### Eager loading with `with`

```ts
// SELECT users.*, posts.* FROM users LEFT JOIN posts ON posts.author_id = users.id
const rows = await db.query.users.findMany({
  with: { posts: true },
});
// rows: { id, email, name, posts: { ... }[] }[]
```

This is **one** SQL statement — Drizzle emits the JOIN automatically. No N+1.

### Nested with

```ts
const project = await db.query.projects.findFirst({
  where: (p, { eq }) => eq(p.id, projectId),
  with: {
    workspace: true,
    tasks: {
      with: { assignee: true, comments: { with: { author: true } } },
    },
  },
});
```

### Partial selects on a relation

```ts
const workspace = await db.query.workspaces.findFirst({
  where: (w, { eq }) => eq(w.id, id),
  columns: { id: true, name: true },   // pick which columns of the parent
  with: {
    projects: {
      columns: { id: true, name: true, status: true },
      where: (p, { eq }) => eq(p.status, "active"),
      orderBy: (p, { desc }) => desc(p.createdAt),
      limit: 10,
    },
  },
});
```

The `where` / `orderBy` / `limit` keys on a relation segment push the constraint **into the JOIN** — not into a follow-up query. That's how you get a single round trip.

### Extras: `extras` (computed columns)

```ts
const posts = await db.query.posts.findMany({
  extras: {
    commentCount: sql<number>`(SELECT COUNT(*)::int FROM comments WHERE comments.post_id = ${posts.id})`.as("comment_count"),
  },
  with: { author: true },
});
// posts: { ..., commentCount: number }[]
```

---

## 3. The SQL-like join API

When you need full control — or when you need a result shape that the relational API doesn't give you — drop to the lower-level builder.

```ts
import { eq, and, desc, sql } from "drizzle-orm";
import { db } from "../db/client.js";
import { users, posts, workspaces } from "../db/schema/index.js";

// SELECT users.*, posts.title
// FROM users
// LEFT JOIN posts ON posts.author_id = users.id
// ORDER BY posts.created_at DESC
const rows = await db
  .select({
    userId:    users.id,
    userName:  users.name,
    postTitle: posts.title,
  })
  .from(users)
  .leftJoin(posts, eq(posts.authorId, users.id))
  .orderBy(desc(posts.createdAt));
```

### `innerJoin` vs `leftJoin` vs `rightJoin`

| API | Emits | When |
|-----|-------|------|
| `.innerJoin(t, on)` | `INNER JOIN` | Drop rows where the joined table is empty |
| `.leftJoin(t, on)`  | `LEFT JOIN`  | Keep all rows from the left, fill right with `null` |
| `.rightJoin(t, on)` | `RIGHT JOIN` | Mirror of `leftJoin`; rare, can usually be rewritten |

```ts
// Users that have at least one post
const active = await db
  .select({ id: users.id, name: users.name })
  .from(users)
  .innerJoin(posts, eq(posts.authorId, users.id))
  .groupBy(users.id);
```

### Multi-join with composite ON

```ts
const rows = await db
  .select()
  .from(tasks)
  .leftJoin(projects,  eq(projects.id, tasks.projectId))
  .leftJoin(workspaces, eq(workspaces.id, projects.workspaceId))
  .where(and(eq(workspaces.id, wsId), eq(tasks.status, "open")));
```

---

## 4. Predicates — the comparison toolkit

Drizzle exports one helper per SQL operator. They all return a `SQL` expression.

```ts
import {
  eq, ne, gt, gte, lt, lte,
  and, or, not,
  inArray, notInArray,
  between, notBetween,
  like, ilike, notIlike,
  isNull, isNotNull,
  exists, notExists,
} from "drizzle-orm";

await db.select().from(users).where(eq(users.id, id));
await db.select().from(users).where(ne(users.role, "admin"));
await db.select().from(users).where(gt(users.createdAt, cutoff));
await db.select().from(users).where(gte(users.age, 18));
await db.select().from(users).where(inArray(users.id, [1, 2, 3]));
await db.select().from(users).where(notInArray(users.role, ["banned"]));
await db.select().from(users).where(between(users.age, 18, 65));
await db.select().from(users).where(like(users.name, "Ada%"));
await db.select().from(users).where(ilike(users.email, "%@acme.%"));
await db.select().from(users).where(isNull(users.deletedAt));
await db.select().from(users).where(and(eq(users.role, "user"), gte(users.age, 18)));
await db.select().from(users).where(or(eq(users.role, "admin"), eq(users.role, "owner")));
```

### The dynamic WHERE pattern

You almost never know at compile time how many filters the user picked.

```ts
import { SQL } from "drizzle-orm";

function listUsers(filters: { q?: string; role?: "admin" | "user" | "owner"; active?: boolean }) {
  const wheres: SQL[] = [];
  if (filters.q)     wheres.push(ilike(users.name, `%${filters.q}%`));
  if (filters.role)  wheres.push(eq(users.role, filters.role));
  if (filters.active !== undefined) wheres.push(eq(users.active, filters.active));

  return db.select().from(users)
    .where(wheres.length ? and(...wheres) : undefined)
    .orderBy(desc(users.createdAt));
}
```

If `wheres` is empty, pass `undefined` — Drizzle skips the `WHERE` clause entirely.

---

## 5. Raw `sql` fragments

`sql` is your escape hatch. It is also the only way to use Postgres functions, CTEs, and `RETURNING` tricks.

```ts
import { sql } from "drizzle-orm";

// Type the result explicitly
const totals = await db.execute<{ day: string; n: number }>(sql`
  SELECT date_trunc('day', created_at)::date AS day, COUNT(*)::int AS n
  FROM users
  WHERE created_at >= now() - interval '30 days'
  GROUP BY 1
  ORDER BY 1
`);

// Interpolate a column safely (Drizzle parameterizes it)
const userById = await db.select().from(users)
  .where(sql`${users.id} = ${id}`);
// equivalent to: eq(users.id, id), but useful for compound expressions
```

### Mixing `sql` into a builder

```ts
const matches = await db.select({
  id: users.id,
  rank: sql<number>`ts_rank(to_tsvector('english', ${users.name}), plainto_tsquery('english', ${q}))`.as("rank"),
}).from(users)
  .where(sql`to_tsvector('english', ${users.name}) @@ plainto_tsquery('english', ${q})`)
  .orderBy(sql`rank DESC`);
```

Anything inside `${...}` is parameterized — never interpolate user input with template strings.

---

## 6. Sorting

```ts
import { asc, desc } from "drizzle-orm";

await db.select().from(users)
  .orderBy(asc(users.name), desc(users.createdAt));
// ORDER BY name ASC, created_at DESC
```

For nullable columns, use `asc` / `desc` with `nullsFirst` / `nullsLast`:

```ts
.orderBy(sql`${users.lastSeenAt} DESC NULLS LAST`)
```

---

## 7. Pagination

### Offset pagination — the simple case

```ts
const page  = Number(qs.page  ?? 1);
const limit = Math.min(Number(qs.limit ?? 20), 100);

const items = await db.select().from(users)
  .orderBy(desc(users.createdAt))
  .limit(limit)
  .offset((page - 1) * limit);

const [{ total }] = await db
  .select({ total: sql<number>`COUNT(*)::int` })
  .from(users);
```

`offset` becomes expensive past a few thousand rows (the DB has to scan-and-discard). It's fine for admin pages, bad for infinite scroll.

### Cursor pagination — the right tool for feeds

```ts
// first page
const first = await db.select().from(tasks)
  .where(eq(tasks.projectId, projectId))
  .orderBy(desc(tasks.createdAt), desc(tasks.id))   // tiebreaker
  .limit(20);

// next page
const lastSeen = first[first.length - 1];
const next = await db.select().from(tasks)
  .where(and(
    eq(tasks.projectId, projectId),
    or(
      lt(tasks.createdAt, lastSeen.createdAt),
      and(eq(tasks.createdAt, lastSeen.createdAt), lt(tasks.id, lastSeen.id)),
    ),
  ))
  .orderBy(desc(tasks.createdAt), desc(tasks.id))
  .limit(20);
```

Always include a **tiebreaker** (`tasks.id`) — without it, two rows with the same `createdAt` can be skipped or duplicated.

Encode the cursor as base64-JSON in the URL so the client doesn't see the internals:

```ts
function encodeCursor(row: { createdAt: Date; id: string }) {
  return Buffer.from(JSON.stringify({ c: row.createdAt.toISOString(), i: row.id })).toString("base64url");
}
function decodeCursor(s: string) {
  const { c, i } = JSON.parse(Buffer.from(s, "base64url").toString("utf8"));
  return { createdAt: new Date(c), id: i };
}
```

---

## 8. Aggregations

```ts
import { count, sum, avg, min, max } from "drizzle-orm";

// COUNT
const [{ n }] = await db.select({ n: count() }).from(users);

// SUM with an expression
const [{ total }] = await db.select({ total: sum(tasks.storyPoints) })
  .from(tasks)
  .where(eq(tasks.status, "done"));

// AVG
const [{ avgAge }] = await db.select({ avgAge: avg(users.age) }).from(users);
```

### `groupBy` + `having`

```ts
const perProject = await db
  .select({
    projectId: tasks.projectId,
    open:      sql<number>`COUNT(*) FILTER (WHERE ${tasks.status} = 'open')::int`,
    done:      sql<number>`COUNT(*) FILTER (WHERE ${tasks.status} = 'done')::int`,
  })
  .from(tasks)
  .groupBy(tasks.projectId)
  .having(sql`COUNT(*) > 5`);
```

`FILTER (WHERE ...)` is the Postgres-standard way to do conditional aggregation. Don't wrap it in a `CASE` unless you have to.

---

## 9. Transactions

```ts
await db.transaction(async (tx) => {
  const [ws] = await tx.insert(workspaces).values({ name, ownerId }).returning();
  await tx.insert(memberships).values({
    userId:      ownerId,
    workspaceId: ws.id,
    role:        "owner",
    status:      "active",
  });
  // throw anywhere → automatic rollback
});
```

### Isolation levels

```ts
await db.transaction(
  async (tx) => { /* ... */ },
  { isolationLevel: "serializable" }   // "read uncommitted" | "read committed" | "repeatable read" | "serializable"
);
```

### Savepoints

```ts
await db.transaction(async (tx) => {
  await tx.insert(a).values({ ... });
  await tx.transaction(async (tx2) => {   // savepoint
    try {
      await tx2.insert(b).values({ ... });
    } catch { /* swallow, continue */ }
  });
});
```

Use a savepoint when one step of a larger transaction may fail and you want to keep the rest of the work. Don't use it as a try/catch for "real" failures — fix the bug.

---

## 10. Migrating the Mission 2 repository from `pg` to Drizzle

The Mission 2 `pg` repo (Module 09) looked like this:

```ts
// BEFORE — raw pg
import { pool } from "../db/pool.js";

export const tasksRepo = {
  async list(projectId: string) {
    const { rows } = await pool.query(
      `SELECT id, title, status, created_at
         FROM tasks
        WHERE project_id = $1
        ORDER BY created_at DESC`,
      [projectId],
    );
    return rows;
  },
  async create(input: { projectId: string; title: string }) {
    const { rows } = await pool.query(
      `INSERT INTO tasks (project_id, title) VALUES ($1, $2) RETURNING *`,
      [input.projectId, input.title],
    );
    return rows[0];
  },
};
```

The Drizzle version is shorter and type-safe end-to-end:

```ts
// AFTER — Drizzle
import { db } from "../db/client.js";
import { tasks, type Task, type NewTask } from "../db/schema/tasks.js";
import { eq, desc } from "drizzle-orm";

export const tasksRepo = {
  async list(projectId: string): Promise<Task[]> {
    return db.select().from(tasks)
      .where(eq(tasks.projectId, projectId))
      .orderBy(desc(tasks.createdAt));
  },
  async create(input: NewTask): Promise<Task> {
    const [row] = await db.insert(tasks).values(input).returning();
    return row;
  },
};
```

### Side-by-side comparison

| Concern | `pg.query` | Drizzle |
|---------|-----------|---------|
| Column types | Manual, hand-written | `$inferSelect` / `$inferInsert` |
| `WHERE $1, $2` | Manual placeholders | Typed helpers (`eq`, `and`, …) |
| `RETURNING *` | Manual string | `.returning()` |
| Transactions | `BEGIN / COMMIT / ROLLBACK` boilerplate | `db.transaction(async tx => …)` |
| Joins | Hand-written SQL | `.leftJoin`, `relations()` |
| Migrations | Hand-written `*.sql` files | `drizzle-kit generate` |
| Connection pool | `Pool` from `pg` | Same `Pool`, wrapped by `drizzle()` |

### Migration steps (do these in order)

1. **Install** Drizzle alongside `pg`. Don't uninstall `pg` yet — you still use its `Pool`.
2. **Define** the schema in `src/db/schema/*.ts` with `pgTable` mirroring your existing tables.
3. **Run** `drizzle-kit generate` to produce the first migration. The diff will be empty if the tables already exist; that's fine — you're capturing the baseline.
4. **Add the `schema` option** to your `drizzle()` call: `drizzle(pool, { schema })`.
5. **Rewrite** one repository at a time, starting with the smallest. Keep the public function signatures identical so the route handlers don't change.
6. **Run tests** after every repo. The behavior should be byte-identical.
7. **Delete** the old raw SQL once all repos are migrated.

---

## 11. Common mistakes

1. **Two relations to the same table without `relationName`.** `workspaces.ownerId` and `tasks.assigneeId` both point to `users`; Drizzle will throw a "two relations" error at startup.
2. **Confusing `with` (relational) with `.leftJoin` (SQL-like).** `with` returns a nested object; `.leftJoin` returns a flat row per match.
3. **Forgetting the tiebreaker on cursor pagination.** Same `createdAt` → rows get skipped or duplicated.
4. **Letting `offset` grow.** `OFFSET 100000` is a full scan. Use cursor pagination for any list that grows forever.
5. **Stringly interpolating user input into `sql`**. `sql\`... WHERE name = '${req.query.name}'\`` is a SQL-injection bug. Use `${column}` or parameter placeholders.
6. **Throwing inside `db.transaction` and expecting partial commit.** Throws always roll back. Use savepoints if you need partial work.
7. **Mutating result rows.** Drizzle returns the same objects you put in. Treat them as read-only.
8. **Using `db:push` in any non-dev environment.** Always `db:generate` + `db:migrate` in CI and prod.

---

## Exercise

1. Add `relations()` declarations for **all** tables in your Mission 2 schema (users, posts, comments, plus the SaaS tables from Module 18).
2. Write a relational query that returns a workspace with its projects, and each project's tasks with their assignees — in a single round trip.
3. Write the same query with the SQL-like join API (`.leftJoin`) and confirm the SQL is identical by enabling Drizzle's `logger: true`.
4. Build `listTasks(filters)` that supports a free-text `q`, a status filter, a date `between`, and dynamic `orderBy`.
5. Implement cursor pagination for `listTasks`. Encode the cursor as base64 in the route.
6. Write an aggregation: for each project, return `open` and `done` task counts and the `avg` story points, filtered to projects with more than 5 tasks.
7. Wrap a workspace + membership creation in `db.transaction`. Add a contrived failure (e.g. `throw new Error("boom")`) and confirm nothing was committed.
8. Port **one** repository from your Mission 2 raw-`pg` code to Drizzle, preserving the public API. Run the route's tests.

---

## AI Industry Note (2026)

- **Drizzle's relational query API is the closest the 2026 TypeScript ecosystem has to "LINQ but for SQL."** It's the only major ORM whose AI-generated code is *usually* right on the first try, because the calls read like SQL clauses. When you prompt an assistant for "find users with their posts, ordered by post count," it produces code that actually runs.
- **Cursor pagination is making a comeback** as AI-generated infinite-scroll UIs become the norm. AI assistants default to offset pagination because it's easier to write; the senior move is to ask for "cursor pagination with a tiebreaker" explicitly.
- **`sql` template literals are the AI's favorite escape hatch.** Most of the Postgres-specific patterns you'll see in production (CTEs, `FILTER` clauses, `tsvector` search, `LATERAL` joins) are best expressed as raw `sql` blocks. Get comfortable reading them.
- **What to read next:** the [Drizzle relational query docs](https://orm.drizzle.team/docs/rqb), the [PostgreSQL transaction docs](https://orm.drizzle.team/docs/transactions), and the [drizzle-kit migration guide](https://orm.drizzle.team/docs/kit-overview).
