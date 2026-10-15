# Module 23 — Authorization: Middleware, Protected Routes & RBAC

> "Anyone can let a user in. The senior engineer's job is to let them in *only where they belong*." This module is about the difference between saying "you are logged in" and "you can do this."

---

## Why this matters

Authentication answered "who is this person". Now we need to answer a different question on almost every request: "is this person allowed to touch *this specific resource*?"

A SaaS that gets this wrong is a SaaS that gets breached. A SaaS that gets this right is a SaaS that can confidently onboard enterprise customers, charge per seat, and pass a SOC 2 audit.

This module builds the layer that protects every feature router from Module 21 forward:

- An auth middleware that verifies a token and attaches `req.user`
- A `requireRole(...roles)` middleware for simple role checks
- A `requireWorkspaceRole(role)` middleware for multi-tenant checks
- Refresh-token rotation with reuse detection
- Password change / forgot / reset flows
- A complete testing playbook in Postman / Bruno

---

## Learning objectives

By the end of this module you can:

1. Build an authentication middleware that extracts a Bearer token and attaches a typed `req.user`
2. Apply middleware per-router so that public endpoints stay public
3. Implement RBAC with `requireRole(...roles)`
4. Implement multi-tenant authorization with `requireWorkspaceRole(role)`
5. Rotate refresh tokens and **detect** token reuse as a credential-theft signal
6. Manage user status (`active` | `blocked` | `deleted`)
7. Implement change/forgot/reset password flows with secure single-use tokens
8. Walk the full auth lifecycle in Postman / Bruno
9. Add basic security headers and rate limiting
10. Audit a request like "is user X allowed to mutate resource Y in workspace Z?"

---

## 1. Type augmentation

```ts
// src/types/express.d.ts
import "express";

declare global {
  namespace Express {
    interface UserContext {
      id: string;
      email: string;
      status: "active" | "blocked" | "deleted";
    }

    interface Request {
      user?: UserContext;
    }
  }
}
export {};
```

Now `req.user` is typed everywhere.

---

## 2. The auth middleware

```ts
// src/modules/auth/auth.middleware.ts
import type { RequestHandler } from "express";
import jwt from "jsonwebtoken";
import { eq } from "drizzle-orm";
import { env } from "../../config/env.js";
import { db } from "../../db/client.js";
import { users } from "../../db/schema.js";
import { UnauthorizedError, ForbiddenError } from "../../lib/http-error.js";

export const requireAuth: RequestHandler = async (req, _res, next) => {
  try {
    const header = req.headers.authorization;
    if (!header?.startsWith("Bearer ")) {
      throw new UnauthorizedError("Missing or malformed Authorization header");
    }
    const token = header.slice("Bearer ".length);

    let payload: { sub?: string };
    try {
      payload = jwt.verify(token, env.JWT_ACCESS_SECRET, {
        algorithms: ["HS256"],
      }) as { sub?: string };
    } catch {
      throw new UnauthorizedError("Invalid or expired access token");
    }

    if (!payload.sub) throw new UnauthorizedError("Invalid token payload");

    const [user] = await db
      .select({ id: users.id, email: users.email, status: users.status })
      .from(users)
      .where(eq(users.id, payload.sub))
      .limit(1);

    if (!user) throw new UnauthorizedError("User no longer exists");
    if (user.status !== "active") {
      throw new ForbiddenError(`Account is ${user.status}`);
    }

    req.user = { id: user.id, email: user.email, status: user.status };
    next();
  } catch (err) {
    next(err);
  }
};
```

This middleware does three things:

1. **Verifies** the JWT cryptographically
2. **Loads** the current user from the database (so a blocked user is blocked *now*, not 15 minutes from now when the token expires)
3. **Attaches** a typed `req.user` for downstream handlers

The "look up the user every request" cost is the price of revocability. Pay it.

---

## 3. Protected routes — applied per-router

You attach `requireAuth` to a router, not globally. Public routes stay public.

