# Module 22 — Authentication: Registration, Login & JWT

> "Authentication is who you are. Authorization is what you can do. Mix them up, and you have built a system that confuses everyone — including the attacker." This module is about the first half of that sentence.

---

## Why this matters

Authentication is the front door of every SaaS product. Get it wrong and you don't have a SaaS — you have a public API. Get it right and the rest of the system can be confidently built on top of it: billing, RBAC, audit logs, multi-tenancy.

In this module we will build the *minimum* auth surface a real product needs:

- Register with email + password
- Login
- Refresh access tokens
- Logout
- Inspect the current user (`/me`)

Along the way we will cover the theory you actually need to defend choices in a code review: hashing vs encryption, JWT structure, access vs refresh tokens, and the most common ways teams shoot themselves in the foot.

---

## Learning objectives

By the end of this module you can:

1. Explain the difference between authentication and authorization
2. Hash passwords with `bcrypt` (or `bcryptjs`) at cost factor 12
3. Implement a registration flow that validates, deduplicates, hashes, persists
4. Implement a login flow that compares hashes in constant time
5. Describe JWT structure (`header.payload.signature`), `HS256` vs `RS256`, and `base64url`
6. Issue short-lived access tokens (15 min) and long-lived refresh tokens (7–30 days)
7. Verify and decode JWTs without leaking secrets into logs
8. Store refresh tokens in a database so they can be revoked
9. Choose between cookies and the `Authorization` header for token transport
10. Identify the top five auth anti-patterns and avoid them

---

## 1. Authentication vs authorization

| Concern   | Question it answers                  | Example            |
| --------- | ------------------------------------ | ------------------ |
| AuthN     | **Who** is this user?                | Login with email   |
| AuthZ     | **What** can this user do?           | `requireRole(admin)` |

Module 22 is authN. Module 23 is authZ. Do not let them leak into each other.

A token that says "I am `sub=user_123`" is authentication. A middleware that says "user_123 may not delete `workspace_456` because they are not an admin" is authorization.

---

## 2. Password hashing — bcrypt

Passwords are **never** stored in plaintext. We hash them with a slow, salted, adaptive function. `bcrypt` is the most battle-tested choice. `argon2id` is the modern recommendation for new projects; we use bcrypt here because the curriculum's mission is continuity with the existing auth module.

### `bcrypt` vs `bcryptjs`

| Library       | Native? | Speed | Install |
| ------------- | ------- | ----- | ------- |
| `bcrypt`      | yes (C++) | fast | may need build tools |
| `bcryptjs`    | pure JS | slower but fine | zero deps |

If you are on Windows or behind a corporate proxy without a C++ toolchain, use `bcryptjs`. The API is the same.

```ts
// src/modules/auth/password.ts
import bcrypt from "bcryptjs";

const COST = 12; // ~250ms on a modern CPU. Tune by measuring.

export async function hashPassword(plain: string): Promise<string> {
  return bcrypt.hash(plain, COST);
}

export async function verifyPassword(plain: string, hash: string): Promise<boolean> {
  // bcrypt.compare is constant-time relative to *matching length* — do not
  // leak the existence of the user before you call it.
  return bcrypt.compare(plain, hash);
}
```

**Rule of thumb:** cost factor ≈ 250 ms on your production hardware. Re-benchmark every 18 months.

---

## 3. The user schema (forward ref to Module 23)

```ts
// src/db/schema.ts (additions)
import {
  pgTable,
  uuid,
  text,
  timestamp,
  jsonb,
  index,
} from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: uuid("id").defaultRandom().primaryKey(),
  email: text("email").notNull().unique(),
  passwordHash: text("password_hash").notNull(),
  status: text("status").notNull().default("active"),
  createdAt: timestamp("created_at", { withTimezone: true })
    .notNull()
    .defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true })
    .notNull()
    .defaultNow(),
});

export const refreshTokens = pgTable(
  "refresh_tokens",
  {
    id: uuid("id").defaultRandom().primaryKey(),
    userId: uuid("user_id")
      .notNull()
      .references(() => users.id, { onDelete: "cascade" }),
    tokenHash: text("token_hash").notNull().unique(),
    // Useful metadata for security audits
    userAgent: text("user_agent"),
    ip: text("ip"),
    expiresAt: timestamp("expires_at", { withTimezone: true }).notNull(),
    revokedAt: timestamp("revoked_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true })
      .notNull()
      .defaultNow(),
  },
  (t) => ({
    userIdx: index("refresh_tokens_user_idx").on(t.userId),
  }),
);
```

