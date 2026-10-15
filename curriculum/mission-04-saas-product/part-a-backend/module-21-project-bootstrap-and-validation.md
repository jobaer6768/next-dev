# Module 21 — Project Setup, Data Modeling & Request Validation

> "Slow down to speed up." A backend repository is not just code — it is a contract with every future engineer (including future you). Bootstrap it deliberately, and the next 50 modules get easier. Skip it, and you will fight your own framework forever.

---

## Why this matters

The first hour you spend on a backend repo decides the next six months. A clean folder structure, strict TypeScript, validated configuration, and a single global error handler are the difference between a codebase that scales and one that becomes a tarpit. Every senior engineer has opened a file like `utils2_final_FINAL.ts` and sworn to never be that person. This module is the antidote.

By the end of this module you will have a backend skeleton that:

- Refuses to start with bad configuration
- Returns consistent error envelopes regardless of where the failure happens
- Validates every byte that enters the system
- Logs structured JSON that you can ship to a real log pipeline
- Has a folder structure that any teammate can navigate without asking you

This is the foundation for everything else in Mission 04: auth, RBAC, multi-tenancy, billing, and the API surface your frontend will consume.

---

## Learning objectives

By the end of this module you can:

1. Bootstrap a Node 22 + TypeScript strict + Express 5 backend from scratch
2. Structure a backend repository so that features are isolated and replaceable
3. Validate environment variables with Zod and fail loudly on misconfiguration
4. Wire helmet, cors, morgan, pino, and `express.json` into a single bootstrap
5. Define a Drizzle schema and generate your first migration
6. Build a typed `HttpError` hierarchy and an `errorHandler` middleware
7. Write a generic `validate()` middleware for `body`, `query`, and `params`
8. Use a `catchAsync` wrapper to eliminate try/catch boilerplate
9. Define a consistent `{ data, meta, error }` response envelope
10. Separate `development`, `test`, and `production` environments

---

## 1. Repository layout

We use a feature-modular layout. Each feature is a vertical slice that owns its router, service, schema, and (later) tests.

```
.
├── .editorconfig
├── .env.example
├── .env                  # gitignored
├── .env.test             # gitignored
├── .gitignore
├── package.json
├── tsconfig.json
├── drizzle.config.ts
├── src/
│   ├── app.ts            # builds the Express app (used by tests)
│   ├── server.ts         # boots the HTTP listener
│   ├── config/
│   │   ├── env.ts        # Zod-validated process.env
│   │   └── logger.ts     # pino instance
│   ├── db/
│   │   ├── client.ts     # drizzle({ schema })
│   │   └── schema.ts     # Drizzle table definitions
│   ├── lib/
│   │   ├── http-error.ts # HttpError hierarchy
│   │   ├── error-handler.ts
│   │   ├── validate.ts   # Zod middleware
│   │   ├── catch-async.ts
│   │   └── response.ts   # envelope helpers
│   ├── modules/
│   │   └── health/
│   │       ├── health.router.ts
│   │       └── health.service.ts
│   └── types/
│       └── express.d.ts  # module augmentation for req.user etc.
├── drizzle/              # generated SQL migrations
└── README.md
```

**Rule of thumb:** if you have to scroll more than a screen to find a feature's router, your folder is too big. Split it.

---

## 2. Tooling baseline

### `package.json` scripts

```jsonc
{
  "name": "saas-backend",
  "private": true,
  "type": "module",
  "engines": { "node": ">=22" },
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "build": "tsc -p tsconfig.json",
    "start": "node dist/server.js",
    "typecheck": "tsc -p tsconfig.json --noEmit",
    "lint": "eslint .",
    "test": "vitest run",
    "test:watch": "vitest",
    "db:generate": "drizzle-kit generate",
    "db:migrate": "drizzle-kit migrate",
    "db:studio": "drizzle-kit studio"
  }
}
```

### `tsconfig.json` (strict, ESM)

```jsonc
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "declaration": false,
    "sourceMap": true
  },
  "include": ["src/**/*"]
}
```

### `.editorconfig`

```ini
root = true

[*]
indent_style = space
indent_size = 2
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true
```

### `.gitignore`

```gitignore
node_modules
dist
.env
.env.local
.env.*.local
*.log
.DS_Store
coverage
.vscode
.idea
```

