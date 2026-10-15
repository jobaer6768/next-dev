# Module 09 — Scalable Backend Architecture & Clean Code

> The architecture that turns "I can build a CRUD API" into "I can build a system that survives the next 18 months of feature requests."

---

## Why this module matters

Right now you have a CRUD API with a `usersRepo`. By the end of this course you will have a multi-tenant SaaS with workspaces, teams, projects, tasks, comments, file uploads, AI features, and a frontend. **The architecture you choose today decides whether that complexity is manageable or a swamp.**

This module teaches **layered, feature-modular architecture** — the pattern that 80% of serious TypeScript backends in 2026 use. You will:

- Refactor the CRUD API into `route → controller → service → repository`.
- Apply **SOLID** in a TypeScript-flavored way.
- Add **Zod validation** at the route boundary.
- Centralize error handling.
- Produce a folder layout you can scale.

---

## Learning objectives

- Explain why project structure matters.
- Apply separation of concerns at the route / controller / service / repository layers.
- Use SOLID principles in functional-style TypeScript.
- Validate requests with Zod and map errors to HTTP responses.
- Document an API.

---

## 1. Why project structure matters

A junior project looks like:

```
src/
├── index.ts
├── helpers.ts
└── utils.ts       (3,000 lines by month 4)
```

A senior project looks like:

```
src/
├── modules/
│   ├── users/
│   │   ├── users.controller.ts
│   │   ├── users.service.ts
│   │   ├── users.repository.ts
│   │   ├── users.schema.ts
│   │   ├── users.routes.ts
│   │   └── users.test.ts
│   └── projects/
├── lib/
│   ├── errors.ts
│   ├── asyncHandler.ts
│   ├── logger.ts
│   └── validate.ts
├── config/
├── db/
└── server.ts
```

**Each feature owns its files.** You can delete a folder and the rest of the app still builds. This is **modular monolith** — the dominant pattern in 2026.

---

## 2. Separation of concerns — the four layers

```
HTTP            Route             "What URL maps to what handler"
   ↓
                Controller        "Parse request, return response"
   ↓
                Service           "Business rules; pure-ish functions"
   ↓
                Repository        "How data is stored/retrieved"
   ↓
              Database / external API
```

### Routes

A routes file declares URLs and binds them to controllers. **It does not contain logic.**

```ts
// src/modules/users/users.routes.ts
import { Router } from "express";
import { usersController } from "./users.controller.js";
import { validate } from "../../lib/validate.js";
import { CreateUserSchema, UpdateUserSchema, ListUsersQuerySchema } from "./users.schema.js";

export const usersRouter = Router();

usersRouter.get("/",                validate(ListUsersQuerySchema, "query"), usersController.list);
usersRouter.post("/",               validate(CreateUserSchema, "body"),    usersController.create);
usersRouter.get("/:id",                                                  usersController.getById);
usersRouter.patch("/:id",            validate(UpdateUserSchema, "body"),   usersController.update);
usersRouter.delete("/:id",                                                 usersController.remove);
```

### Controllers

Controllers handle HTTP concerns: extract params, call the service, shape the response.

```ts
// src/modules/users/users.controller.ts
import type { Request, Response } from "express";
import { usersService } from "./users.service.js";
import { NotFoundError } from "../../lib/errors.js";

export const usersController = {
  async list(req: Request, res: Response) {
    const { page, limit } = req.query as unknown as { page: number; limit: number };
    const result = await usersService.list({ page, limit });
    res.json({ data: result.items, meta: { page: result.page, total: result.total } });
  },

  async getById(req: Request, res: Response) {
    const u = await usersService.findById(req.params.id);
    if (!u) throw new NotFoundError("user");
    res.json({ data: u });
  },

  async create(req: Request, res: Response) {
    const u = await usersService.create(req.body);
    res.status(201).json({ data: u });
  },

  async update(req: Request, res: Response) {
    const u = await usersService.update(req.params.id, req.body);
    if (!u) throw new NotFoundError("user");
    res.json({ data: u });
  },

  async remove(req: Request, res: Response) {
    const ok = await usersService.delete(req.params.id);
    if (!ok) throw new NotFoundError("user");
    res.status(204).end();
  },
};
```

