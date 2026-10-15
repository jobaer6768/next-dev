# Module 07 — Express Fundamentals & Your First REST API

> The framework that took over the Node world — and the patterns every backend in 2026 still uses.

---

## Why this module matters

In Module 06 you built a server with raw `http.createServer`. By the end of that exercise you had a 200-line file full of `if (pathname === ...)` chains. Express replaces all of that with declarative routing, middleware composition, and a thriving ecosystem.

Express is *not* the newest framework (Fastify, Hono, and Elysia are nipping at its heels in 2026), but it is still the **most deployed** and the easiest to read in a code review. Every concept you learn here — routing, middleware, error handling — applies to every other Node framework.

---

## Learning objectives

- Set up an Express + TypeScript project from scratch.
- Use routing, route parameters, and query strings.
- Compose built-in and third-party middleware.
- Implement a CRUD API with consistent status codes and response shapes.
- Handle errors with custom error classes and a global error handler.
- Test your API with Postman or Bruno.

---

## 1. Setup

```bash
pnpm init
pnpm add express cors helmet morgan
pnpm add -D typescript tsx @types/express @types/cors @types/morgan
npx tsc --init
```

Adjust `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "noUncheckedIndexedAccess": true
  },
  "include": ["src/**/*"]
}
```

`package.json` scripts:

```json
"scripts": {
  "dev": "tsx watch src/index.ts",
  "build": "tsc",
  "start": "node dist/index.js"
}
```

---

## 2. The smallest possible Express app

```ts
// src/index.ts
import express from "express";
import { config } from "./config.js";

const app = express();

app.use(express.json());
app.use(express.urlencoded({ extended: true }));

app.get("/health", (_req, res) => {
  res.json({ status: "ok" });
});

app.listen(config.port, () => {
  console.log(`Listening on http://localhost:${config.port}`);
});
```

---

## 3. `req` and `res` — what's actually on them

```ts
app.post("/users", (req, res) => {
  const { name, email } = req.body as { name?: string; email?: string };
  const id = req.params.id;                  // route params
  const filter = req.query.filter as string; // query string
  const auth = req.headers.authorization;    // headers
  const ip = req.ip;                         // client IP
  res.status(201).json({ id, name, email });
});
```

`req` is `IncomingMessage` + Express's `Request`. `res` is `ServerResponse` + Express's `Response`. They are augmented with `body`, `params`, `query`, `cookies` by middleware.

---

## 4. Routing

```ts
app.get("/users", listUsers);
app.get("/users/:id", getUser);
app.post("/users", createUser);
app.put("/users/:id", replaceUser);
app.patch("/users/:id", updateUser);
app.delete("/users/:id", deleteUser);

// Chained handlers
app.route("/posts/:id")
  .get(getPost)
  .put(updatePost)
  .delete(deletePost);
```

### Route parameters

```ts
app.get("/users/:userId/orders/:orderId", (req, res) => {
  const { userId, orderId } = req.params;
  res.json({ userId, orderId });
});
```

### Query strings

```ts
app.get("/users", (req, res) => {
  const { page = "1", limit = "20" } = req.query as Record<string, string>;
  res.json({ page: Number(page), limit: Number(limit) });
});
```

---

## 5. Middleware

Middleware is the **heart** of Express. A middleware is just a function with the shape:

```ts
type Middleware = (req: Request, res: Response, next: NextFunction) => void | Promise<void>;
```

`next()` calls the next middleware. `next(err)` jumps to the error handler.

```ts
// src/middleware/requestId.ts
import { randomUUID } from "node:crypto";
import type { Request, Response, NextFunction } from "express";

export function requestId(req: Request, res: Response, next: NextFunction) {
  const id = req.headers["x-request-id"] ?? randomUUID();
  res.setHeader("x-request-id", String(id));
  (req as Request & { id: string }).id = String(id);
  next();
}
```

### Built-in middleware

```ts
app.use(express.json());             // parse JSON bodies
app.use(express.urlencoded({ extended: true }));
app.use(express.static("public"));   // serve files
```

### Third-party

```ts
import cors from "cors";
import helmet from "helmet";
import morgan from "morgan";

app.use(helmet());                // security headers
app.use(cors({ origin: "http://localhost:3000", credentials: true }));
app.use(morgan("dev"));           // request logging
```

### Custom middleware

```ts
app.use((req, _res, next) => {
  console.log(`[${new Date().toISOString()}] ${req.method} ${req.url}`);
  next();
});
```

### Mounting sub-apps

```ts
const apiRouter = express.Router();
apiRouter.get("/users", listUsers);
apiRouter.post("/users", createUser);