```ts
// src/modules/tasks/tasks.router.ts
import { Router } from "express";
import { requireAuth } from "../auth/auth.middleware.js";
import { validate } from "../../lib/validate.js";
import { catchAsync } from "../../lib/catch-async.js";
import { ok } from "../../lib/response.js";
import { tasksService } from "./tasks.service.js";
import { createTaskSchema, listTasksSchema } from "./tasks.schemas.js";

// Public routes (none for now)
export const tasksRouter: Router = Router();

// Everything below requires a valid user
tasksRouter.use(requireAuth);

tasksRouter.post(
  "/",
  validate(createTaskSchema),
  catchAsync(async (req, res) => {
    const task = await tasksService.create(req.user!.id, req.body);
    return ok(res, task);
  }),
);

tasksRouter.get(
  "/",
  validate(listTasksSchema, "query"),
  catchAsync(async (req, res) => {
    const tasks = await tasksService.listForUser(req.user!.id, req.query);
    return ok(res, tasks);
  }),
);
```

The pattern is: mount public routes first, **then** `router.use(requireAuth)`. Anyone who tries to add a route below the middleware without realizing it's protected gets caught by code review.

---

## 4. RBAC with `requireRole`

For product-wide roles — `owner`, `admin`, `member` — we attach a role to the JWT (or, more commonly, look it up per request). For simplicity here we assume `users.role`:

```ts
// src/db/schema.ts (additions)
export const users = pgTable("users", {
  // ...existing fields...
  role: text("role").notNull().default("member"), // owner | admin | member
});
```

```ts
// src/lib/require-role.ts
import type { RequestHandler } from "express";
import { ForbiddenError } from "./http-error.js";

export const requireRole =
  (...allowed: string[]): RequestHandler =>
  (req, _res, next) => {
    if (!req.user) return next(new ForbiddenError("Not authenticated"));
    // role is not on the request yet — load it. In a hot path, cache it on the JWT.
    // For this example we assume a sync lookup happened in requireAuth.
    const role = (req as any).userRole as string | undefined;
    if (!role || !allowed.includes(role)) {
      return next(new ForbiddenError("Insufficient role"));
    }
    next();
  };
```

Use it:

```ts
import { requireRole } from "../../lib/require-role.js";

adminRouter.use(requireAuth, requireRole("admin", "owner"));
```

---

## 5. Multi-tenant RBAC — `requireWorkspaceRole`

For a SaaS, product-wide roles aren't enough. A user might be an `admin` of workspace A and a `member` of workspace B. Authorization is **per workspace, per resource**.

```ts
// src/db/schema.ts (additions)
export const workspaces = pgTable("workspaces", {
  id: uuid("id").defaultRandom().primaryKey(),
  name: text("name").notNull(),
  createdAt: timestamp("created_at").notNull().defaultNow(),
});

export const workspaceMembers = pgTable(
  "workspace_members",
  {
    workspaceId: uuid("workspace_id").notNull().references(() => workspaces.id, { onDelete: "cascade" }),
    userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
    role: text("role").notNull(), // owner | admin | member | guest
    createdAt: timestamp("created_at").notNull().defaultNow(),
  },
  (t) => ({
    pk: { columns: [t.workspaceId, t.userId] },
  }),
);
```

```ts
// src/modules/workspaces/workspace-auth.ts
import type { RequestHandler } from "express";
import { and, eq } from "drizzle-orm";
import { db } from "../../db/client.js";
import { workspaceMembers } from "../../db/schema.js";
import { ForbiddenError, NotFoundError, ValidationError } from "../../lib/http-error.js";

export interface WorkspaceContext {
  workspaceId: string;
  role: string;
}

declare global {
  namespace Express {
    interface Request {
      workspace?: WorkspaceContext;
    }
  }
}

/**
 * Reads workspace id from :workspaceId in the URL, loads the user's
 * membership, attaches it to req.workspace. If `allowedRoles` is given,
 * enforces membership-level role.
 */
export const requireWorkspaceRole =
  (...allowedRoles: string[]): RequestHandler =>
  async (req, _res, next) => {
    try {
      if (!req.user) throw new ForbiddenError("Not authenticated");

      const workspaceId = req.params.workspaceId;
      if (!workspaceId || !/^[0-9a-f-]{36}$/i.test(workspaceId)) {
        throw new ValidationError("Invalid workspaceId in URL");
      }

      const [membership] = await db
        .select()
        .from(workspaceMembers)
        .where(
          and(
            eq(workspaceMembers.workspaceId, workspaceId),
            eq(workspaceMembers.userId, req.user.id),
          ),
        )
        .limit(1);

      if (!membership) {
        // Use 404 to avoid leaking the existence of the workspace
        throw new NotFoundError("Workspace not found");
      }

      if (allowedRoles.length > 0 && !allowedRoles.includes(membership.role)) {
        throw new ForbiddenError("Insufficient workspace role");
      }

      req.workspace = { workspaceId, role: membership.role };
      next();
    } catch (err) {
      next(err);
    }
  };
```