Note that we store the **hash** of the refresh token, not the token itself. If the database leaks, the attacker cannot replay tokens.

---

## 4. JWT theory — what is actually inside?

A JWT is `base64url(header) + "." + base64url(payload) + "." + base64url(signature)`.

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyXzEyMyIsImVtYWlsIjoiYUBiLmNvbSIsImlhdCI6MTcwMDAwMDAwMCwiZXhwIjoxNzAwMDAwOTAwfQ.5K3q...
```

```ts
// Decoded header
{ "alg": "HS256", "typ": "JWT" }

// Decoded payload
{
  "sub": "user_123",
  "email": "a@b.com",
  "iat": 1700000000,   // issued at
  "exp": 1700000900    // expires at (15 min later)
}
```

The signature is `HMAC-SHA256(base64url(header) + "." + base64url(payload), secret)`.

| Algorithm | Key type         | When to use                                   |
| --------- | ---------------- | --------------------------------------------- |
| `HS256`   | shared secret    | One service signs and verifies. Simpler.      |
| `RS256`   | RSA private/public | Multiple services verify, only one signs. |

For a single-service backend, `HS256` is fine. Once you need a separate auth service that other services verify against, switch to `RS256` (or `EdDSA`).

**Crucial caveat:** anyone with the secret can forge tokens. Anyone *without* the secret can read the payload — it is **not encrypted**, only signed. Put no secrets in the payload.

---

## 5. Token utilities

```ts
// src/modules/auth/tokens.ts
import jwt from "jsonwebtoken";
import crypto from "node:crypto";
import { env } from "../../config/env.js";

export interface AccessTokenClaims {
  sub: string;
  email: string;
}

const ACCESS_TTL = "15m";
const REFRESH_TTL_DAYS = 30;

export function signAccessToken(claims: AccessTokenClaims): string {
  return jwt.sign(claims, env.JWT_ACCESS_SECRET, {
    algorithm: "HS256",
    expiresIn: ACCESS_TTL,
  });
}

export function verifyAccessToken(token: string): AccessTokenClaims {
  const payload = jwt.verify(token, env.JWT_ACCESS_SECRET, {
    algorithms: ["HS256"],
  });
  if (typeof payload === "string" || !("sub" in payload) || !("email" in payload)) {
    throw new Error("Invalid access token payload");
  }
  return { sub: payload.sub as string, email: payload.email as string };
}

// Opaque, URL-safe random token (NOT a JWT). Stored hashed in DB.
export function generateRefreshToken(): { token: string; hash: string } {
  const token = crypto.randomBytes(48).toString("base64url");
  const hash = crypto.createHash("sha256").update(token).digest("hex");
  return { token, hash };
}

export function hashRefreshToken(token: string): string {
  return crypto.createHash("sha256").update(token).digest("hex");
}

export function refreshExpiry(): Date {
  const d = new Date();
  d.setDate(d.getDate() + REFRESH_TTL_DAYS);
  return d;
}
```

Notice the refresh token is **opaque**. The client receives a random string; the server stores its SHA-256. There is no payload to leak and no algorithm to misuse.

---

## 6. The auth service

```ts
// src/modules/auth/auth.service.ts
import { eq, and, isNull } from "drizzle-orm";
import { db } from "../../db/client.js";
import { users, refreshTokens } from "../../db/schema.js";
import {
  hashPassword,
  verifyPassword,
} from "./password.js";
import {
  signAccessToken,
  verifyAccessToken,
  generateRefreshToken,
  hashRefreshToken,
  refreshExpiry,
} from "./tokens.js";
import {
  ConflictError,
  UnauthorizedError,
  ValidationError,
} from "../../lib/http-error.js";

