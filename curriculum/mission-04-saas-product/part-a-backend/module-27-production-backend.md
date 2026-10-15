# Module 27: Production Backend Engineering

> "If it isn't observable, it isn't deployable. If it isn't resilient, it isn't production."

---

## 1. Why this matters

The last three modules gave you a working backend. This module is the difference between "works on my laptop" and "works for paying customers at 3 AM."

Production engineering is a long list of small disciplines:

- Structured logging so you can find the one log line that matters in a million.
- A request id propagated to every log and outbound call.
- An error hierarchy that maps cleanly to HTTP responses without leaking stack traces.
- Rate limiting so a single user can't melt your database.
- Security middleware so the basics (headers, CORS, body size) are handled once.
- Transactions for multi-step writes that must succeed or fail together.
- Health and readiness probes so your orchestrator knows when to send traffic.
- Graceful shutdown so in-flight requests finish before the process dies.
- API docs that are generated, not maintained, so they never go stale.

This module is the kit. Every line of code is something you'll reach for in *every* production backend you ever build.

---

## 2. Learning objectives

By the end of this module you can:

1. Wrap multi-step operations in `db.transaction(...)` with the right isolation level.
2. Compute aggregations and percentiles for analytics endpoints.
3. Emit structured logs with pino, propagate a request id, and configure log levels per environment.
4. Build an `HttpError` hierarchy that maps cleanly to JSON responses.
5. Rate-limit per-IP and per-user with `express-rate-limit` and a Redis store.
6. Apply helmet, CORS, body size limits, and basic input sanitization.
7. Implement `/health` and `/ready` endpoints and a graceful shutdown handler.
8. Auto-generate OpenAPI from Zod and ship Swagger UI.
9. Recap the backend and look ahead to the frontend.

---

## 3. Conceptual model

### 3.1 The application entry point

This is the wiring diagram for everything in this module.

```ts
// src/index.ts
import { createApp } from './app';
import { logger } from './lib/logger';
import { startServer, shutdown } from './lib/server';

const app = createApp();
const server = startServer(app, Number(process.env.PORT ?? 4000));

// Last-resort safety net
process.on('unhandledRejection', (err) => logger.error({ err }, 'unhandledRejection'));
process.on('uncaughtException',  (err) => logger.fatal({ err }, 'uncaughtException'));

for (const sig of ['SIGTERM', 'SIGINT'] as const) {
  process.on(sig, () => { logger.info({ sig }, 'shutting down'); shutdown(server); });
}
```

### 3.2 Structured logging with pino

```ts
// src/lib/logger.ts
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL ?? (process.env.NODE_ENV === 'production' ? 'info' : 'debug'),
  base: { service: 'saas-api', env: process.env.NODE_ENV, version: process.env.GIT_SHA },
  redact: {
    paths: ['req.headers.authorization', 'req.headers.cookie', '*.password', '*.token', '*.secret'],
    censor: '[redacted]',
  },
  timestamp: pino.stdTimeFunctions.isoTime,
});
```

A request-scoped logger:

```ts
// src/middleware/requestLogger.ts
import { randomUUID } from 'node:crypto';
import type { Request, Response, NextFunction } from 'express';
import { logger } from '../lib/logger';

declare module 'express-serve-static-core' {
  interface Request { id: string; log: typeof logger }
}

export function requestLogger(req: Request, res: Response, next: NextFunction) {
  req.id = req.header('x-request-id') ?? randomUUID();
  res.setHeader('x-request-id', req.id);
  req.log = logger.child({ reqId: req.id, method: req.method, path: req.path });

  const start = process.hrtime.bigint();
  res.on('finish', () => {
    const durationMs = Number(process.hrtime.bigint() - start) / 1e6;
    req.log.info({
      status: res.statusCode,
      durationMs: Math.round(durationMs),
      userId: req.user?.id,
      workspaceId: req.workspace?.id,
    }, 'request');
  });
  next();
}
```

Pass `req.log` into services instead of the global `logger` so every log line carries the request id.

### 3.3 The error hierarchy