### Services

Services contain business rules. They take plain inputs, return plain outputs, and **never touch `req` or `res`**. This makes them trivial to unit-test.

```ts
// src/modules/users/users.service.ts
import { usersRepo } from "./users.repository.js";
import type { CreateUserInput, UpdateUserInput, PublicUser } from "./users.types.js";

export const usersService = {
  async list({ page, limit }: { page: number; limit: number }) {
    const [items, total] = await Promise.all([
      usersRepo.list({ page, limit }),
      usersRepo.count(),
    ]);
    return { items: items.map(toPublic), page, total };
  },

  async findById(id: string): Promise<PublicUser | null> {
    const u = await usersRepo.findById(id);
    return u ? toPublic(u) : null;
  },

  async create(input: CreateUserInput): Promise<PublicUser> {
    const exists = await usersRepo.findByEmail(input.email);
    if (exists) throw new ConflictError("email already in use");
    return toPublic(await usersRepo.create(input));
  },

  async update(id: string, patch: UpdateUserInput): Promise<PublicUser | null> {
    const u = await usersRepo.update(id, patch);
    return u ? toPublic(u) : null;
  },

  async delete(id: string): Promise<boolean> {
    return usersRepo.delete(id);
  },
};

function toPublic(u: { id: string; email: string; name: string; createdAt: Date }): PublicUser {
  return { id: u.id, email: u.email, name: u.name, createdAt: u.createdAt };
}
```

### Repository

The repository owns the database. The service above doesn't know SQL exists.

```ts
// src/modules/users/users.repository.ts
import { query, queryOne } from "../../db/query.js";
import type { CreateUserInput, UpdateUserInput } from "./users.types.js";

type Row = { id: string; email: string; name: string; created_at: Date };

function fromRow(r: Row) {
  return { id: r.id, email: r.email, name: r.name, createdAt: r.created_at };
}

export const usersRepo = {
  async list({ page, limit }: { page: number; limit: number }) {
    const offset = (page - 1) * limit;
    return (await query<Row>(`SELECT id, email, name, created_at FROM users ORDER BY created_at DESC LIMIT $1 OFFSET $2`, [limit, offset])).map(fromRow);
  },
  async count() {
    const r = await queryOne<{ count: string }>(`SELECT COUNT(*)::text AS count FROM users`);
    return Number(r?.count ?? 0);
  },
  async findById(id: string) {
    const r = await queryOne<Row>(`SELECT id, email, name, created_at FROM users WHERE id = $1`, [id]);
    return r ? fromRow(r) : null;
  },
  async findByEmail(email: string) {
    const r = await queryOne<Row>(`SELECT id, email, name, created_at FROM users WHERE email = $1`, [email]);
    return r ? fromRow(r) : null;
  },
  async create(input: CreateUserInput) {
    const r = await queryOne<Row>(`INSERT INTO users (email, name) VALUES ($1,$2) RETURNING id,email,name,created_at`, [input.email, input.name]);
    if (!r) throw new Error("insert failed");
    return fromRow(r);
  },
  async update(id: string, patch: UpdateUserInput) {
    const r = await queryOne<Row>(`UPDATE users SET name=COALESCE($2,name), email=COALESCE($3,email) WHERE id=$1 RETURNING id,email,name,created_at`, [id, patch.name ?? null, patch.email ?? null]);
    return r ? fromRow(r) : null;
  },
  async delete(id: string) {
    const r = await query(`DELETE FROM users WHERE id = $1`, [id]);
    return r.length >= 0; // see pg result API
  },
};
```

### Why this layering pays off

- **Test the service without HTTP.** Mock the repo.
- **Swap Postgres for a fake in tests.** Mock the repo.
- **Change the URL shape without touching business logic.** Edit the route.
- **Add a `users.events.ts` for domain events** without touching the controller.

---

## 3. SOLID in functional TypeScript