export interface RegisterInput {
  email: string;
  password: string;
}

export interface AuthResult {
  user: { id: string; email: string };
  accessToken: string;
  refreshToken: string;
}

async function issueTokenPair(userId: string, email: string, meta: {
  userAgent?: string;
  ip?: string;
}): Promise<{ accessToken: string; refreshToken: string }> {
  const accessToken = signAccessToken({ sub: userId, email });
  const { token, hash } = generateRefreshToken();

  await db.insert(refreshTokens).values({
    userId,
    tokenHash: hash,
    userAgent: meta.userAgent,
    ip: meta.ip,
    expiresAt: refreshExpiry(),
  });

  return { accessToken, refreshToken: token };
}

export const authService = {
  async register(input: RegisterInput, meta: { userAgent?: string; ip?: string }): Promise<AuthResult> {
    const existing = await db
      .select({ id: users.id })
      .from(users)
      .where(eq(users.email, input.email))
      .limit(1);

    if (existing.length > 0) {
      throw new ConflictError("Email already registered");
    }

    const passwordHash = await hashPassword(input.password);
    const [created] = await db
      .insert(users)
      .values({ email: input.email, passwordHash })
      .returning({ id: users.id, email: users.email });

    const tokens = await issueTokenPair(created.id, created.email, meta);
    return { user: created, ...tokens };
  },

  async login(input: RegisterInput, meta: { userAgent?: string; ip?: string }): Promise<AuthResult> {
    const [user] = await db
      .select()
      .from(users)
      .where(eq(users.email, input.email))
      .limit(1);

    if (!user || user.status !== "active") {
      // Same error either way: do not leak which one is wrong.
      throw new UnauthorizedError("Invalid email or password");
    }

    const ok = await verifyPassword(input.password, user.passwordHash);
    if (!ok) {
      throw new UnauthorizedError("Invalid email or password");
    }

    const tokens = await issueTokenPair(user.id, user.email, meta);
    return { user: { id: user.id, email: user.email }, ...tokens };
  },

  async refresh(refreshToken: string, meta: { userAgent?: string; ip?: string }): Promise<AuthResult> {
    const hash = hashRefreshToken(refreshToken);
    const [row] = await db
      .select({
        id: refreshTokens.id,
        userId: refreshTokens.userId,
        expiresAt: refreshTokens.expiresAt,
        revokedAt: refreshTokens.revokedAt,
      })
      .from(refreshTokens)
      .where(and(eq(refreshTokens.tokenHash, hash), isNull(refreshTokens.revokedAt)))
      .limit(1);

    if (!row || row.expiresAt < new Date()) {
      throw new UnauthorizedError("Refresh token invalid");
    }

    // Rotation: revoke old, issue new (covered in detail in Module 23)
    await db
      .update(refreshTokens)
      .set({ revokedAt: new Date() })
      .where(eq(refreshTokens.id, row.id));

    const [user] = await db
      .select({ id: users.id, email: users.email })
      .from(users)
      .where(eq(users.id, row.userId))
      .limit(1);

    if (!user) throw new UnauthorizedError("User no longer exists");

    const tokens = await issueTokenPair(user.id, user.email, meta);
    return { user, ...tokens };
  },

  async logout(refreshToken: string): Promise<void> {
    const hash = hashRefreshToken(refreshToken);
    await db
      .update(refreshTokens)
      .set({ revokedAt: new Date() })
      .where(eq(refreshTokens.tokenHash, hash));
  },

  async me(accessToken: string) {
    let claims;
    try {
      claims = verifyAccessToken(accessToken);
    } catch {
      throw new UnauthorizedError("Invalid or expired access token");
    }
    const [user] = await db
      .select({ id: users.id, email: users.email, status: users.status })
      .from(users)
      .where(eq(users.id, claims.sub))
      .limit(1);
    if (!user || user.status !== "active") {
      throw new UnauthorizedError("User no longer active");
    }
    return user;
  },
};
```

---

## 7. The auth router

```ts
// src/modules/auth/auth.router.ts
import { Router } from "express";
import { z } from "zod";
import { validate } from "../../lib/validate.js";
import { catchAsync } from "../../lib/catch-async.js";
import { ok } from "../../lib/response.js";
import { authService } from "./auth.service.js";

