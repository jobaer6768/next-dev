# Module 18 — ORM Essentials (Drizzle)

> TypeScript types *from* your schema, queries that read like SQL, and zero runtime surprises.

---

## Why this module matters

In Missions 2 and 3 you wrote raw SQL with `pg.query` and hand-rolled repository functions. That works. But for a 30-table SaaS, you want:

- A single source of truth for the schema, **in TypeScript**.
- Type-safe queries that fail at compile time when you typo a column.
- Migration tooling that generates the SQL diffs for you.

Drizzle ORM gives you all three, with the smallest possible runtime footprint and the most SQL-like API of any TypeScript ORM in 2026.

This module is the first in Mission 4. The full SaaS backend in Assignments 4 and 5 will be built entirely on Drizzle.

---

## Learning objectives

- Explain why ORMs exist and when *not* to use one.
- Install and configure Drizzle with PostgreSQL.
- Define tables, columns, data types, and constraints in Drizzle.
- Generate, run, and push migrations.
- Write type-safe queries with `$inferSelect` and `$inferInsert`.
- Seed the database.
- Implement basic CRUD with Drizzle.

---

## 1. ORM vs query builder vs raw SQL

| Tool | Examples | Strengths | Trade-offs |
|------|----------|-----------|------------|
| **Raw SQL** | `pg`, `mysql2` | Total control, fastest | Verbose, no type safety, manual `IN` lists |
| **Query builder** | Drizzle, Kysely | Type-safe SQL, no abstraction tax | Still SQL-shaped, you write the joins |
| **Full ORM** | Prisma, TypeORM | Domain API, relations | Magic, runtime overhead, sometimes hides SQL |

Drizzle sits in the **query builder** column. It is essentially a type-safe wrapper around SQL. When you can't express something in Drizzle, drop down to `sql` template literals — they return typed rows.

---

## 2. Installation

```bash
pnpm add drizzle-orm pg
pnpm add -D drizzle-kit @types/pg
```

`drizzle.config.ts`:

```ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./src/db/schema/*.ts",
  out: "./src/db/migrations",
  dialect: "postgresql",
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
  strict: true,
  verbose: true,
});
```

`src/db/client.ts`:

```ts
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import * as schema from "./schema/index.js";

export const pool = new Pool({ connectionString: process.env.DATABASE_URL });
export const db = drizzle(pool, { schema });
export { schema };
```

---

## 3. Drizzle schema anatomy

```ts
// src/db/schema/users.ts
import { pgTable, uuid, text, timestamp } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id:        uuid("id").primaryKey().defaultRandom(),
  email:     text("email").notNull().unique(),
  name:      text("name").notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

export type User        = typeof users.$inferSelect;
export type NewUser     = typeof users.$inferInsert;
```

`pgTable` maps to a Postgres table. Each column method maps to a Postgres type with options. `$inferSelect` and `$inferInsert` are the magic — they generate TypeScript types from the schema, exactly mirroring the table.

### Common column types

| Drizzle | Postgres | Notes |
|---------|----------|-------|
| `uuid("id").primaryKey().defaultRandom()` | `UUID PRIMARY KEY DEFAULT gen_random_uuid()` | |
| `text("email").notNull().unique()` | `TEXT NOT NULL UNIQUE` | |
| `varchar("name", { length: 100 })` | `VARCHAR(100)` | prefer `text` + `CHECK` |
| `integer("age")` | `INTEGER` | |
| `bigint("views", { mode: "number" })` | `BIGINT` | `mode: "bigint"` for >2^53 |
| `numeric("amount", { precision: 10, scale: 2 })` | `NUMERIC(10,2)` | money |
| `boolean("active").notNull().default(true)` | `BOOLEAN NOT NULL DEFAULT true` | |
| `timestamp("created_at", { withTimezone: true })` | `TIMESTAMPTZ` | always use `withTimezone: true` |
| `date("spent_at")` | `DATE` | |
| `jsonb("settings").$type<MyShape>()` | `JSONB` | typed JSON |
| `pgEnum("status", ["active", "banned"])` | custom `ENUM` | |

### Constraints

```ts
import { check, uniqueIndex, index } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  // ...
}, (t) => ({
  emailLowerIdx: uniqueIndex("users_email_lower_idx").on(t.email),
  ageRange:      check("users_age_range", sql`${t.age} BETWEEN 0 AND 150`),
}));
```

`check`, `uniqueIndex`, and `index` are added in the second argument to `pgTable`.

---

## 4. The full SaaS schema (the one for Assignment 4)

```ts
// src/db/schema/index.ts
export * from "./users.js";
export * from "./workspaces.js";
export * from "./memberships.js";
export * from "./projects.js";
export * from "./tasks.js";
export * from "./comments.js";
export * from "./attachments.js";
export * from "./plans.js";
export * from "./subscriptions.js";
```

