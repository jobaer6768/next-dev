# Project 2 — Express + PostgreSQL Mini API (Layered)

> A real backend, the way a senior would build it: route → controller → service → repository, with Zod validation, centralized errors, and a search + summary endpoint.

---

## Goal

Build either the **Expense Tracker** or the **Book Catalogue** API from Assignment 02. This file is the guide for the **Expense Tracker** (the most-common choice). Adapt names if you pick Books.

---

## Stack

```bash
pnpm init
pnpm add express cors helmet morgan pino pg zod
pnpm add -D typescript tsx @types/express @types/cors @types/morgan @types/pg @types/node vitest supertest @types/supertest
```

`tsconfig.json` — same as Module 09.

---

## Folder layout

```
src/
├── app.ts                 # express() with middleware wired
├── server.ts              # http.createServer + graceful shutdown
├── config/
│   └── env.ts             # Zod-validated process.env
├── db/
│   ├── pool.ts            # pg.Pool
│   ├── migrate.ts         # migration runner
│   └── migrations/
│       ├── 001_categories.sql
│       └── 002_expenses.sql
├── lib/
│   ├── asyncHandler.ts
│   ├── errors.ts
│   ├── errorHandler.ts
│   ├── logger.ts
│   └── validate.ts
├── modules/
│   ├── categories/
│   │   ├── categories.controller.ts
│   │   ├── categories.service.ts
│   │   ├── categories.repository.ts
│   │   ├── categories.routes.ts
│   │   ├── categories.schema.ts
│   │   ├── categories.types.ts
│   │   └── categories.test.ts
│   └── expenses/
│       ├── (same shape)
│       └── ...
└── types/
```

---

## Migrations

`migrations/001_categories.sql`:

```sql
CREATE TABLE categories (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name        TEXT NOT NULL UNIQUE,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

`migrations/002_expenses.sql`:

```sql
CREATE TABLE expenses (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  amount      NUMERIC(10,2) NOT NULL CHECK (amount > 0),
  category_id UUID NOT NULL REFERENCES categories(id) ON DELETE RESTRICT,
  note        TEXT,
  spent_at    DATE NOT NULL DEFAULT CURRENT_DATE,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX expenses_spent_at_idx ON expenses (spent_at);
CREATE INDEX expenses_category_idx  ON expenses (category_id);
```

`db/migrate.ts`:

```ts
import { readdir, readFile } from "node:fs/promises";
import path from "node:path";
import { pool } from "./pool.js";
import { logger } from "../lib/logger.js";

const MIGRATIONS_DIR = path.resolve(process.cwd(), "src/db/migrations");

async function ensureTable() {
  await pool.query(`
    CREATE TABLE IF NOT EXISTS _migrations (
      filename TEXT PRIMARY KEY,
      applied_at TIMESTAMPTZ NOT NULL DEFAULT now()
    );
  `);
}

async function applied(): Promise<Set<string>> {
  const r = await pool.query<{ filename: string }>(`SELECT filename FROM _migrations`);
  return new Set(r.rows.map((row) => row.filename));
}

async function run() {
  await ensureTable();
  const done = await applied();
  const files = (await readdir(MIGRATIONS_DIR)).filter((f) => f.endsWith(".sql")).sort();
  for (const f of files) {
    if (done.has(f)) continue;
    logger.info({ file: f }, "applying migration");
    const sql = await readFile(path.join(MIGRATIONS_DIR, f), "utf8");
    const client = await pool.connect();
    try {
      await client.query("BEGIN");
      await client.query(sql);
      await client.query("INSERT INTO _migrations (filename) VALUES ($1)", [f]);
      await client.query("COMMIT");
    } catch (e) {
      await client.query("ROLLBACK");
      throw e;
    } finally {
      client.release();
    }
  }
  logger.info("migrations done");
  await pool.end();
}

run().catch((e) => {
  console.error(e);
  process.exit(1);
});
```

`pnpm db:migrate` runs this.

---

## Schemas (Zod)

```ts
// src/modules/expenses/expenses.schema.ts
import { z } from "zod";

export const CreateExpense = z.object({
  amount: z.coerce.number().positive().multipleOf(0.01),
  categoryId: z.string().uuid(),
  note: z.string().max(500).optional(),
  spentAt: z.coerce.date().optional(),
});
export type CreateExpenseInput = z.infer<typeof CreateExpense>;

export const UpdateExpense = CreateExpense.partial();
export type UpdateExpenseInput = z.infer<typeof UpdateExpense>;

export const ListExpensesQuery = z.object({
  from: z.coerce.date().optional(),
  to:   z.coerce.date().optional(),
  categoryId: z.string().uuid().optional(),
  page:  z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
});
export type ListExpensesQuery = z.infer<typeof ListExpensesQuery>;

export const SummaryQuery = z.object({
  month: z.string().regex(/^\d{4}-\d{2}$/),
});
```

---

## Repository

```ts
// src/modules/expenses/expenses.repository.ts
import { query, queryOne } from "../../db/query.js";
import type { CreateExpenseInput, UpdateExpenseInput } from "./expenses.schema.js";

type Row = {
  id: string;
  amount: string;            // pg returns numeric as string
  category_id: string;
  note: string | null;
  spent_at: Date;
  created_at: Date;
};

function fromRow(r: Row) {
  return {
    id: r.id,
    amount: Number(r.amount),
    categoryId: r.category_id,
    note: r.note,
    spentAt: r.spent_at,
    createdAt: r.created_at,
  };
}

export const expensesRepo = {
  async list(q: { from?: Date; to?: Date; categoryId?: string; page: number; limit: number }) {
    const where: string[] = [];
    const params: unknown[] = [];
    if (q.from)      { params.push(q.from);      where.push(`spent_at >= $${params.length}`); }
    if (q.to)        { params.push(q.to);        where.push(`spent_at <= $${params.length}`); }
    if (q.categoryId){ params.push(q.categoryId);where.push(`category_id = $${params.length}`); }
    const whereSql = where.length ? `WHERE ${where.join(" AND ")}` : "";
    const offset = (q.page - 1) * q.limit;
    params.push(q.limit, offset);
    const sql = `
      SELECT id, amount, category_id, note, spent_at, created_at
      FROM expenses
      ${whereSql}
      ORDER BY spent_at DESC, created_at DESC
      LIMIT $${params.length - 1} OFFSET $${params.length}
    `;
    return (await query<Row>(sql, params)).map(fromRow);
  },

  async count(q: { from?: Date; to?: Date; categoryId?: string }) {
    const where: string[] = [];
    const params: unknown[] = [];
    if (q.from)      { params.push(q.from);      where.push(`spent_at >= $${params.length}`); }
    if (q.to)        { params.push(q.to);        where.push(`spent_at <= $${params.length}`); }
    if (q.categoryId){ params.push(q.categoryId);where.push(`category_id = $${params.length}`); }
    const whereSql = where.length ? `WHERE ${where.join(" AND ")}` : "";
    const r = await queryOne<{ c: string }>(`SELECT COUNT(*)::text AS c FROM expenses ${whereSql}`, params);
    return Number(r?.c ?? 0);
  },

  async findById(id: string) {
    const r = await queryOne<Row>(`SELECT id, amount, category_id, note, spent_at, created_at FROM expenses WHERE id = $1`, [id]);
    return r ? fromRow(r) : null;
  },

  async create(input: CreateExpenseInput) {
    const r = await queryOne<Row>(
      `INSERT INTO expenses (amount, category_id, note, spent_at)
       VALUES ($1, $2, $3, COALESCE($4, CURRENT_DATE))
       RETURNING id, amount, category_id, note, spent_at, created_at`,
      [input.amount, input.categoryId, input.note ?? null, input.spentAt ?? null],
    );
    if (!r) throw new Error("insert failed");
    return fromRow(r);
  },

  async update(id: string, patch: UpdateExpenseInput) {
    const r = await queryOne<Row>(
      `UPDATE expenses
         SET amount      = COALESCE($2, amount),
             category_id = COALESCE($3, category_id),
             note        = COALESCE($4, note),
             spent_at    = COALESCE($5, spent_at)
       WHERE id = $1
       RETURNING id, amount, category_id, note, spent_at, created_at`,
      [id, patch.amount ?? null, patch.categoryId ?? null, patch.note ?? null, patch.spentAt ?? null],
    );
    return r ? fromRow(r) : null;
  },

  async delete(id: string) {
    const r = await query(`DELETE FROM expenses WHERE id = $1`, [id]);
    return r.length >= 0; // or use result.rowCount
  },

  // Summary: group by category for a month
  async summaryByCategory(month: string) {
    const r = await query<{ category_id: string; name: string; total: string; count: string }>(
      `SELECT c.id AS category_id, c.name,
              SUM(e.amount)::text AS total,
              COUNT(*)::text AS count
         FROM expenses e
         JOIN categories c ON c.id = e.category_id
        WHERE to_char(e.spent_at, 'YYYY-MM') = $1
        GROUP BY c.id, c.name
        ORDER BY SUM(e.amount) DESC`,
      [month],
    );
    return r.map((row) => ({
      categoryId: row.category_id,
      name: row.name,
      total: Number(row.total),
      count: Number(row.count),
    }));
  },
};
```

---

## Service

```ts
// src/modules/expenses/expenses.service.ts
import { expensesRepo } from "./expenses.repository.js";
import { categoriesRepo } from "../categories/categories.repository.js";
import { NotFoundError, ValidationError } from "../../lib/errors.js";
import type { CreateExpenseInput, UpdateExpenseInput } from "./expenses.schema.js";

export const expensesService = {
  async list(q: { from?: Date; to?: Date; categoryId?: string; page: number; limit: number }) {
    const [items, total] = await Promise.all([
      expensesRepo.list(q),
      expensesRepo.count(q),
    ]);
    return { items, total, page: q.page, limit: q.limit };
  },

  async findById(id: string) {
    const e = await expensesRepo.findById(id);
    if (!e) throw new NotFoundError("expense");
    return e;
  },

  async create(input: CreateExpenseInput) {
    const cat = await categoriesRepo.findById(input.categoryId);
    if (!cat) throw new ValidationError({ categoryId: "unknown" });
    return expensesRepo.create(input);
  },

  async update(id: string, patch: UpdateExpenseInput) {
    const e = await expensesRepo.update(id, patch);
    if (!e) throw new NotFoundError("expense");
    return e;
  },

  async delete(id: string) {
    const ok = await expensesRepo.delete(id);
    if (!ok) throw new NotFoundError("expense");
  },

  async summary(month: string) {
    if (!/^\d{4}-\d{2}$/.test(month)) throw new ValidationError({ month: "must be YYYY-MM" });
    return expensesRepo.summaryByCategory(month);
  },
};
```

---

## Controller + routes

```ts
// src/modules/expenses/expenses.controller.ts
import type { Request, Response } from "express";
import { expensesService } from "./expenses.service.js";
import { asyncHandler } from "../../lib/asyncHandler.js";
import { validate } from "../../lib/validate.js";
import { CreateExpense, UpdateExpense, ListExpensesQuery, SummaryQuery } from "./expenses.schema.js";

export const expensesController = {
  list: [
    validate(ListExpensesQuery, "query"),
    asyncHandler(async (req: Request, res: Response) => {
      const data = res.locals.valid.query as any; // or extend Express Request
      const result = await expensesService.list(data);
      res.json({ data: result.items, meta: { page: result.page, limit: result.limit, total: result.total } });
    }),
  ],
  getById: asyncHandler(async (req, res) => {
    res.json({ data: await expensesService.findById(req.params.id) });
  }),
  create: [
    validate(CreateExpense, "body"),
    asyncHandler(async (req, res) => {
      res.status(201).json({ data: await expensesService.create(res.locals.valid.body) });
    }),
  ],
  update: [
    validate(UpdateExpense, "body"),
    asyncHandler(async (req, res) => {
      res.json({ data: await expensesService.update(req.params.id, res.locals.valid.body) });
    }),
  ],
  remove: asyncHandler(async (req, res) => {
    await expensesService.delete(req.params.id);
    res.status(204).end();
  }),
  summary: [
    validate(SummaryQuery, "query"),
    asyncHandler(async (req, res) => {
      res.json({ data: await expensesService.summary(res.locals.valid.query.month) });
    }),
  ],
};
```

(For production, prefer attaching `parsed` onto `res.locals.valid` rather than mutating `req`.)

```ts
// src/modules/expenses/expenses.routes.ts
import { Router } from "express";
import { expensesController } from "./expenses.controller.js";

export const expensesRouter = Router();
expensesRouter.get("/summary", ...expensesController.summary);
expensesRouter.get("/", ...expensesController.list);
expensesRouter.post("/", ...expensesController.create);
expensesRouter.get("/:id", expensesController.getById);
expensesRouter.patch("/:id", ...expensesController.update);
expensesRouter.delete("/:id", expensesController.remove);
```

---

## Test

```ts
// src/modules/expenses/expenses.test.ts
import { describe, it, expect } from "vitest";
import { expensesService } from "./expenses.service.js";

describe("expensesService.summary", () => {
  it("rejects malformed month", async () => {
    await expect(expensesService.summary("2026-1")).rejects.toThrow();
  });
  it("rejects missing month", async () => {
    await expect(expensesService.summary("not-a-date")).rejects.toThrow();
  });
});
```

For integration tests against a real DB, use `pg-mem` or `testcontainers`.

---

## Acceptance

- `pnpm db:migrate` creates the tables.
- `pnpm dev` starts the server.
- All CRUD endpoints work via Bruno.
- Search/filter by date range and category works.
- `GET /api/v1/expenses/summary?month=2026-01` returns per-category totals.
- Errors are consistent: `400 VALIDATION_ERROR`, `404 NOT_FOUND`, `500 INTERNAL`.
- README documents how to run, migrate, and test.

When you finish, you have completed Assignment 2 and are ready for **Mission 3 — Design & Optimize Databases**.