export const authRouter: Router = Router();

const registerSchema = z.object({
  email: z.string().email().max(254),
  password: z.string().min(10).max(128),
});

const loginSchema = registerSchema; // same shape

const refreshSchema = z.object({
  refreshToken: z.string().min(20),
});

const REFRESH_COOKIE = "rt";

function cookieOpts() {
  return {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "lax" as const,
    path: "/auth",
    maxAge: 30 * 24 * 60 * 60 * 1000,
  };
}

authRouter.post(
  "/register",
  validate(registerSchema),
  catchAsync(async (req, res) => {
    const result = await authService.register(
      req.body as z.infer<typeof registerSchema>,
      { userAgent: req.headers["user-agent"], ip: req.ip },
    );
    res.cookie(REFRESH_COOKIE, result.refreshToken, cookieOpts());
    return ok(res, { user: result.user, accessToken: result.accessToken });
  }),
);

authRouter.post(
  "/login",
  validate(loginSchema),
  catchAsync(async (req, res) => {
    const result = await authService.login(
      req.body as z.infer<typeof loginSchema>,
      { userAgent: req.headers["user-agent"], ip: req.ip },
    );
    res.cookie(REFRESH_COOKIE, result.refreshToken, cookieOpts());
    return ok(res, { user: result.user, accessToken: result.accessToken });
  }),
);

authRouter.post(
  "/refresh",
  validate(refreshSchema),
  catchAsync(async (req, res) => {
    const body = req.body as z.infer<typeof refreshSchema>;
    // Allow refresh token via cookie OR body for client flexibility
    const token = body.refreshToken ?? req.cookies?.[REFRESH_COOKIE];
    if (!token) throw new (await import("../../lib/http-error.js")).UnauthorizedError("Missing refresh token");

    const result = await authService.refresh(token, {
      userAgent: req.headers["user-agent"],
      ip: req.ip,
    });
    res.cookie(REFRESH_COOKIE, result.refreshToken, cookieOpts());
    return ok(res, { user: result.user, accessToken: result.accessToken });
  }),
);

authRouter.post(
  "/logout",
  catchAsync(async (req, res) => {
    const token = req.cookies?.[REFRESH_COOKIE] ?? req.body?.refreshToken;
    if (token) await authService.logout(token);
    res.clearCookie(REFRESH_COOKIE, { path: "/auth" });
    return ok(res, { loggedOut: true });
  }),
);

authRouter.get(
  "/me",
  catchAsync(async (req, res) => {
    const header = req.headers.authorization;
    if (!header?.startsWith("Bearer ")) {
      throw new (await import("../../lib/http-error.js")).UnauthorizedError("Missing access token");
    }
    const me = await authService.me(header.slice("Bearer ".length));
    return ok(res, me);
  }),
);
```

---

## 8. Cookies vs Authorization header

| Transport            | Pros                                                | Cons                                                  |
| -------------------- | --------------------------------------------------- | ----------------------------------------------------- |
| **httpOnly cookie**  | Immune to XSS exfiltration; browser sends it for you | Vulnerable to CSRF (mitigate with `sameSite=lax`)    |
| **Authorization header** | Easy to use from SPAs and CLI                  | Must be stored client-side; XSS-readable              |

**Recommended pattern:**

- **Access token** in the response body. Frontend stores it in memory (or `sessionStorage` if it must survive a refresh). Never `localStorage` for an access token.
- **Refresh token** in an `httpOnly`, `secure`, `sameSite=lax` cookie scoped to `/auth`. The browser sends it automatically on the `/auth/refresh` call. CSRF is mitigated because the cookie is `sameSite=lax`.

---

## 9. Wire it in

```ts
// src/app.ts (add)
import cookieParser from "cookie-parser";
import { authRouter } from "./modules/auth/auth.router.js";

