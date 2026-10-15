# Module 08 — PostgreSQL Essentials & Connecting Express with PG

> Your data now has a home. Welcome to the database 80% of the backend world uses.

---

## Why this module matters

In Module 07 you built an API with an in-memory `Map`. The moment the process restarts, all your users vanish. In production, your data must survive crashes, deploys, and traffic spikes. That means a real database.

PostgreSQL is the **default choice** for new TypeScript backends in 2026 because:

- Rock-solid ACID guarantees.
- Excellent JSON support (for when relational isn't quite right).
- pgvector extension for AI embeddings (we use it in Mission 6).
- Open source, runs anywhere.

This module is hands-on. By the end, you will have wired Express to a real Postgres database.

---

## Learning objectives

- Understand what a relational database is and how to think in tables.
- Install PostgreSQL and pgAdmin.
- Connect Express to Postgres using `pg` and a connection pool.
- Write parameterized queries to prevent SQL injection.
- Perform CRUD via raw SQL.

---

## 1. What is a relational database?

A **relational database** stores data in **tables** (relations). Each table has:

- **Columns** with names and types (`name TEXT`, `age INT`).
- **Rows** (records).
- **Constraints** (`NOT NULL`, `UNIQUE`, `PRIMARY KEY`, `FOREIGN KEY`).

You query the data with **SQL** (Structured Query Language). SQL is declarative — you describe *what* you want, not *how* to get it.

The database engine handles:

- **Storage** on disk.
- **Indexes** for fast lookup.
- **Transactions** for atomic updates.
- **Concurrency control** so two requests don't clobber each other.

You handle:

- **Schema design** (which tables, which columns, which relations).
- **Queries** (how to fetch/update data).
- **Migrations** (how the schema evolves over time).

---

## 2. Installing PostgreSQL

**macOS (Homebrew):**

```bash
brew install postgresql@16
brew services start postgresql@16
```

**Ubuntu / WSL:**

```bash
sudo apt install -y postgresql-16
sudo service postgresql start
```

**Windows:** use the [EnterpriseDB installer](https://www.postgresql.org/download/windows/) or run it in Docker:

```bash
docker run -d --name pg -e POSTGRES_PASSWORD=postgres -p 5432:5432 postgres:16
```

**Verify:**

```bash
psql --version
psql -U postgres -c "SELECT version();"
```

---

## 3. pgAdmin — your visual client

Install [pgAdmin 4](https://www.pgadmin.org/). Connect to `localhost:5432`, user `postgres`.

In pgAdmin you can:

- Browse schemas, tables, indexes.
- Run SQL queries.
- View execution plans.
- See live query activity.

Use pgAdmin for *exploration*; use the command line for *real work*.

---

## 4. Database and table basics

```sql
-- Create a database
CREATE DATABASE next_level;

-- Connect to it
\c next_level

-- Create a table
CREATE TABLE users (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email       TEXT NOT NULL UNIQUE,
  name        TEXT NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Common types
-- TEXT, VARCHAR(n)
-- INT, BIGINT, NUMERIC(p,s)
-- BOOLEAN
-- DATE, TIMESTAMP, TIMESTAMPTZ
-- UUID
-- JSONB    (binary JSON, indexable)
-- BYTEA    (binary)

-- Insert
INSERT INTO users (email, name) VALUES ('rumi@x.com', 'Rumi');

-- Read
SELECT id, email, name, created_at FROM users ORDER BY created_at DESC;

-- Update
UPDATE users SET name = 'Rumi A.' WHERE id = '...';

-- Delete
DELETE FROM users WHERE id = '...';
```

`gen_random_uuid()` is built into Postgres 13+.

---

## 5. Primary keys, foreign keys, joins (preview)

```sql
CREATE TABLE projects (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name        TEXT NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Join
SELECT p.id, p.name, u.email AS owner_email
FROM projects p
JOIN users u ON u.id = p.owner_id;
```

We will go deep on schema design in Mission 3 (Modules 11–17). For now, this is enough to wire up the API.

---

## 6. Connecting Node to PostgreSQL with `pg`

```bash
pnpm add pg
pnpm add -D @types/pg
```

### A connection pool

A **pool** keeps N connections open and reuses them. Never open a connection per request — that's the #1 cause of "the database is slow" in production.

```ts
// src/db/pool.ts
import { Pool } from "pg";

export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,                       // max concurrent connections
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000,
});

pool.on("error", (err) => {
  console.error("Unexpected pg pool error", err);
});
```

### A query helper

```ts
// src/db/query.ts
import { pool } from "./pool.js";

export async function query<T extends Record<string, unknown>>(
  text: string,
  params: unknown[] = [],
): Promise<T[]> {
  const result = await pool.query<T>(text, params as unknown[]);
  return result.rows;
}

export async function queryOne<T extends Record<string, unknown>>(
  text: string,
  params: unknown[] = [],
): Promise<T | null> {
  const rows = await query<T>(text, params);
  return rows[0] ?? null;
}
```

`pool.query<T>` is generic on the row type, giving you type-safe results when combined with the schema you wrote.

---

## 7. Parameterized queries — the SQL injection defense

```ts
// ❌ NEVER do this
const sql = `SELECT * FROM users WHERE email = '${email}'`;

// ✅ Always parameterize
const sql = `SELECT * FROM users WHERE email = $1`;
const rows = await query(sql, [email]);
```

The `$1`, `$2` placeholders are filled in by the driver with proper escaping. There is **no excuse** to build SQL by string concatenation. Tools like `pg` make it trivial.

---

## 8. A users repository backed by Postgres

```ts
// src/data/users.ts
import { randomUUID } from "node:crypto";
import { query, queryOne } from "../db/query.js";
import type { User } from "../types.js";

type UserRow = {
  id: string;
  email: string;
  name: string;
  created_at: Date;
};

function toUser(r: UserRow): User {
  return { id: r.id, email: r.email, name: r.name, createdAt: r.created_at };
}

export const usersRepo = {
  async list(limit = 50): Promise<User[]> {
    const rows = await query<UserRow>(
      `SELECT id, email, name, created_at FROM users ORDER BY created_at DESC LIMIT $1`,
      [limit],
    );
    return rows.map(toUser);
  },

  async findById(id: string): Promise<User | null> {
    const row = await queryOne<UserRow>(
      `SELECT id, email, name, created_at FROM users WHERE id = $1`,
      [id],
    );
    return row ? toUser(row) : null;
  },

  async create(input: { email: string; name: string }): Promise<User> {
    const row = await queryOne<UserRow>(
      `INSERT INTO users (email, name) VALUES ($1, $2)
       RETURNING id, email, name, created_at`,
      [input.email, input.name],
    );
    if (!row) throw new Error("insert returned no row");
    return toUser(row);
  },

  async update(id: string, patch: Partial<{ name: string; email: string }>): Promise<User | null> {
    const row = await queryOne<UserRow>(
      `UPDATE users
         SET name = COALESCE($2, name),
             email = COALESCE($3, email)
       WHERE id = $1
       RETURNING id, email, name, created_at`,
      [id, patch.name ?? null, patch.email ?? null],
    );
    return row ? toUser(row) : null;
  },

  async delete(id: string): Promise<boolean> {
    const r = await query(`DELETE FROM users WHERE id = $1`, [id]);
    return r.length > 0; // query returns rows for SELECT, 0 for DELETE; use result API
  },
};
```

For `delete`, `pg` returns a `result` object with `rowCount`:

```ts
async delete(id: string): Promise<boolean> {
  const result = await pool.query(`DELETE FROM users WHERE id = $1`, [id]);
  return (result.rowCount ?? 0) > 0;
},
```

---

## 9. Migrations — how the schema changes

For now, a simple `migrations/001_init.sql`:

```sql
-- migrations/001_init.sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE IF NOT EXISTS users (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email       TEXT NOT NULL UNIQUE,
  name        TEXT NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Run with:

```bash
psql "$DATABASE_URL" -f migrations/001_init.sql
```

In Mission 3 you will replace this with **Drizzle migrations** (or any other tool). For now, plain SQL is fine.

---

## 10. `DATABASE_URL` and `.env`

```bash
# .env (DO NOT COMMIT)
DATABASE_URL=postgres://postgres:postgres@localhost:5432/next_level
PORT=3000
NODE_ENV=development
```

```gitignore
# .gitignore
node_modules
dist
.env
```

Commit `.env.example` with placeholders so teammates know what to set.

---

## 11. Smoke-test the wired API

```bash
# Start the API
pnpm dev

# In another terminal
curl -X POST http://localhost:3000/api/v1/users \
  -H "content-type: application/json" \
  -d '{"email":"rumi@x.com","name":"Rumi"}'

curl http://localhost:3000/api/v1/users
```

You should see a JSON object that *survives a process restart*. You have a real backend.

---

## Exercise

1. Add a `projects` table and `projectsRepo` with the same CRUD shape as `usersRepo`.
2. Add a `GET /api/v1/users/:id/projects` endpoint that joins.
3. Write a `migrations/002_projects.sql` and a script that runs both migrations in order.
4. Add a `search(q: string)` method on `usersRepo` that uses `ILIKE` for case-insensitive search.
5. Add a Vitest integration test that spins up a test database, runs migrations, and exercises the API.

---

## Common mistakes

1. **One connection per request.** Always use a pool.
2. **String concatenation in SQL.** Always parameterize.
3. **Catching pool errors.** Listen for `'error'` and let the process restart in production.
4. **Forgetting the `RETURNING` clause.** Postgres can return the inserted/updated row in one round-trip — use it.
5. **Running migrations on every deploy by hand.** We'll automate this in Mission 3.

---

## AI Industry Note (2026)

- **Postgres is the AI backend's favorite database.** `pgvector` (Module 49) gives you a real production vector store with ACID, JOINs, and indexing. Most AI products in 2026 use Postgres for *both* their transactional data and their vector data. The moment you internalize `pg`, the moment you unlock the entire AI stack.
- **The pattern AI assistants reach for first:** the `query<T>` + `queryOne<T>` helper above. If your project has these two functions, the AI can write 90% of your repository code. **Set the pattern; the AI amplifies it.**
- **What to read after this module:** the official `pg` docs (short, dense, and good). Then skim the Postgres [`SELECT` tutorial](https://www.postgresqltutorial.com/) chapters on `WHERE`, `ORDER BY`, `LIMIT`, and `JOIN` — we use them in Mission 3.