app.use("/api/v1", apiRouter);   // -> /api/v1/users
```

Use routers to organize by feature. We will use this in Mission 4.

---

## 6. Error handling

A middleware with **four** parameters is an error handler. It must be registered **last**.

```ts
// src/lib/errors.ts
export class HttpError extends Error {
  constructor(public status: number, message: string, public details?: unknown) {
    super(message);
    this.name = "HttpError";
  }
}

export class NotFoundError extends HttpError {
  constructor(resource: string) { super(404, `${resource} not found`); }
}

export class ValidationError extends HttpError {
  constructor(details: unknown) { super(400, "validation failed", details); }
}
```

```ts
// Throw in handlers — Express will route to the error handler
app.get("/users/:id", (req, res) => {
  const user = users.find((u) => u.id === req.params.id);
  if (!user) throw new NotFoundError("user");
  res.json(user);
});

// The error handler — last
app.use((err: unknown, req: Request, res: Response, _next: NextFunction) => {
  if (err instanceof HttpError) {
    return res.status(err.status).json({
      error: { code: err.name, message: err.message, details: err.details },
    });
  }
  console.error("unhandled", err);
  res.status(500).json({ error: { code: "INTERNAL", message: "internal error" } });
});
```

To make `throw` work with async handlers, wrap them:

```ts
// src/lib/asyncHandler.ts
import type { Request, Response, NextFunction, RequestHandler } from "express";

export const asyncHandler =
  <P = unknown, ResBody = unknown, ReqBody = unknown, ReqQuery = unknown>(
    fn: (req: Request<P, ResBody, ReqBody, ReqQuery>, res: Response<ResBody>, next: NextFunction) => Promise<unknown>,
  ): RequestHandler<P, ResBody, ReqBody, ReqQuery> =>
  (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);

// Usage
app.get("/users/:id", asyncHandler(async (req, res) => {
  const user = await db.getUser(req.params.id);
  if (!user) throw new NotFoundError("user");
  res.json(user);
}));
```

Express 5 (now stable) supports async handlers natively, so this wrapper is a legacy convenience. Use it for clarity either way.

---

## 7. Consistent response shapes

Adopt these conventions from day one:

```ts
// Success
res.status(200).json({ data: user });
res.status(201).json({ data: created });
res.status(204).end();

// Error
res.status(400).json({
  error: {
    code: "VALIDATION_ERROR",
    message: "Invalid input",
    details: zodErrors,
  },
});
```

This makes the frontend's life dramatically easier and is a strong senior-engineer signal.

---

## 8. API design principles

### Resource-oriented URLs

```
GET    /users              list
GET    /users/:id          read
POST   /users              create
PUT    /users/:id          replace
PATCH  /users/:id          partial update
DELETE /users/:id          delete

# Nested resources
GET    /users/:id/orders
POST   /users/:id/orders

# Actions (when no resource fits)
POST   /users/:id/password  action: change password
```

### Idempotency

- `GET`, `PUT`, `DELETE` should be **idempotent** — calling them N times has the same effect as once.
- `POST` is **not** idempotent — each call creates a new resource.
- For retried `POST`s (e.g. payment), pass an `Idempotency-Key` header.

### Status codes you must know

| Code | Meaning |
|------|---------|
| 200 | OK |
| 201 | Created |
| 204 | No Content |
| 301/302 | Redirect (permanent/temporary) |
| 400 | Bad Request (client error, validation) |
| 401 | Unauthorized (no/invalid auth) |
| 403 | Forbidden (auth ok, but not allowed) |
| 404 | Not Found |
| 409 | Conflict (duplicate, version mismatch) |
| 422 | Unprocessable Entity (semantic error) |
| 429 | Too Many Requests (rate-limited) |
| 500 | Internal Server Error |
| 503 | Service Unavailable |

---

## 9. Building an in-memory CRUD API

```ts
// src/data/users.ts
import { randomUUID } from "node:crypto";
import type { User } from "../types.js";

const users = new Map<string, User>();

export const usersRepo = {
  list(): User[] { return [...users.values()]; },
  findById(id: string): User | undefined { return users.get(id); },
  create(input: Omit<User, "id" | "createdAt">): User {
    const user: User = { id: randomUUID(), createdAt: new Date(), ...input };
    users.set(user.id, user);
    return user;
  },
  update(id: string, patch: Partial<User>): User | undefined {
    const u = users.get(id);
    if (!u) return undefined;
    const updated = { ...u, ...patch, id: u.id, createdAt: u.createdAt };
    users.set(id, updated);
    return updated;
  },
  delete(id: string): boolean { return users.delete(id); },
};
```

```ts
// src/routes/users.ts
import { Router } from "express";
import { z } from "zod";
import { asyncHandler } from "../lib/asyncHandler.js";
import { usersRepo } from "../data/users.js";
import { NotFoundError, ValidationError } from "../lib/errors.js";