### `.env.example`

```bash
# Server
NODE_ENV=development
PORT=4000
LOG_LEVEL=info

# Database
DATABASE_URL=postgres://postgres:postgres@localhost:5432/saas

# Auth (Module 22)
JWT_ACCESS_SECRET=change-me-please
JWT_REFRESH_SECRET=change-me-too

# CORS
CORS_ORIGIN=http://localhost:3000
```

---

## 3. Config layer — Zod-validated env

Never trust `process.env`. Parse it once at boot, then the rest of the app gets a fully typed object.

```ts
// src/config/env.ts
import { z } from "zod";

const EnvSchema = z.object({
  NODE_ENV: z
    .enum(["development", "test", "production"])
    .default("development"),
  PORT: z.coerce.number().int().positive().default(4000),
  LOG_LEVEL: z
    .enum(["fatal", "error", "warn", "info", "debug", "trace"])
    .default("info"),
  DATABASE_URL: z.string().url(),
  JWT_ACCESS_SECRET: z.string().min(32, "use a long random secret"),
  JWT_REFRESH_SECRET: z.string().min(32),
  CORS_ORIGIN: z.string().url(),
});

export type Env = z.infer<typeof EnvSchema>;

export const env: Env = (() => {
  const parsed = EnvSchema.safeParse(process.env);
  if (!parsed.success) {
    // Print every issue and crash. No silent defaults.
    console.error("Invalid environment configuration:");
    console.error(parsed.error.flatten().fieldErrors);
    process.exit(1);
  }
  return parsed.data;
})();
```

Now `env.PORT` is `number`, not `string | undefined`. The compiler will catch typos.

### `pino` logger

```ts
// src/config/logger.ts
import pino from "pino";
import { env } from "./env.js";

export const logger = pino({
  level: env.LOG_LEVEL,
  base: { service: "saas-backend", env: env.NODE_ENV },
  // In dev, pretty-print; in prod, ship raw JSON.
  transport:
    env.NODE_ENV === "development"
      ? { target: "pino-pretty", options: { colorize: true } }
      : undefined,
});
```

---

## 4. Express bootstrap

```ts
// src/app.ts
import express, { type Application } from "express";
import helmet from "helmet";
import cors from "cors";
import pinoHttp from "pino-http";
import { randomUUID } from "node:crypto";
import { env } from "./config/env.js";
import { logger } from "./config/logger.js";
import { errorHandler, notFoundHandler } from "./lib/error-handler.js";
import { healthRouter } from "./modules/health/health.router.js";

export function buildApp(): Application {
  const app = express();

  // Trust the local proxy if you ever sit behind one (nginx, fly, render)
  app.set("trust proxy", 1);

  app.use(helmet());
  app.use(
    cors({
      origin: env.CORS_ORIGIN,
      credentials: true,
    }),
  );
  app.use(express.json({ limit: "1mb" }));
  app.use(express.urlencoded({ extended: true }));

  // Request id + structured access log
  app.use(
    pinoHttp({
      logger,
      genReqId: (req, res) => {
        const id =
          (req.headers["x-request-id"] as string | undefined) ?? randomUUID();
        res.setHeader("x-request-id", id);
        return id;
      },
      customLogLevel: (_req, res, err) => {
        if (err || res.statusCode >= 500) return "error";
        if (res.statusCode >= 400) return "warn";
        return "info";
      },
    }),
  );

  // Feature routers
  app.use("/health", healthRouter);

  // 404
  app.use(notFoundHandler);

  // Global error handler MUST be last
  app.use(errorHandler);

  return app;
}
```

```ts
// src/server.ts
import { buildApp } from "./app.js";
import { env } from "./config/env.js";
import { logger } from "./config/logger.js";

const app = buildApp();
const server = app.listen(env.PORT, () => {
  logger.info({ port: env.PORT }, "saas-backend listening");
});

// Graceful shutdown
const shutdown = (signal: NodeJS.Signals) => {
  logger.info({ signal }, "shutting down");
  server.close(() => process.exit(0));
};
process.on("SIGINT", shutdown);
process.on("SIGTERM", shutdown);
```