| Principle | Translation in TS |
|-----------|------------------|
| **S**ingle responsibility | One module = one feature. One service = one set of related business rules. |
| **O**pen/closed | Add new strategies by passing functions, not by `extends` chains. |
| **L**iskov substitution | Type-safe interfaces — if a function expects `Repository<T>`, any compatible repo works. |
| **I**nterface segregation | Small, focused types. Prefer many specific types over one fat one. |
| **D**ependency inversion | Services depend on `Repository` *interfaces*; the concrete `pg` impl is injected. |

The repo *type* is a value, not a class. That makes mocking trivial:

```ts
// src/modules/users/users.types.ts
export interface UsersRepository {
  list(opts: { page: number; limit: number }): Promise<User[]>;
  count(): Promise<number>;
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  create(input: CreateUserInput): Promise<User>;
  update(id: string, patch: UpdateUserInput): Promise<User | null>;
  delete(id: string): Promise<boolean>;
}
```

The pg-backed `usersRepo` implements it. In tests you can pass a stub.

---

## 4. Zod validation as a request boundary

```ts
// src/lib/validate.ts
import type { Request, Response, NextFunction } from "express";
import { ZodSchema } from "zod";
import { ValidationError } from "./errors.js";

export function validate<T>(schema: ZodSchema<T>, source: "body" | "query" | "params") {
  return (req: Request, _res: Response, next: NextFunction) => {
    const data = source === "body" ? req.body : source === "query" ? req.query : req.params;
    const parsed = schema.safeParse(data);
    if (!parsed.success) {
      return next(new ValidationError(parsed.error.flatten()));
    }
    // attach parsed (typed) value
    (req as Request & { valid: Record<string, unknown> }).valid[source] = parsed.data;
    next();
  };
}
```

```ts
// src/modules/users/users.schema.ts
import { z } from "zod";

export const CreateUserSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
});
export type CreateUserInput = z.infer<typeof CreateUserSchema>;

export const UpdateUserSchema = CreateUserSchema.partial();
export type UpdateUserInput = z.infer<typeof UpdateUserSchema>;

export const ListUsersQuerySchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
});
```

Now the controller reads `req.valid.body` and gets a *type-safe* object — no more `as unknown as`.

---

## 5. Centralized error handling

```ts
// src/lib/errors.ts
export class HttpError extends Error {
  constructor(public status: number, public code: string, message: string, public details?: unknown) {
    super(message);
    this.name = code;
  }
}

export class NotFoundError extends HttpError {
  constructor(resource: string) {
    super(404, "NOT_FOUND", `${resource} not found`);
  }
}

export class ConflictError extends HttpError {
  constructor(message: string) { super(409, "CONFLICT", message); }
}

export class ValidationError extends HttpError {
  constructor(details: unknown) { super(400, "VALIDATION_ERROR", "validation failed", details); }
}

export class UnauthorizedError extends HttpError {
  constructor(message = "unauthorized") { super(401, "UNAUTHORIZED", message); }
}

export class ForbiddenError extends HttpError {
  constructor(message = "forbidden") { super(403, "FORBIDDEN", message); }
}
```

```ts
// src/lib/errorHandler.ts
import type { ErrorRequestHandler } from "express";
import { HttpError } from "./errors.js";
import { logger } from "./logger.js";

export const errorHandler: ErrorRequestHandler = (err, req, res, _next) => {
  if (err instanceof HttpError) {
    return res.status(err.status).json({
      error: { code: err.code, message: err.message, details: err.details },
    });
  }
  logger.error({ err, path: req.path, method: req.method }, "unhandled error");
  res.status(500).json({ error: { code: "INTERNAL", message: "internal error" } });
};
```

---

## 6. Logging — the boring but essential

Use **pino** in 2026. It's the fastest, smallest, JSON-structured logger in the Node ecosystem.

```bash
pnpm add pino
pnpm add -D pino-pretty
```

```ts
// src/lib/logger.ts
import pino from "pino";

export const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  transport: process.env.NODE_ENV === "production"
    ? undefined
    : { target: "pino-pretty", options: { translateTime: "HH:MM:ss.l", ignore: "pid,hostname" } },
});
```

Use it everywhere instead of `console.log`. Structured JSON logs are what production observability tools consume.

---