// after express.json(...)
app.use(cookieParser());
app.use("/auth", authRouter);
```

```bash
pnpm add bcryptjs jsonwebtoken cookie-parser
pnpm add -D @types/bcryptjs @types/jsonwebtoken @types/cookie-parser
```

---

## Common mistakes

1. **Storing passwords as plaintext.** The number of times I have seen it in 2026 is non-zero. Use bcrypt.
2. **Returning the JWT to `localStorage`.** Every XSS becomes an account takeover. Use httpOnly cookies for long-lived tokens.
3. **Putting the user's password hash, MFA secret, or PII inside the JWT.** The payload is base64-decodable by anyone.
4. **No expiration.** A leaked token is valid forever. Always set `exp`.
5. **Short secrets (`"secret"`).** `HS256` keys should be at least 256 bits of entropy. Generate them with `crypto.randomBytes(32).toString("hex")`.
6. **No revocation.** Access tokens can't be revoked cheaply (that is the trade-off of stateless JWTs). Refresh tokens *can* — store them and check `revokedAt` on every refresh.
7. **Re-using the same secret for `access` and `refresh`.** Compromise of one compromises both. Two different secrets, two different risks.
8. **Logging tokens.** Make sure your `pino` config excludes `Authorization`, `Cookie`, and any field literally named `token` / `password`.

---

## Exercise

**Goal:** Wire up auth end-to-end and prove the full lifecycle works.

1. Add `bcryptjs`, `jsonwebtoken`, `cookie-parser` and their `@types` to the project.
2. Generate and run the migration for the `refresh_tokens` table.
3. Register a user via `POST /auth/register`. Capture the `accessToken` from the response and the `rt` cookie from headers.
4. Call `GET /auth/me` with `Authorization: Bearer <accessToken>` — confirm `200`.
5. Call `POST /auth/refresh` — confirm you get a **new** access token and the cookie value has changed (rotation).
6. Re-use the **old** refresh token — confirm you get `401`.
7. Call `POST /auth/logout` — confirm the cookie is cleared and a subsequent refresh fails.
8. Try registering with the same email twice — confirm `409` with `code: "CONFLICT"`.
9. Try logging in with a wrong password — confirm you get `401` with the same generic message as a wrong email.

**Bonus:** add a Vitest test that monkey-patches `bcrypt.compare` to throw, and confirm the error envelope is still well-formed.

---

## AI Industry Note (2026)

In 2026, AI assistants can produce a working JWT signup/refresh/logout flow in one prompt. They will also, on the same prompt, happily:

- Suggest putting the access token in `localStorage`
- Use the same secret for access and refresh tokens
- Forget to hash the refresh token before storing it
- Set the JWT expiration to "never"
- Not check `token.revokedAt` on refresh

These are not hypothetical mistakes. They are the *median* output of unguided AI in 2025, and they appear in production code with depressing regularity.

What separates a senior engineer is the **threat model in your head**:

- "If this database leaks tomorrow, what does the attacker get?" — answer: nothing useful, because refresh tokens are hashed and passwords are bcrypt.
- "If the access token leaks today, how long does the attacker have?" — answer: at most 15 minutes, plus the next refresh is rejected because it's rotated.
- "If a user's laptop is compromised, what can the attacker steal?" — answer: an access token (short-lived) and a refresh cookie (rotated on first use from a different IP). The session is recoverable.

A senior engineer in 2026 is not someone who can recall `jsonwebtoken`'s API. It's someone who can run that threat model in their head while reviewing AI-generated code, and who can write the missing lines — the constant-time comparisons, the rotation logic, the revocation checks — that the AI will leave out.

You are not here to type code. You are here to *decide what code should exist*.