```ts
// src/modules/health/health.router.ts
import { Router } from "express";

export const healthRouter: Router = Router();

healthRouter.get("/", (_req, res) => {
  res.json({ status: "ok", uptime: process.uptime() });
});
```

---

## 5. Drizzle schema (first table + first migration)

We keep this brief — Module 18/19 covers Drizzle in depth. The point here is to get **one** table so migrations work end-to-end.

```ts
// src/db/schema.ts
import {
  pgTable,
  uuid,
  text,
  timestamp,
} from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: uuid("id").defaultRandom().primaryKey(),
  email: text("email").notNull().unique(),
  passwordHash: text("password_hash").notNull(),
  status: text("status").notNull().default("active"), // active | blocked | deleted
  createdAt: timestamp("created_at", { withTimezone: true })
    .notNull()
    .defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true })
    .notNull()
    .defaultNow(),
});
```

```ts
// src/db/client.ts
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import { env } from "../config/env.js";
import * as schema from "./schema.js";

export const pool = new Pool({ connectionString: env.DATABASE_URL });
export const db = drizzle(pool, { schema });
export type Database = typeof db;
```

```ts
// drizzle.config.ts
import { defineConfig } from "drizzle-kit";
import "dotenv/config";

export default defineConfig({
  schema: "./src/db/schema.ts",
  out: "./drizzle",
  dialect: "postgresql",
  dbCredentials: { url: process.env.DATABASE_URL! },
  strict: true,
});
```

```bash
pnpm db:generate   # writes drizzle/0000_init.sql
pnpm db:migrate    # applies it
```

---

## 6. The `HttpError` hierarchy

Throw typed errors. Catch them in one place. Never sprinkle `res.status(500).send(...)` across the codebase.

```ts
// src/lib/http-error.ts
export class HttpError extends Error {
  public readonly status: number;
  public readonly code: string;
  public readonly details?: unknown;

  constructor(status: number, code: string, message: string, details?: unknown) {
    super(message);
    this.name = this.constructor.name;
    this.status = status;
    this.code = code;
    this.details = details;
  }
}

export class ValidationError extends HttpError {
  constructor(message = "Invalid request", details?: unknown) {
    super(400, "VALIDATION_ERROR", message, details);
  }
}
export class UnauthorizedError extends HttpError {
  constructor(message = "Authentication required") {
    super(401, "UNAUTHORIZED", message);
  }
}
export class ForbiddenError extends HttpError {
  constructor(message = "You do not have access to this resource") {
    super(403, "FORBIDDEN", message);
  }
}
export class NotFoundError extends HttpError {
  constructor(message = "Resource not found") {
    super(404, "NOT_FOUND", message);
  }
}
export class ConflictError extends HttpError {
  constructor(message = "Resource already exists") {
    super(409, "CONFLICT", message);
  }
}
export class InternalError extends HttpError {
  constructor(message = "Internal server error") {
    super(500, "INTERNAL_ERROR", message);
  }
}
```

Notice: every error has a stable `code` string. Clients can switch on `code`, not on localized `message`.

---

## 7. Global error handler

```ts
// src/lib/error-handler.ts
import type { ErrorRequestHandler, RequestHandler } from "express";
import { ZodError } from "zod";
import { HttpError, InternalError, ValidationError } from "./http-error.js";
import { logger } from "../config/logger.js";
import { env } from "../config/env.js";

export const notFoundHandler: RequestHandler = (req, _res, next) => {
  next(new (class extends HttpError {
    constructor() {
      super(404, "ROUTE_NOT_FOUND", `No route for ${req.method} ${req.path}`);
    }
  })());
};

export const errorHandler: ErrorRequestHandler = (err, req, res, _next) => {
  // 1. Normalize Zod failures into a 400
  if (err instanceof ZodError) {
    err = new ValidationError("Request failed schema validation", err.flatten());
  }

  // 2. Normalize anything we did not expect
  if (!(err instanceof HttpError)) {
    logger.error({ err }, "unhandled error");
    err = new InternalError(env.NODE_ENV === "production" ? "Internal server error" : (err as Error).message);
  }

  const httpErr = err as HttpError;
  // 3. Log operational errors at warn, programmer errors at error
  if (httpErr.status >= 500) {
    logger.error({ err: httpErr, reqId: req.id }, httpErr.message);
  } else {
    logger.warn({ err: httpErr, reqId: req.id }, httpErr.message);
  }

  res.status(httpErr.status).json({
    data: null,
    meta: null,
    error: {
      code: httpErr.code,
      message: httpErr.message,
      details: httpErr.details,
      requestId: req.id,
    },
  });
};
```