Wire it into a router:

```ts
import { requireWorkspaceRole } from "../workspaces/workspace-auth.js";

projectsRouter.use(requireAuth);

projectsRouter.post(
  "/:workspaceId/projects",
  requireWorkspaceRole("admin", "owner"),
  catchAsync(async (req, res) => {
    const project = await projectsService.create(req.workspace!.workspaceId, req.body);
    return ok(res, project);
  }),
);

projectsRouter.get(
  "/:workspaceId/projects",
  requireWorkspaceRole("member", "admin", "owner", "guest"),
  catchAsync(async (req, res) => {
    const list = await projectsService.list(req.workspace!.workspaceId);
    return ok(res, list);
  }),
);
```

**The discipline:** every mutating route specifies *exactly* which workspace roles may call it. Every read route specifies *at least* `guest` (or whatever the read role is). When in doubt, deny.

---

## 6. Refresh token rotation + reuse detection

Module 22 issued a new refresh token on every refresh and revoked the old one. Rotation by itself is good. **Reuse detection** turns it into a security feature.

```ts
// src/modules/auth/auth.service.ts (refined)
async refresh(refreshToken: string, meta: { ip?: string; userAgent?: string }) {
  const hash = hashRefreshToken(refreshToken);
  const [row] = await db
    .select()
    .from(refreshTokens)
    .where(eq(refreshTokens.tokenHash, hash))
    .limit(1);

  if (!row) throw new UnauthorizedError("Refresh token invalid");

  // Reuse detection: if it's already revoked, someone is replaying it.
  // That means the token leaked. Revoke ALL tokens for this user.
  if (row.revokedAt) {
    await db
      .update(refreshTokens)
      .set({ revokedAt: new Date() })
      .where(and(eq(refreshTokens.userId, row.userId), isNull(refreshTokens.revokedAt)));
    logger.warn({ userId: row.userId, ip: meta.ip }, "refresh token reuse detected");
    throw new UnauthorizedError("Refresh token reuse detected");
  }

  if (row.expiresAt < new Date()) {
    throw new UnauthorizedError("Refresh token expired");
  }

  // Normal rotation: revoke old, issue new
  await db
    .update(refreshTokens)
    .set({ revokedAt: new Date() })
    .where(eq(refreshTokens.id, row.id));

  const [user] = await db
    .select({ id: users.id, email: users.email })
    .from(users)
    .where(eq(users.id, row.userId))
    .limit(1);
  if (!user || user.status !== "active") throw new UnauthorizedError("User no longer active");

  return { user, ...(await this.issueTokenPair(user.id, user.email, meta)) };
},
```

The principle: if the *same* refresh token is used twice, it is treated as a stolen token. The legitimate user has already rotated to a new one. The thief is trying to use the old one. The right response is **lock everything down** and force re-login.

---

## 7. User status management

```ts
// src/modules/users/users.service.ts
export const usersService = {
  async block(userId: string) {
    await db.update(users).set({ status: "blocked", updatedAt: new Date() }).where(eq(users.id, userId));
    await db.update(refreshTokens).set({ revokedAt: new Date() })
      .where(and(eq(refreshTokens.userId, userId), isNull(refreshTokens.revokedAt)));
  },

  async softDelete(userId: string) {
    await db.update(users).set({ status: "deleted", updatedAt: new Date() }).where(eq(users.id, userId));
    await db.update(refreshTokens).set({ revokedAt: new Date() })
      .where(and(eq(refreshTokens.userId, userId), isNull(refreshTokens.revokedAt)));
  },

  async reactivate(userId: string) {
    await db.update(users).set({ status: "active", updatedAt: new Date() }).where(eq(users.id, userId));
  },
};
```

`requireAuth` (Section 2) refuses anything but `active`, so the moment a user is blocked, every subsequent request fails. Existing refresh tokens are revoked so they cannot mint new access tokens.

---

## 8. Password flows

### Change password (logged-in user)