```ts
// src/lib/errors.ts
export class HttpError extends Error {
  constructor(public status: number, public override message: string, public details?: unknown) {
    super(message);
    this.name = 'HttpError';
  }
}
export class BadRequest   extends HttpError { constructor(m='bad request', d?: any) { super(400, m, d); } }
export class Unauthorized extends HttpError { constructor(m='unauthorized')          { super(401, m); } }
export class Forbidden    extends HttpError { constructor(m='forbidden')             { super(403, m); } }
export class NotFound     extends HttpError { constructor(m='not found')             { super(404, m); } }
export class Conflict     extends HttpError { constructor(m='conflict', d?: any)    { super(409, m, d); } }
export class TooLarge     extends HttpError { constructor(m='payload too large')    { super(413, m); } }
export class TooMany      extends HttpError { constructor(m='rate limited')          { super(429, m); } }
```

The central error middleware:

```ts
// src/middleware/errorHandler.ts
import type { ErrorRequestHandler } from 'express';
import { ZodError } from 'zod';
import { HttpError } from '../lib/errors';
import { logger } from '../lib/logger';

export const errorHandler: ErrorRequestHandler = (err, req, res, _next) => {
  // 1) Zod validation
  if (err instanceof ZodError) {
    return res.status(422).json({ error: 'validation_failed', issues: err.flatten() });
  }

  // 2) Known HTTP errors
  if (err instanceof HttpError) {
    if (err.status >= 500) req.log?.error({ err }, 'http error 5xx');
    return res.status(err.status).json({ error: err.message, details: err.details });
  }

  // 3) Postgres unique violation, etc — map known codes
  if ((err as any)?.code === '23505') {
    return res.status(409).json({ error: 'duplicate', details: (err as any).detail });
  }

  // 4) Unknown — NEVER leak the stack
  req.log?.error({ err }, 'unhandled error');
  res.status(500).json({ error: 'internal_error' });
};
```

Production rule: **stack traces go to logs, never to the response**. The user gets `{ "error": "internal_error" }`; you get the full stack with the request id.

### 3.4 Transactions for multi-step operations

Drizzle's `db.transaction` opens a transaction, hands you a `tx`, and commits if the callback resolves, rolls back if it throws or rejects.

```ts
// src/services/projectService.ts
export async function archiveProjectWithTasks(workspaceId: string, projectId: string, userId: string) {
  return db.transaction(async (tx) => {
    const [project] = await tx.update(projects)
      .set({ deletedAt: new Date(), updatedBy: userId })
      .where(and(eq(projects.id, projectId), eq(projects.workspaceId, workspaceId), isNull(projects.deletedAt)))
      .returning({ id: projects.id });
    if (!project) throw new NotFound('project not found');

    await tx.update(tasks)
      .set({ status: 'archived', updatedAt: new Date(), updatedBy: userId })
      .where(and(eq(tasks.projectId, projectId), eq(tasks.workspaceId, workspaceId)));

    await tx.insert(auditLog).values({
      workspaceId, actorId: userId, action: 'project.archive', targetId: projectId,
    });
  }, { isolationLevel: 'read committed' });
}
```

The default isolation level (`read committed`) is correct here. Reach for `serializable` only when you have a write-skew hazard you can prove, and back it with a retry loop:

```ts
async function withSerializableRetry<T>(fn: () => Promise<T>, max = 3): Promise<T> {
  for (let attempt = 0; ; attempt++) {
    try {
      return await db.transaction(fn, { isolationLevel: 'serializable' });
    } catch (err: any) {
      if (err?.code === '40001' && attempt < max) continue;   // serialization_failure
      throw err;
    }
  }
}
```

### 3.5 Analytics: aggregations and percentiles

A weekly summary of task throughput per workspace:

```ts
// src/services/analyticsService.ts
import { sql, and, eq, gte } from 'drizzle-orm';
import { tasks } from '../db/schema/tasks';

export async function weeklyTaskStats(workspaceId: string, sinceDays = 30) {
  const since = new Date(Date.now() - sinceDays * 86400_000);

  const byDay = await db.execute(sql`
    SELECT
      date_trunc('day', ${tasks.createdAt}) AS day,
      COUNT(*)::int                                              AS created,
      COUNT(*) FILTER (WHERE ${tasks.status} = 'done')::int      AS done
    FROM ${tasks}
    WHERE ${tasks.workspaceId} = ${workspaceId}
      AND ${tasks.deletedAt} IS NULL
      AND ${tasks.createdAt} >= ${since}
    GROUP BY 1
    ORDER BY 1
  `);

  // Postgres has no native percentile_disc/percentile_cont, but it does:
  const percentiles = await db.execute(sql`
    SELECT
      percentile_cont(0.50) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (updated_at - created_at))) AS p50_seconds,
      percentile_cont(0.90) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (updated_at - created_at))) AS p90_seconds
    FROM ${tasks}
    WHERE ${tasks.workspaceId} = ${workspaceId}
      AND ${tasks.status} = 'done'
      AND ${tasks.deletedAt} IS NULL
      AND ${tasks.createdAt} >= ${since}
  `);

  return { byDay, percentiles };
}
```