**Important:** every error leaves the API in the same shape. Your frontend can rely on it.

---

## 8. Response envelope

```ts
// src/lib/response.ts
import type { Response } from "express";

export interface Envelope<T> {
  data: T | null;
  meta: Record<string, unknown> | null;
  error: { code: string; message: string; details?: unknown; requestId?: string } | null;
}

export function ok<T>(res: Response, data: T, meta?: Record<string, unknown>) {
  const body: Envelope<T> = {
    data,
    meta: meta ?? null,
    error: null,
  };
  return res.json(body);
}
```

Use it everywhere:

```ts
import { ok } from "../../lib/response.js";

usersRouter.get("/", async (_req, res) => {
  const all = await db.select().from(users);
  return ok(res, all, { count: all.length });
});
```

A response is **always**:

```json
{
  "data":  { ... } | null,
  "meta":  { ... } | null,
  "error": null | { "code": "...", "message": "...", "details": {...}, "requestId": "..." }
}
```

Exactly one of `data` or `error` is non-null. Never both. Never neither.

---

## 9. The `validate()` middleware

Zod schemas per route. Three flavors: `body`, `query`, `params`. We replace the inputs with the parsed, typed value so downstream code sees clean data.

```ts
// src/lib/validate.ts
import type { RequestHandler } from "express";
import type { ZodSchema } from "zod";

type Source = "body" | "query" | "params";

export const validate =
  (schema: ZodSchema, source: Source = "body"): RequestHandler =>
  (req, _res, next) => {
    const result = schema.safeParse(req[source]);
    if (!result.success) {
      return next(result.error); // errorHandler converts to 400
    }
    // Override the request input with the parsed (typed) value
    (req as unknown as Record<Source, unknown>)[source] = result.data;
    next();
  };
```

Example usage:

```ts
// src/modules/users/users.schemas.ts
import { z } from "zod";

export const createUserSchema = z.object({
  body: z.object({
    email: z.string().email(),
    password: z.string().min(10).max(128),
  }),
});

export const getUserSchema = z.object({
  params: z.object({
    id: z.string().uuid(),
  }),
});

// Types come free
export type CreateUserInput = z.infer<typeof createUserSchema.shape.body>;
export type GetUserParams = z.infer<typeof getUserSchema.shape.params>;
```

```ts
// src/modules/users/users.router.ts
import { Router } from "express";
import { validate } from "../../lib/validate.js";
import { createUserSchema, getUserSchema } from "./users.schemas.js";
import { usersService } from "./users.service.js";
import { ok } from "../../lib/response.js";

export const usersRouter: Router = Router();

usersRouter.post(
  "/",
  validate(createUserSchema.shape.body),
  async (req, res) => {
    const input = req.body as CreateUserInput;
    const user = await usersService.create(input);
    return ok(res, user);
  },
);

usersRouter.get(
  "/:id",
  validate(getUserSchema.shape.params),
  async (req, res) => {
    const { id } = req.params as unknown as GetUserParams;
    const user = await usersService.findById(id);
    return ok(res, user);
  },
);
```

Notice: by the time the handler runs, `req.body` and `req.params` are already validated and typed. No defensive parsing in the handler.

---

## 10. `catchAsync` wrapper

Express 4 does not forward rejections from async handlers. Express 5 finally does — but writing the wrapper is still cleaner and explicit.

```ts
// src/lib/catch-async.ts
import type { RequestHandler } from "express";

type AsyncHandler = (
  req: Parameters<RequestHandler>[0],
  res: Parameters<RequestHandler>[1],
  next: Parameters<RequestHandler>[2],
) => Promise<unknown>;

export const catchAsync =
  (fn: AsyncHandler): RequestHandler =>
  (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
```

Now routes read like ordinary async functions:

```ts
usersRouter.get(
  "/",
  catchAsync(async (_req, res) => {
    const all = await usersService.list();
    return ok(res, all);
  }),
);
```