One file per table. Each file is small, focused, and the type inference flows from there.

Example: `workspaces.ts`:

```ts
import { pgTable, uuid, text, timestamp } from "drizzle-orm/pg-core";
import { users } from "./users.js";

export const workspaces = pgTable("workspaces", {
  id:        uuid("id").primaryKey().defaultRandom(),
  name:      text("name").notNull(),
  ownerId:   uuid("owner_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

export type Workspace    = typeof workspaces.$inferSelect;
export type NewWorkspace = typeof workspaces.$inferInsert;
```

---

## 5. Migrations — generate, run, push

### Generate from schema changes

```bash
pnpm drizzle-kit generate
```

This compares your schema files to the last snapshot in `src/db/migrations` and produces a new SQL migration.

```bash
pnpm drizzle-kit migrate     # apply migrations
pnpm drizzle-kit push        # push schema directly (dev only — never in prod)
pnpm drizzle-kit studio      # open Drizzle Studio in the browser
```

### Migration runner

`src/db/migrate.ts`:

```ts
import { drizzle } from "drizzle-orm/node-postgres";
import { migrate } from "drizzle-orm/node-postgres/migrator";
import { pool } from "./client.js";

const db = drizzle(pool);

async function main() {
  await migrate(db, { migrationsFolder: "./src/db/migrations" });
  await pool.end();
}
main();
```

`package.json`:
```json
"db:generate": "drizzle-kit generate",
"db:migrate":  "tsx src/db/migrate.ts",
"db:studio":   "drizzle-kit studio"
```

In production, you run `db:migrate` as a one-off job before starting the app.

---

## 6. Basic queries

```ts
import { db } from "../db/client.js";
import { users } from "../db/schema/users.js";
import { eq, ilike, and, desc } from "drizzle-orm";

// SELECT * FROM users
const all = await db.select().from(users);

// SELECT * FROM users WHERE id = $1
const one = await db.select().from(users).where(eq(users.id, id)).limit(1);

// SELECT * FROM users WHERE email ILIKE $1
const found = await db.select().from(users).where(ilike(users.email, "%@x.com%"));

// SELECT * FROM users WHERE active = true AND created_at > $1
const recent = await db.select().from(users)
  .where(and(eq(users.active, true), gt(users.createdAt, cutoff)));

// SELECT * FROM users ORDER BY created_at DESC LIMIT 20
const page = await db.select().from(users).orderBy(desc(users.createdAt)).limit(20);

// INSERT
const [created] = await db.insert(users).values({ email, name }).returning();

// UPDATE
const [updated] = await db.update(users)
  .set({ name: "Rumi A." })
  .where(eq(users.id, id))
  .returning();

// DELETE
const deleted = await db.delete(users).where(eq(users.id, id)).returning();
```

`returning()` returns the affected rows, exactly like Postgres' `RETURNING`. With `.returning()` on insert/update/delete, you get the typed result.

### Partial selects

```ts
const publicUsers = await db.select({
  id:    users.id,
  email: users.email,
  name:  users.name,
}).from(users);
```

The return type is `{ id, email, name }[]`. No password hash, no internals.

### Raw SQL escape hatch

```ts
import { sql } from "drizzle-orm";

const counts = await db.execute(sql`
  SELECT date_trunc('day', created_at) AS day, COUNT(*)::int AS n
  FROM users
  WHERE created_at >= now() - interval '30 days'
  GROUP BY 1
  ORDER BY 1
`);
```

The result is typed as `unknown[]` — assert or transform as needed.

---

## 7. Transactions

```ts
await db.transaction(async (tx) => {
  const [workspace] = await tx.insert(workspaces).values({ name, ownerId }).returning();
  await tx.insert(memberships).values({
    userId: ownerId,
    workspaceId: workspace.id,
    role: "owner",
    status: "active",
  });
});
```

The callback runs in a transaction. Throw inside and it rolls back; return and it commits.

---

## 8. Seeding