For "count of tasks completed per project" you want `GROUP BY project_id` and a join to projects for the name. For "how many distinct users did anything", `COUNT(DISTINCT user_id)`.

Two things to watch: aggregations across millions of rows need materialized views or a dedicated analytics warehouse, not your OLTP database. And never run a `SELECT ... GROUP BY` over the whole `tasks` table for a request that has to return in 200ms — precompute, cache, or deny.

### 3.6 Rate limiting with `express-rate-limit` + Redis

```ts
// src/middleware/rateLimit.ts
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import { redis } from '../lib/redis';

export const perIp = rateLimit({
  windowMs: 60_000,
  limit: 100,                     // 100 req/min/IP for anonymous-ish routes
  standardHeaders: 'draft-7',
  legacyHeaders: false,
  store: new RedisStore({ sendCommand: (...args) => redis.call(args[0], args.slice(1)) as any }),
  message: { error: 'rate_limited' },
});

export const perUser = rateLimit({
  windowMs: 60_000,
  limit: 300,
  keyGenerator: (req) => req.user?.id ?? req.ip!,
  store: new RedisStore({ sendCommand: (...args) => redis.call(args[0], args.slice(1)) as any }),
});
```

For tighter limits on expensive routes, scope a limiter to that route:

```ts
router.post('/workspaces/:workspaceId/invitations', perUser, createInvitation);
```

### 3.7 Security middleware

```ts
// src/middleware/security.ts
import helmet from 'helmet';
import cors from 'cors';
import express from 'express';

export const securityMiddleware = [
  helmet({ contentSecurityPolicy: false }),   // API, no HTML
  cors({
    origin: process.env.WEB_ORIGIN?.split(',') ?? ['http://localhost:3000'],
    credentials: true,
    methods: ['GET', 'POST', 'PATCH', 'DELETE', 'OPTIONS'],
  }),
  express.json({ limit: '1mb' }),
  express.urlencoded({ extended: false, limit: '1mb' }),
];
```

A request-sanitization middleware to strip null bytes and normalize content-type:

```ts
export function sanitizeBody(req: Request, _res: Response, next: NextFunction) {
  if (req.body && typeof req.body === 'string') {
    req.body = req.body.replace(/\0/g, '');
  } else if (req.body && typeof req.body === 'object') {
    for (const k of Object.keys(req.body)) {
      if (typeof req.body[k] === 'string') req.body[k] = req.body[k].replace(/\0/g, '');
    }
  }
  next();
}
```

Helmet also handles `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, and a sane default set of headers that you almost never want to override.

### 3.8 Health checks

Two endpoints, two meanings:

- `/health` — process is alive. Returns 200 if the event loop is responsive.
- `/ready` — process can serve traffic. Returns 200 only when the database and Redis are reachable.

```ts
// src/routes/health.ts
import { Router } from 'express';
import { db } from '../db';
import { sql } from 'drizzle-orm';
import { redis } from '../lib/redis';

const router = Router();

router.get('/health', (_req, res) => res.json({ status: 'ok' }));

router.get('/ready', async (_req, res) => {
  const checks = { db: false, redis: false };
  try { await db.execute(sql`SELECT 1`); checks.db = true; } catch {}
  try { const r = await redis.ping(); checks.redis = r === 'PONG'; } catch {}
  const ready = Object.values(checks).every(Boolean);
  res.status(ready ? 200 : 503).json({ ready, checks });
});

export default router;
```

Your orchestrator (Kubernetes, ECS, Fly) will hit `/health` for liveness and `/ready` for readiness, restarting the first and adding/removing the second from the load balancer.

### 3.9 Graceful shutdown

```ts
// src/lib/server.ts
import type { Server } from 'node:http';
import type { Express } from 'express';

export function startServer(app: Express, port: number) {
  const server = app.listen(port, () => logger.info({ port }, 'listening'));
  server.keepAliveTimeout = 65_000;     // > ALB idle timeout
  server.headersTimeout = 66_000;
  return server;
}