## 7. API documentation — the 5-minute version

For a small API, a `docs/API.md` is enough. For a larger one, generate OpenAPI from Zod with [`zod-to-openapi`](https://github.com/asteasolutions/zod-to-openapi) or use the [`@asteasolutions/zod-to-openapi`](https://github.com/asteasolutions/zod-to-openapi) + Swagger UI.

For this course we keep a hand-written `docs/API.md` for each module:

```md
## POST /api/v1/users
Create a new user.

### Body
| Field | Type | Required | Notes |
|-------|------|----------|-------|
| name  | string | yes | 1..100 chars |
| email | string | yes | valid email |

### Responses
- 201 — `{ data: User }`
- 400 — `{ error: { code: "VALIDATION_ERROR", details: ... } }`
- 409 — `{ error: { code: "CONFLICT" } }`
```

We will graduate to OpenAPI in Mission 4.

---

## 8. A feature-based folder layout (the one we'll use in Mission 4)

```
src/
├── server.ts                # http.createServer + graceful shutdown
├── app.ts                   # express() with all middleware wired
├── config/
│   └── env.ts               # Zod-validated process.env
├── db/
│   ├── pool.ts              # pg.Pool
│   ├── migrate.ts           # migration runner
│   └── migrations/          # *.sql files
├── lib/
│   ├── asyncHandler.ts
│   ├── errors.ts
│   ├── errorHandler.ts
│   ├── logger.ts
│   └── validate.ts
├── modules/
│   ├── users/
│   │   ├── users.controller.ts
│   │   ├── users.service.ts
│   │   ├── users.repository.ts
│   │   ├── users.routes.ts
│   │   ├── users.schema.ts
│   │   ├── users.types.ts
│   │   └── users.test.ts
│   └── projects/
│       └── ... (same shape)
└── types/
    └── index.ts             # shared types
```

**Each module is a folder, not a file.** You can `git mv modules/projects` to a new repo and the rest of the app doesn't notice.

---

## Exercise — refactor your CRUD

Take the in-memory CRUD API from Module 07 and refactor it into the layered structure above, with:

1. A real `pg` repository.
2. A service that returns `{ items, total, page }` from a paginated `list`.
3. Zod validation on every body and query string.
4. Centralized `HttpError` subclasses.
5. A single `errorHandler` registered last.
6. A `docs/API.md` documenting every endpoint.
7. Vitest unit tests for the service (mock the repo) and at least one integration test against a real Postgres (use `pnpm dlx testcontainers` for ephemeral DBs).

Run `pnpm dev`, hit the endpoints with Bruno, and commit a clean PR titled `[M2] Refactor to layered architecture`.

---

## Common mistakes

1. **Service imports `req` or `res`.** That's a controller's job. If your service knows about HTTP, it can't be reused or tested.
2. **Controller does business logic.** It should be 5 lines of "extract, call, respond."
3. **Repository returns DB row shapes.** Always map to a domain type.
4. **Validation scattered across the app.** One `validate()` middleware, used everywhere.
5. **Catch-all `try/catch` in the controller.** Throw, let `errorHandler` do it.
6. **Logging with `console.log`.** Use `pino`. The day you need to ship logs to a tool, you'll thank past-you.

---

## AI Industry Note (2026)

- **The "controller / service / repository" pattern is the lingua franca of TypeScript backends in 2026.** tRPC blurs the controller (it generates the type-safe client), and Clean Architecture purists add Use Cases and Entities. But *every* senior TS engineer you interview with will recognize and expect this structure. Make it muscle memory.
- **Zod-first schemas are the AI force multiplier.** When you write Zod schemas for your domain (CreateUser, UpdateUser, etc.), an AI assistant can generate *the entire* controller, service, and repository layer from those schemas. I've seen engineers ship a full CRUD module in 10 minutes this way. Set the schemas; let the AI fill in the boilerplate.
- **What hiring managers look for in 2026:** can the engineer explain the trade-off between "controller does too much" and "service does nothing"? Can they point to *one* line in their own code that demonstrates dependency inversion? If yes, they're senior. The layered architecture is the simplest way to *show* that understanding in code.