```ts
// src/modules/auth/auth.service.ts (additions)
async changePassword(userId: string, current: string, next: string) {
  const [user] = await db.select().from(users).where(eq(users.id, userId)).limit(1);
  if (!user || user.status !== "active") throw new UnauthorizedError();

  const ok = await verifyPassword(current, user.passwordHash);
  if (!ok) throw new UnauthorizedError("Current password incorrect");

  if (current === next) throw new ValidationError("New password must differ");

  const newHash = await hashPassword(next);
  await db.update(users).set({ passwordHash: newHash, updatedAt: new Date() }).where(eq(users.id, userId));

  // Rotate out all existing sessions
  await db.update(refreshTokens).set({ revokedAt: new Date() })
    .where(and(eq(refreshTokens.userId, userId), isNull(refreshTokens.revokedAt)));
},
```

Rotating sessions on a password change is non-negotiable. Otherwise an attacker who knows the old password keeps their session.

### Forgot / reset password

```ts
// src/db/schema.ts (additions)
export const passwordResets = pgTable("password_resets", {
  id: uuid("id").defaultRandom().primaryKey(),
  userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  tokenHash: text("token_hash").notNull().unique(),
  expiresAt: timestamp("expires_at", { withTimezone: true }).notNull(),
  usedAt: timestamp("used_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});
```

```ts
// src/modules/auth/password-reset.service.ts
import { eq, and, isNull, gt } from "drizzle-orm";
import crypto from "node:crypto";
import { db } from "../../db/client.js";
import { users, passwordResets, refreshTokens } from "../../db/schema.js";
import { hashPassword } from "./password.js";

const TTL_MIN = 30;

export const passwordResetService = {
  async request(email: string): Promise<{ token?: string } | void> {
    // Always respond in the same shape; never reveal whether the email exists.
    const [user] = await db.select().from(users).where(eq(users.email, email)).limit(1);
    if (!user) return;

    const raw = crypto.randomBytes(32).toString("base64url");
    const tokenHash = crypto.createHash("sha256").update(raw).digest("hex");
    const expiresAt = new Date(Date.now() + TTL_MIN * 60 * 1000);

    await db.insert(passwordResets).values({ userId: user.id, tokenHash, expiresAt });
    // In real life: send `raw` via email. Returned here only for testing.
    return { token: raw };
  },

  async reset(token: string, newPassword: string) {
    const tokenHash = crypto.createHash("sha256").update(token).digest("hex");
    const [row] = await db
      .select()
      .from(passwordResets)
      .where(and(
        eq(passwordResets.tokenHash, tokenHash),
        isNull(passwordResets.usedAt),
        gt(passwordResets.expiresAt, new Date()),
      ))
      .limit(1);

    if (!row) throw new UnauthorizedError("Invalid or expired reset token");

    const newHash = await hashPassword(newPassword);
    await db.transaction(async (tx) => {
      await tx.update(users).set({ passwordHash: newHash, updatedAt: new Date() })
        .where(eq(users.id, row.userId));
      await tx.update(passwordResets).set({ usedAt: new Date() })
        .where(eq(passwordResets.id, row.id));
      await tx.update(refreshTokens).set({ revokedAt: new Date() })
        .where(and(eq(refreshTokens.userId, row.userId), isNull(refreshTokens.revokedAt)));
    });
  },
};
```

Three properties:
- Single-use (`usedAt`)
- Short-lived (30 minutes)
- Invalidates all sessions on success

---

## 9. Rate limiting (preview)

```ts
// src/app.ts
import rateLimit from "express-rate-limit";

const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 20,                  // 20 attempts per window per IP
  standardHeaders: true,
  legacyHeaders: false,
});

app.use("/auth/login", authLimiter);
app.use("/auth/register", authLimiter);
app.use("/auth/forgot", authLimiter);
```

Always rate-limit auth endpoints. Even a 10× slowdown on credential stuffing is a win. (Module 32 covers the production-grade setup.)

---

## 10. Testing the full flow in Postman / Bruno

The minimum sequence to prove the system works end-to-end:

| # | Step                                 | Expected                              |
| - | ------------------------------------ | ------------------------------------- |
| 1 | `POST /auth/register`                | `200`, `data.accessToken` returned, `rt` cookie set |
| 2 | `GET /auth/me` with `Bearer` token   | `200`, `data.email` matches           |
| 3 | `POST /auth/refresh` (cookie only)   | `200`, new access token, rotated cookie |
| 4 | Replay the **old** cookie            | `401`, *and* all sessions revoked     |
| 5 | `POST /auth/login`                   | `200`                                 |
| 6 | `POST /auth/change-password`         | `200`, cookie rotated again           |
| 7 | `GET /auth/me` with old access token | `401`                                 |
| 8 | `POST /auth/logout`                  | `200`, subsequent refresh fails       |
| 9 | `POST /auth/forgot` for same email   | `200` with `meta: { delivered: true }` (or whatever your envelope says) |
| 10 | `POST /auth/reset` with the token   | `200`, can log in with new password    |

Save this as a Bruno collection so you can replay it on every change.

---

## Common mistakes

1. **Authorization in the handler, not in middleware.** Every handler forgets once. Push the check up.
2. **Checking role from the JWT alone.** JWTs are 15 minutes stale. Look the user up. (See Section 2.)
3. **Returning `403` for "workspace does not exist".** That leaks the existence of resources. Return `404`.
4. **One global `admin` role for everything.** A user with product-admin powers should not automatically have workspace powers in every workspace.
5. **Forgetting to scope queries to `req.workspace.workspaceId`.** A `where workspaceId = ?` clause that hardcodes "all" turns your multi-tenant SaaS into a single-tenant leak waiting to happen.
6. **No reuse detection on refresh tokens.** Token rotation without reuse detection catches *some* attacks; with it, you catch the rest.
7. **Not rotating sessions on password change / reset.** The user's account is "secure" but the attacker's session keeps going.
8. **Skipping rate limits on `/auth/login`.** Brute force is free for the attacker; make it expensive.

---

## Exercise

**Goal:** Build a multi-tenant project API with full RBAC and prove it in tests.

1. Add the `workspaces` and `workspace_members` tables and run it.
3. Implement `requireWorkspaceRole(...roles)` exactly as in Section 5.
4. Implement `requireRole(...roles)` for product-wide admin endpoints.
5. Add `usersService.block` and verify that calling `requireAuth` with a blocked user returns `403`.
6. Implement `refresh` with reuse detection. Add a Vitest test that:
   - Calls `register`
   - Calls `refresh` once — expect a new cookie
   - Calls `refresh` again with the **first** cookie — expect `401` and the second cookie to also stop working
7. Implement `changePassword` and verify the rotation test from above.
8. Add `express-rate-limit` to `/auth/login` and verify the 21st request inside 15 min returns `429`.
9. In Postman / Bruno, save the entire auth flow as a collection and commit it to the repo.

**Bonus:** add an audit-log table and a `withAudit("project.create", handler)` higher-order middleware that records `(userId, workspaceId, action, ip, ts)`. Senior engineers start every feature with the audit log.

---

## AI Industry Note (2026)

In 2026, AI assistants are very good at writing the *happy path* of an RBAC system. They will scaffold `requireAuth`, `requireRole`, `requireWorkspaceRole`, and even the resource-ownership checks. They are *bad* at the negative space:

- The audit log that gets added in month 4 when the first SOC 2 auditor shows up
- The "this workspace doesn't exist for you" `404` that prevents enumeration
- The session-rotation-on-password-change that gets added after the first breach
- The reuse-detection on refresh tokens that catches the credential stuffing that nobody noticed in week 2
- The rate limit on `/auth/login` that turns a free brute force into a billable attack

The senior engineer's value is not "I know how `jwt.verify` works." It is:

- "I can read a request and immediately name the four authorization decisions that need to be made and the data each one needs."
- "I know which of those decisions need a database round-trip and which can be cached for 60 seconds."
- "I know what to *not* return (404 vs 403, generic vs specific errors)."
- "I know what to *add later* — the audit log, the per-IP anomaly detection, the geo-fencing — and I leave seams for them."

In 2026 the AI writes the code. You write the threat model.

When you review AI-generated auth code, ask:

1. Does it actually look the user up on every request, or does it trust the JWT?
2. Does it scope every query by `workspaceId`, or does it leak across tenants?
3. Does it rotate sessions on password change?
4. Does it detect refresh token reuse?
5. Does it rate-limit the auth endpoints?
6. Does its error response leak whether an email is registered?

If the answer to any of those is "no", the AI's code is *not* production-ready. Your job is to make it so.