export async function shutdown(server: Server) {
  // 1) Stop accepting new connections
  server.close((err) => {
    if (err) logger.error({ err }, 'server.close failed');
    process.exit(0);
  });

  // 2) Hard deadline — if shutdown takes too long, exit anyway
  setTimeout(() => {
    logger.warn('forcing exit after shutdown timeout');
    process.exit(1);
  }, 30_000).unref();
}
```

In flight requests get a chance to finish, the server stops accepting new ones, and the process exits. The `unref()` on the timer is critical — it doesn't keep the process alive on its own.

### 3.10 OpenAPI from Zod, served on Swagger UI

`@asteasolutions/zod-to-openapi` extends Zod with `.openapi(...)` so each schema carries its own metadata. You describe your endpoints once, and the OpenAPI doc is generated.

```ts
// src/schemas/listTasks.ts
import { extendZodWithOpenApi } from '@asteasolutions/zod-to-openapi';
import { z } from 'zod';
extendZodWithOpenApi(z);

export const Task = z.object({
  id: z.string().uuid().openapi({ example: 'b3...' }),
  title: z.string(),
  status: z.enum(['todo', 'in_progress', 'done', 'archived']),
  createdAt: z.string().datetime(),
}).openapi('Task');

export const ListTasksQuery = z.object({
  q: z.string().optional().openapi({ description: 'full-text search' }),
  status: z.enum(['todo', 'in_progress', 'done', 'archived']).optional(),
  sortBy: z.enum(['createdAt', 'updatedAt', 'dueAt', 'title', 'status']).default('createdAt'),
  sortDir: z.enum(['asc', 'desc']).default('desc'),
  cursor: z.string().optional(),
  limit: z.coerce.number().int().min(1).max(100).default(25),
}).openapi('ListTasksQuery');
```

```ts
// src/openapi.ts
import { OpenAPIRegistry, OpenApiGeneratorV3 } from '@asteasolutions/zod-to-openapi';
import { Task, ListTasksQuery } from './schemas/listTasks';

const registry = new OpenAPIRegistry();

registry.registerPath({
  method: 'get',
  path: '/workspaces/{workspaceId}/tasks',
  summary: 'List tasks in a workspace',
  request: { query: ListTasksQuery, params: z.object({ workspaceId: z.string().uuid() }) },
  responses: { 200: { description: 'OK', content: { 'application/json': { schema: z.object({ rows: z.array(Task), pageInfo: z.any() }) } } } },
});

export function buildOpenApiDocument() {
  return new OpenApiGeneratorV3(registry.definitions).generateDocument({
    openapi: '3.0.3',
    info: { title: 'SaaS API', version: '1.0.0' },
  });
}
```

```ts
// src/routes/docs.ts
import { Router } from 'express';
import swaggerUi from 'swagger-ui-express';
import { buildOpenApiDocument } from '../openapi';

const router = Router();
router.get('/openapi.json', (_req, res) => res.json(buildOpenApiDocument()));
router.use('/docs', swaggerUi.serve, swaggerUi.setup(buildOpenApiDocument()));
export default router;
```

The win: schemas are written *once*, validated at runtime by Zod, and turned into a public spec by the generator. They never go stale because there is no second source to forget.

### 3.11 The full app composition

```ts
// src/app.ts
import express from 'express';
import { securityMiddleware } from './middleware/security';
import { requestLogger } from './middleware/requestLogger';
import { errorHandler } from './middleware/errorHandler';
import { perIp, perUser } from './middleware/rateLimit';
import healthRouter from './routes/health';
import docsRouter from './routes/docs';