```ts
// src/db/seed.ts
import { db } from "./client.js";
import { users, workspaces, memberships, projects, tasks, plans } from "./schema/index.js";

async function main() {
  await db.insert(plans).values([
    { code: "free",     name: "Free",     priceCents: 0,    features: { seats: 3 } },
    { code: "pro",      name: "Pro",      priceCents: 1900, features: { seats: 20 } },
    { code: "business", name: "Business", priceCents: 9900, features: { seats: 100 } },
  ]);

  const [u1] = await db.insert(users).values({ email: "rumi@x.com",  name: "Rumi"  }).returning();
  const [u2] = await db.insert(users).values({ email: "nadia@x.com", name: "Nadia" }).returning();

  const [w1] = await db.insert(workspaces).values({ name: "Acme", ownerId: u1.id }).returning();
  await db.insert(memberships).values([
    { userId: u1.id, workspaceId: w1.id, role: "owner",  status: "active" },
    { userId: u2.id, workspaceId: w1.id, role: "member", status: "active" },
  ]);

  const [p1] = await db.insert(projects).values({ workspaceId: w1.id, name: "Launch", createdBy: u1.id }).returning();
  await db.insert(tasks).values([
    { projectId: p1.id, title: "Write copy",  assigneeId: u1.id, status: "open",   priority: "high" },
    { projectId: p1.id, title: "Design logo", assigneeId: u2.id, status: "open",   priority: "medium" },
    { projectId: p1.id, title: "Pick domain", assigneeId: u1.id, status: "done",   priority: "low" },
  ]);
}
main().then(() => process.exit(0));
```

Run with `pnpm tsx src/db/seed.ts`.

---

## 9. The repository pattern with Drizzle

```ts
// src/modules/users/users.repository.ts
import { db } from "../../db/client.js";
import { users, type User, type NewUser } from "../../db/schema/users.js";
import { eq, ilike, and, sql } from "drizzle-orm";

export const usersRepo = {
  async list({ q, page, limit }: { q?: string; page: number; limit: number }) {
    const where = q ? ilike(users.name, `%${q}%`) : undefined;
    const items = await db.select().from(users)
      .where(where)
      .orderBy(sql`${users.createdAt} DESC`)
      .limit(limit)
      .offset((page - 1) * limit);
    const [{ count }] = await db.select({ count: sql<number>`COUNT(*)::int` }).from(users).where(where);
    return { items, total: count };
  },

  async findById(id: string): Promise<User | null> {
    const [u] = await db.select().from(users).where(eq(users.id, id)).limit(1);
    return u ?? null;
  },

  async findByEmail(email: string): Promise<User | null> {
    const [u] = await db.select().from(users).where(eq(users.email, email)).limit(1);
    return u ?? null;
  },

  async create(input: NewUser): Promise<User> {
    const [u] = await db.insert(users).values(input).returning();
    return u;
  },

  async update(id: string, patch: Partial<NewUser>): Promise<User | null> {
    const [u] = await db.update(users).set(patch).where(eq(users.id, id)).returning();
    return u ?? null;
  },

  async delete(id: string): Promise<boolean> {
    const deleted = await db.delete(users).where(eq(users.id, id)).returning();
    return deleted.length > 0;
  },
};
```

Compared to the raw `pg` repository in Module 09, this version:

- Has zero hand-rolled SQL.
- Is fully type-safe — typo `users.emial` is a compile error.
- Returns typed `User` and `NewUser` directly.

---

## 10. Common mistakes

1. **Mutating returned rows.** Drizzle returns the *same* objects you put in. Don't mutate them.
2. **Forgetting `.returning()` on update/delete.** Without it, you get the affected count, not the rows.
3. **Using Drizzle when raw SQL is clearer.** For complex analytical queries, raw `sql` is fine.
4. **Not using migrations.** `db:push` is a dev convenience. In production, use `db:generate` + `db:migrate`.
5. **Mixing `pg` and Drizzle in the same module.** Pick one per query.

---

## Exercise

1. Set up Drizzle in a fresh project. Configure `drizzle.config.ts`, define the `users` table with all the right types and constraints.
2. Generate the first migration. Inspect the generated SQL. Apply it.
3. Build a `usersRepo` with list, findById, findByEmail, create, update, delete using Drizzle's query builder.
4. Add a seed script that inserts 5 users.
5. Write a query that uses `and`, `ilike`, and `orderBy` together.
6. Write a transaction that creates a workspace and a membership in one call.
7. Open Drizzle Studio and visually confirm the data.

---

## AI Industry Note (2026)

- **Drizzle + Zod is the most productive combination in the 2026 TypeScript backend stack.** Drizzle owns the *persistence* types; Zod owns the *boundary* types. Together they let an AI assistant scaffold an entire CRUD module in 5 minutes. The senior move is to keep the boundary clean: Zod for input, Drizzle inferred types for output, manual mapping when you need a DTO.
- **Drizzle is the only ORM that AI assistants get right most of the time.** Because the API mirrors SQL, the AI's "guesses" are usually valid. Prisma's higher-level API causes more hallucination.
- **What to read next:** the [Drizzle docs on relations](https://orm.drizzle.team/docs/rqb) (Module 19) and the [Postgres-specific docs](https://orm.drizzle.team/docs/dialect/postgresql-new) for transactions, returning, and prepared statements.