const CreateUser = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
});

const UpdateUser = CreateUser.partial();

export const usersRouter = Router();

usersRouter.get("/", asyncHandler(async (_req, res) => {
  res.json({ data: usersRepo.list() });
}));

usersRouter.get("/:id", asyncHandler(async (req, res) => {
  const u = usersRepo.findById(req.params.id);
  if (!u) throw new NotFoundError("user");
  res.json({ data: u });
}));

usersRouter.post("/", asyncHandler(async (req, res) => {
  const parsed = CreateUser.safeParse(req.body);
  if (!parsed.success) throw new ValidationError(parsed.error.flatten());
  const u = usersRepo.create(parsed.data);
  res.status(201).json({ data: u });
}));

usersRouter.patch("/:id", asyncHandler(async (req, res) => {
  const parsed = UpdateUser.safeParse(req.body);
  if (!parsed.success) throw new ValidationError(parsed.error.flatten());
  const u = usersRepo.update(req.params.id, parsed.data);
  if (!u) throw new NotFoundError("user");
  res.json({ data: u });
}));

usersRouter.delete("/:id", asyncHandler(async (req, res) => {
  if (!usersRepo.delete(req.params.id)) throw new NotFoundError("user");
  res.status(204).end();
}));
```

```ts
// src/index.ts
import express from "express";
import cors from "cors";
import helmet from "helmet";
import morgan from "morgan";
import { usersRouter } from "./routes/users.js";
import { errorHandler } from "./lib/errorHandler.js";
import { config } from "./config.js";

const app = express();

app.use(helmet());
app.use(cors({ origin: config.corsOrigin, credentials: true }));
app.use(express.json({ limit: "1mb" }));
app.use(morgan("dev"));

app.get("/health", (_req, res) => res.json({ status: "ok" }));
app.use("/api/v1/users", usersRouter);

app.use(errorHandler);

app.listen(config.port, () => {
  console.log(`API listening on :${config.port}`);
});
```

---

## 10. Testing the API with Bruno

[Bruno](https://www.usebruno.com/) is a file-driven API client that lives in your repo. We use it instead of Postman in 2026 because:

- The collection is plain files you can commit and code-review.
- No cloud account needed.
- Plays nicely with Git.

Create `bruno/users/`:

```
bruno/
├── bruno.json
└── users/
    ├── list.bru
    ├── create.bru
    ├── get.bru
    └── delete.bru
```

`create.bru`:
```
meta { name: create }
post { url: {{baseUrl}}/api/v1/users }
body { content-type: application/json }
{
  "name": "Rumi",
  "email": "rumi@example.com"
}
```

`{{baseUrl}}` resolves to `http://localhost:3000` from `bruno/environments/local.bru`.

---

## Exercise

Build the user CRUD API above. Add:

1. A `GET /api/v1/users/search?q=...` that does case-insensitive substring match on name or email.
2. Pagination via `?page=1&limit=20` on the list endpoint.
3. A `404` handler for unknown routes.
4. A health endpoint at `/health` that also pings the in-memory store.
5. A request-logging middleware that prints method, path, status, and duration.
6. Validation on every body and query string using Zod.

Test with Bruno. Run with `pnpm dev`. Commit a clean PR.

---

## Common mistakes

1. **Forgetting to parse the body** (`express.json()` middleware missing).
2. **Throwing strings** (`throw "not found"` — never do this; throw an `Error` subclass).
3. **Returning 200 on errors.** Status codes are part of the API contract.
4. **Mixing `req.body` validation across endpoints.** Centralize Zod schemas.
5. **Trusting `req.body` shape.** Always validate with Zod.

---

## AI Industry Note (2026)

- **The "Express is dead" hot take is wrong.** Express 5 is stable, and the migration story from 4 is straightforward. Fastify is faster on benchmarks; Hono is great for edge runtimes. **For a course teaching the fundamentals, Express wins on readability and ecosystem.** Once you know Express, Fastify is a 2-day migration.
- **REST is still the lingua franca of B2B APIs.** GraphQL has its place; tRPC is amazing for monorepo TypeScript apps. But when a mobile app, a partner integration, and a web client all need to talk to the same backend, REST + OpenAPI is unbeatable.
- **The pattern that makes AI agents useful in your codebase:** a tightly-typed `asyncHandler` + a centralized `errorHandler` + Zod-validated inputs. When the AI writes a new endpoint, it copies *this* shape. The shape is your convention; the AI amplifies it.