export function createApp() {
  const app = express();
  app.disable('x-powered-by');
  app.set('trust proxy', 1);                  // behind a load balancer

  app.use(requestLogger);
  app.use(securityMiddleware);
  app.use(perIp);

  app.use(healthRouter);
  app.use(docsRouter);

  app.use('/v1', perUser, v1Router);          // tenant-scoped routes

  app.use(errorHandler);
  return app;
}
```

---

## 4. Common mistakes

1. **Logging `req.user` in plaintext.** Always redact at the logger level (`pino redact`), and treat any new sensitive field as a bug to fix in `redact.paths`.

2. **Returning 500 with the stack trace in production.** It's the single most common way SaaS products leak internal information. The error handler is the only place that decides what the user sees; the answer is "almost nothing."

3. **A single rate limiter for the whole app.** `100 req/min` across the whole product is meaningless — login and search have wildly different cost profiles. Scope limiters to routes.

4. **`/ready` doing the same thing as `/health`.** If your readiness check is just "I can return JSON", the load balancer will keep sending traffic to a process whose database is down. The check must call the dependencies it needs.

5. **Transactions across HTTP boundaries.** "I'll open a transaction in the route and commit in a webhook" — no. Open and close the transaction within a single function call.

6. **OpenAPI generated by hand.** Hand-written OpenAPI drifts within a week. Generate it from the same Zod schemas you validate against. There is no other source of truth.

7. **Forgetting `app.set('trust proxy', 1)`** behind a load balancer. Rate limits and IP-based logs become wrong; "1" means "trust the first proxy", which is what you want behind a single ALB.

---

## 5. Exercise

1. Add `pino`, `requestLogger`, and `errorHandler` to your app. Hit any route and verify the log line includes `reqId`, `status`, and `durationMs`.
2. Wrap `archiveProjectWithTasks` in a transaction. Force a failure halfway through and verify no partial state is committed.
3. Add `helmet`, `cors`, and a JSON body size limit. Inspect the response headers.
4. Add `perIp` and `perUser` limiters. Test with a `for` loop in your terminal — you should see 429s.
5. Add `/health` and `/ready` and a graceful shutdown handler. Test by sending `SIGTERM` to the process.
6. Add `zod-to-openapi` and ship Swagger UI at `/docs`. Open it in a browser; it should document your routes.
7. **Stretch:** Add a `circuitBreaker()` around the Cloudinary upload (open after 5 consecutive failures, half-open after 30s, close after 2 successes).

**Acceptance criteria:**
- An unhandled error in a route returns 500 with `{"error":"internal_error"}` and a full stack in the logs.
- Rate limiter returns 429 with a `Retry-After` header.
- `SIGTERM` lets in-flight requests complete and exits within 30 seconds.
- `/ready` returns 503 if you stop the database.
- The OpenAPI doc includes your `ListTasks` query schema and your `Task` response schema.

---

## 6. Part A recap and what to expect in Part B

You now have a backend that:

- Authenticates users, manages workspaces and roles, scopes every query by tenant.
- Models domain data with soft delete, audit fields, and idempotent write endpoints.
- Stores files in cloud object storage and serves them via short-lived signed URLs.
- Searches, filters, sorts, and paginates with explicit, allow-listed contracts.
- Logs structured, request-scoped events; maps errors to clean HTTP responses; rate-limits and locks down headers; shuts down gracefully.

That's a real backend. It's the kind of code you'd be comfortable deploying behind a load balancer and handing the on-call rotation over to.

**In Part B, you'll build the Next.js frontend that consumes it.** Expect:

- A typed API client generated from the OpenAPI spec (so the frontend *cannot* call a route that doesn't exist).
- Server Components for fast initial renders with the workspace-scoped data.
- Mutations via Server Actions that include the `Idempotency-Key` header we built in Module 25.
- Auth handled by an httpOnly cookie that the Express backend sets, with `SameSite=Lax` and `Secure`.
- A design system in Tailwind that the product team can extend without touching the API.

## 7. AI Industry Note (2026)

The 2026 production stack looks familiar — same Postgres, same Express, same Redis — but the *practices* have hardened:

- **AI-assisted observability.** Log lines go to a vendor that uses an LLM to summarize incidents. "Why was the API slow yesterday between 2pm and 3pm?" is a question you ask, not a dashboard you build. For that to work, your logs must be structured, request-scoped, and redacted by default — exactly the patterns in this module.

- **AI-generated migrations are routine** but reviewed by humans, and every migration runs through a "expand-contract" pipeline: add the new column, backfill, dual-write, switch reads, drop the old column. AIs don't get to do an in-place destructive migration in a single PR. The schema patterns in Module 25 (audit fields, soft delete) make expand-contract trivial because the new columns are additive.

- **AI does not replace the production engineer.** It writes the *boilerplate* — the helmet config, the pino redact list, the graceful shutdown, the OpenAPI registry. You are the one who decides *which* trade-offs to make: serializable vs read committed, per-IP vs per-user rate limit, soft delete vs hard delete, the "last owner" rule, the idempotency window. The patterns in this module are the *language* you use to make those decisions explicit so the model can implement them without you having to specify every line.

You're now a backend engineer who can ship, not just a developer who can code. On to Part B.