If `usersService.list()` throws, the rejection flows into `next(err)` and the global handler turns it into a clean JSON envelope.

---

## 11. Environment separation

Three modes, three config files:

- `.env` — local development
- `.env.test` — loaded by `vitest` via `dotenv/config`
- `.env.production` — never committed; set in your host's secret store

In `src/config/env.ts` we already constrain `NODE_ENV` to those three values. In CI we run:

```bash
NODE_ENV=test pnpm test
NODE_ENV=production pnpm start
```

A common pattern: in `vitest.config.ts`, force-load `.env.test` first.

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";
import "dotenv/config";

export default defineConfig({
  test: {
    setupFiles: ["./src/test/setup.ts"],
    env: { NODE_ENV: "test" },
  },
});
```

---

## Common mistakes

1. **Skipping Zod on env vars.** You ship to staging, the secret is missing, the server starts with `undefined` and crashes two hours later. Validate at boot.
2. **Throwing raw strings or `new Error()` in services.** The error handler has no status code to use. Throw `HttpError` subclasses.
3. **Returning `{ error: ... }` with HTTP 200.** Frontend devs have to check both `ok` and `error`. Use the right status code.
4. **Forgetting `next(err)` in `catchAsync` rejections.** The promise resolves "successfully" with an unhandled rejection; the client hangs.
5. **Mixing response shapes.** Sometimes `{ user }`, sometimes `{ data: { user } }`, sometimes `[...]`. Pick an envelope and enforce it in code review.
6. **Logging secrets.** `pino` will happily log your `req.body` if you spread it. Whitelist fields. Rotate any secret that hits a log.
7. **Forgetting `trust proxy`.** When you finally deploy behind nginx, your IP-based rate limiting sees `127.0.0.1` for every request.
8. **No request id.** When a user reports "the API broke", you have nothing to grep. Generate one early, return it in the response and every log line.

---

## Exercise

**Goal:** Build a `tasks` feature with full validation and a tested error path.

1. Add a `tasks` table to `src/db/schema.ts` with: `id` (uuid), `title` (text), `done` (boolean), `ownerId` (uuid fk → users), `createdAt`. Generate and run the migration.
2. Create `src/modules/tasks/` with `tasks.router.ts`, `tasks.service.ts`, `tasks.schemas.ts`.
3. Define Zod schemas for `POST /tasks` (body) and `GET /tasks` (query with optional `done` boolean and `limit` 1..100).
4. Implement `tasksService.create` and `tasksService.list`. Have `create` throw `ValidationError` if `title.trim().length === 0`.
5. Wire the router into `app.ts` at `/tasks`.
6. Write a Vitest test that:
   - Posts a valid task and expects `200` with `data.title === "Buy milk"`.
   - Posts an empty title and expects `400` with `error.code === "VALIDATION_ERROR"`.
   - Sends a malformed JSON body and expects `400`.

**Bonus:** add a `meta.pagination` block in `list` that returns `{ limit, returned, hasMore }`.

---

## AI Industry Note (2026)

In 2026, AI coding assistants will happily scaffold a backend repo for you in under a minute — including the `package.json`, the `tsconfig`, the folder structure, even the Zod schemas. The bottleneck is no longer "how do I get a server running"; it is **judgment**.

What separates a senior engineer from a competent one in 2026 is not the ability to type fast or to remember API signatures. It is the ability to look at a generated scaffold and say:

- "This error handler leaks stack traces in production — fix it."
- "This CORS config will break when we add a second frontend origin."
- "This `validate()` middleware runs after the body parser has already allocated the buffer — I should add a size limit *before* parsing."
- "This logger includes `req.body` — strip the password field before logging."

The machine can write the first draft. The senior engineer owns the *shape* of the system: the contracts, the failure modes, the operational characteristics. Treat AI as a very fast junior who needs a strong tech lead. Your job is the tech lead. The files in this module — the env validator, the error hierarchy, the envelope, the `validate()` middleware — are the artifacts a senior engineers *insist* on, regardless of who or what wrote the first draft.

If you ever see yourself deleting these in the name of "speed", remember: every production outage I have ever debugged was downstream of one of these four files being sloppy.
