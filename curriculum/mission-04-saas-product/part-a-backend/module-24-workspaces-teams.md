# Module 24: Workspace, Team & Membership Management

> "Every serious SaaS bug I've ever seen started with a missing tenant check." — every senior engineer, ever

---

## 1. Why this matters

A SaaS product is a building that hosts many companies inside it. Each company is a **tenant**, and in this product the tenant boundary is the **workspace**. Once a user is "in" a workspace, they see data scoped to that workspace. Once they're "out", they see *nothing*.

The single most common class of vulnerability in B2B SaaS is **cross-tenant data leakage** — User A in Workspace X somehow reading or writing data from Workspace Y. It is almost always caused by a missing `WHERE workspace_id = ?` clause. We'll make sure that never happens in this codebase.

Beyond security, the workspace model is also a *product* concept. Users expect to:
- create workspaces, name them, switch between them
- invite teammates by email
- have roles (owner / admin / member)
- transfer ownership when the founder leaves
- revoke access when someone exits the company

This module is about getting all of that right — model, API, authorization, and edge cases.

---

## 2. Learning objectives

By the end of this module you can:

1. Model a multi-tenant data layout where `workspace_id` is on every tenant-scoped table.
2. Implement an **authorization guard** that loads `(user, workspace, membership)` from the URL and rejects unauthorized access.
3. Build an **invitation system** with secure single-use tokens, expiration, and acceptance flow.
4. Implement the membership status state machine: `invited → active → suspended → removed`.
5. Enforce workspace-level **RBAC** (owner / admin / member) cleanly.
6. Handle the "last owner" edge case and ownership transfer.
7. Apply the **tenant-scoping rule** to every query — even the ones you write in a hurry.

---

## 3. Conceptual model

### 3.1 The tenant boundary

A workspace is the tenant. Every row that belongs to a workspace carries `workspace_id`:

```
users               (global identity)
workspaces          (the tenant)
workspace_members   (user × workspace × role × status)  <-- the join
invitations         (pending invites scoped to a workspace)
projects            workspace_id NOT NULL
tasks               workspace_id NOT NULL   (denormalized for tenant-scoping speed)
comments            workspace_id NOT NULL
files               workspace_id NOT NULL
```

We denormalize `workspace_id` onto `tasks` and `comments` even though they belong to a `project` which already has `workspace_id`. This is deliberate — it lets the authorization layer issue a single index lookup instead of a join on every request.

### 3.2 The schema (Drizzle)

```ts
// src/db/schema/workspaces.ts
import { pgTable, uuid, text, timestamp, pgEnum, index, uniqueIndex } from 'drizzle-orm/pg-core';
import { users } from './users';

export const memberRole = pgEnum('member_role', ['owner', 'admin', 'member']);
export const memberStatus = pgEnum('member_status', ['invited', 'active', 'suspended', 'removed']);

export const workspaces = pgTable('workspaces', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  slug: text('slug').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow().notNull(),
}, (t) => ({
  slugUnique: uniqueIndex('workspaces_slug_uq').on(t.slug),
}));

export const workspaceMembers = pgTable('workspace_members', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspaces.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  role: memberRole('role').notNull().default('member'),
  status: memberStatus('status').notNull().default('active'),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
}, (t) => ({
  uniqMember: uniqueIndex('workspace_members_uniq').on(t.workspaceId, t.userId),
  byUser: index('workspace_members_by_user').on(t.userId),
}));

export const invitations = pgTable('invitations', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspaces.id, { onDelete: 'cascade' }),
  email: text('email').notNull(),
  role: memberRole('role').notNull().default('member'),
  tokenHash: text('token_hash').notNull(),          // sha256 of the token
  invitedBy: uuid('invited_by').notNull().references(() => users.id),
  expiresAt: timestamp('expires_at', { withTimezone: true }).notNull(),
  acceptedAt: timestamp('accepted_at', { withTimezone: true }),
  revokedAt: timestamp('revoked_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
}, (t) => ({
  uniqPending: uniqueIndex('invitations_uniq_pending')
    .on(t.workspaceId, t.email)
    .where(sql`accepted_at IS NULL AND revoked_at IS NULL`),
}));
```

Note the **partial unique index** on pending invitations — it prevents two admins from accidentally inviting the same email twice.

### 3.3 Authorization guard — the most important code in this module

```ts
// src/middleware/workspaceAuth.ts
import type { Request, Response, NextFunction } from 'express';
import { eq, and } from 'drizzle-orm';
import { db } from '../db';
import { workspaceMembers, workspaces } from '../db/schema/workspaces';
import { HttpError } from '../lib/errors';

export type WorkspaceRole = 'owner' | 'admin' | 'member';

declare module 'express-serve-static-core' {
  interface Request {
    workspace?: { id: string; slug: string; role: WorkspaceRole };
  }
}

/**
 * Loads the workspace from the URL, verifies the user is a member,
 * and attaches the membership to the request.
 *
 * This is the ONLY way routes should access workspace-scoped data.
 * If you find yourself writing `where(eq(table.workspaceId, x))` in a route,
 * stop — use the workspace from `req.workspace` instead.
 */
export async function requireWorkspaceMember(req: Request, _res: Response, next: NextFunction) {
  try {
    const userId = req.user!.id;
    const workspaceIdOrSlug = req.params.workspaceId;
    if (!workspaceIdOrSlug) throw new HttpError(400, 'workspace id required');

    const ws = await db.query.workspaces.findFirst({
      where: (w, { or, eq }) => or(eq(w.id, workspaceIdOrSlug), eq(w.slug, workspaceIdOrSlug)),
    });
    if (!ws) throw new HttpError(404, 'workspace not found');

    const membership = await db.query.workspaceMembers.findFirst({
      where: (m, { and, eq }) => and(eq(m.workspaceId, ws.id), eq(m.userId, userId), eq(m.status, 'active')),
    });
    if (!membership) throw new HttpError(403, 'not a member of this workspace');

    req.workspace = { id: ws.id, slug: ws.slug, role: membership.role };
    next();
  } catch (err) {
    next(err);
  }
}

/** Role-gated variant — for routes that only owners/admins can hit */
export function requireRole(...allowed: WorkspaceRole[]) {
  return (req: Request, _res: Response, next: NextFunction) => {
    if (!req.workspace) return next(new HttpError(500, 'guard misconfigured'));
    if (!allowed.includes(req.workspace.role)) {
      return next(new HttpError(403, `requires role: ${allowed.join(' or ')}`));
    }
    next();
  };
}
```

Wiring it up:

```ts
// src/routes/workspaces.ts
import { Router } from 'express';
import { requireAuth } from '../middleware/auth';
import { requireWorkspaceMember, requireRole } from '../middleware/workspaceAuth';

const router = Router({ mergeParams: true });
router.use(requireAuth, requireWorkspaceMember);

router.get('/workspaces/:workspaceId', (req, res) => res.json({ workspace: req.workspace }));
router.get('/workspaces/:workspaceId/members', requireRole('owner', 'admin'), listMembers);
router.post('/workspaces/:workspaceId/invitations', requireRole('owner', 'admin'), createInvitation);
router.delete('/workspaces/:workspaceId/members/:userId', requireRole('owner', 'admin'), removeMember);
```

### 3.4 Creating and switching workspaces

```ts
// src/services/workspaceService.ts
import { db } from '../db';
import { workspaces, workspaceMembers } from '../db/schema/workspaces';
import { HttpError } from '../lib/errors';

export async function createWorkspace(userId: string, name: string) {
  const slug = await uniqueSlug(name);
  return db.transaction(async (tx) => {
    const [ws] = await tx.insert(workspaces).values({ name, slug }).returning();
    // The creator is the owner — always.
    await tx.insert(workspaceMembers).values({
      workspaceId: ws.id, userId, role: 'owner', status: 'active',
    });
    return ws;
  });
}

async function uniqueSlug(name: string): Promise<string> {
  const base = name.toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-|-$/g, '').slice(0, 40);
  for (let i = 0; i < 5; i++) {
    const candidate = i === 0 ? base : `${base}-${crypto.randomUUID().slice(0, 4)}`;
    const exists = await db.query.workspaces.findFirst({ where: (w, { eq }) => eq(w.slug, candidate) });
    if (!exists) return candidate;
  }
  throw new HttpError(500, 'could not allocate slug');
}

export async function listMyWorkspaces(userId: string) {
  return db
    .select({ workspace: workspaces, role: workspaceMembers.role })
    .from(workspaceMembers)
    .innerJoin(workspaces, eq(workspaces.id, workspaceMembers.workspaceId))
    .where(and(eq(workspaceMembers.userId, userId), eq(workspaceMembers.status, 'active')));
}
```

### 3.5 Invitation tokens: secure, single-use, expiring

We never store the raw token. We store its `sha256` hash and email the raw token in the link.

```ts
// src/services/invitationService.ts
import { createHash, randomBytes } from 'node:crypto';
import { and, eq, isNull } from 'drizzle-orm';
import { db } from '../db';
import { invitations, workspaceMembers } from '../db/schema/workspaces';
import { users } from '../db/schema/users';
import { HttpError } from '../lib/errors';

const TOKEN_TTL_MS = 1000 * 60 * 60 * 24 * 7; // 7 days

export async function createInvitation(opts: {
  workspaceId: string; email: string; role: 'admin' | 'member'; invitedBy: string;
}) {
  const rawToken = randomBytes(32).toString('base64url');
  const tokenHash = createHash('sha256').update(rawToken).digest('hex');

  await db.insert(invitations).values({
    workspaceId: opts.workspaceId,
    email: opts.email.toLowerCase(),
    role: opts.role,
    tokenHash,
    invitedBy: opts.invitedBy,
    expiresAt: new Date(Date.now() + TOKEN_TTL_MS),
  });

  // Return the raw token ONCE — the caller is responsible for emailing it.
  return { token: rawToken, expiresInDays: 7 };
}

export async function acceptInvitation(rawToken: string, acceptingUserId: string) {
  const tokenHash = createHash('sha256').update(rawToken).digest('hex');

  return db.transaction(async (tx) => {
    const inv = await tx.query.invitations.findFirst({
      where: (i, { and, eq, isNull }) =>
        and(eq(i.tokenHash, tokenHash), isNull(i.acceptedAt), isNull(i.revokedAt)),
      with: { workspace: true },
    });
    if (!inv) throw new HttpError(400, 'invalid invitation');
    if (inv.expiresAt.getTime() < Date.now()) throw new HttpError(400, 'invitation expired');

    // Optional: ensure the accepting user matches the invited email.
    const user = await tx.query.users.findFirst({ where: (u, { eq }) => eq(u.id, acceptingUserId) });
    if (!user || user.email.toLowerCase() !== inv.email) {
      throw new HttpError(403, 'this invitation was sent to a different email');
    }

    // Idempotent: if already a member, just mark the invite accepted.
    await tx.insert(workspaceMembers)
      .values({ workspaceId: inv.workspaceId, userId: acceptingUserId, role: inv.role, status: 'active' })
      .onConflictDoUpdate({
        target: [workspaceMembers.workspaceId, workspaceMembers.userId],
        set: { role: inv.role, status: 'active' },
      });

    await tx.update(invitations).set({ acceptedAt: new Date() }).where(eq(invitations.id, inv.id));
    return inv.workspace;
  });
}

export async function revokeInvitation(workspaceId: string, invitationId: string) {
  const result = await db
    .update(invitations)
    .set({ revokedAt: new Date() })
    .where(and(eq(invitations.id, invitationId), eq(invitations.workspaceId, workspaceId), isNull(invitations.acceptedAt)))
    .returning({ id: invitations.id });
  if (result.length === 0) throw new HttpError(404, 'invitation not found or already used');
}
```

### 3.6 Membership state machine

```
                ┌──────────────┐
                │   invited    │   (via invitation row; no membership yet)
                └──────┬───────┘
                       │ acceptInvitation()
                       ▼
                ┌──────────────┐
        ┌──────►│    active    │◄──────┐
        │       └──────┬───────┘       │
        │ reactivate   │               │ transfer
        │              ▼               │
        │       ┌──────────────┐       │
        │       │  suspended   │       │
        │       └──────┬───────┘       │
        │              │ remove        │
        │              ▼               │
        │       ┌──────────────┐       │
        │       │   removed    │───────┘  (soft state; row retained for audit)
        │       └──────────────┘
        └──────────────────── (any active user is allowed to be demoted)
```

```ts
export async function setMembershipStatus(workspaceId: string, userId: string, status: 'active' | 'suspended' | 'removed') {
  return db.transaction(async (tx) => {
    if (status === 'removed') await assertNotLastOwner(tx, workspaceId, userId);
    const [row] = await tx.update(workspaceMembers)
      .set({ status })
      .where(and(eq(workspaceMembers.workspaceId, workspaceId), eq(workspaceMembers.userId, userId)))
      .returning();
    if (!row) throw new HttpError(404, 'membership not found');
    return row;
  });
}

async function assertNotLastOwner(tx: typeof db, workspaceId: string, userId: string) {
  const owners = await tx
    .select({ id: workspaceMembers.id })
    .from(workspaceMembers)
    .where(and(
      eq(workspaceMembers.workspaceId, workspaceId),
      eq(workspaceMembers.role, 'owner'),
      eq(workspaceMembers.status, 'active'),
    ));
  const isThisUserOwner = owners.some((o) => o.id === userId);
  if (isThisUserOwner && owners.length === 1) {
    throw new HttpError(409, 'cannot remove the last owner; transfer ownership first');
  }
}
```

### 3.7 Ownership transfer

```ts
export async function transferOwnership(workspaceId: string, fromUserId: string, toUserId: string) {
  if (fromUserId === toUserId) throw new HttpError(400, 'already the owner');
  return db.transaction(async (tx) => {
    const target = await tx.query.workspaceMembers.findFirst({
      where: (m, { and, eq }) => and(eq(m.workspaceId, workspaceId), eq(m.userId, toUserId), eq(m.status, 'active')),
    });
    if (!target) throw new HttpError(404, 'target user is not an active member');

    await tx.update(workspaceMembers)
      .set({ role: 'owner' })
      .where(and(eq(workspaceMembers.workspaceId, workspaceId), eq(workspaceMembers.userId, toUserId)));

    // Demote the old owner to admin (they keep access, lose ownership).
    await tx.update(workspaceMembers)
      .set({ role: 'admin' })
      .where(and(eq(workspaceMembers.workspaceId, workspaceId), eq(workspaceMembers.userId, fromUserId)));

    return { ok: true };
  });
}
```

### 3.8 The tenant-scoping rule

Every query that touches a tenant-scoped table MUST filter on `workspace_id` and that workspace id MUST come from `req.workspace.id`, never from the request body or query string.

```ts
// GOOD
const tasks = await db.query.tasks.findMany({
  where: (t, { eq }) => and(eq(t.workspaceId, req.workspace!.id), eq(t.projectId, projectId)),
});

// BAD — never trust the client
const tasks = await db.query.tasks.findMany({ where: (t, { eq }) => eq(t.id, taskIdFromBody) });
```

A useful pattern is a tiny helper that "freezes" the workspace id:

```ts
export function inWorkspace<T extends { workspaceId: any }>(table: T, workspaceId: string) {
  return eq(table.workspaceId, workspaceId);
}
```

---

## 4. Common mistakes

1. **Joining through the parent for tenant checks.** "It's a task of a project of a workspace, so I can fetch the project first and check" — too late. The attacker can put any project id in the URL. Always check the *direct* `workspace_id` row.

2. **Storing raw invitation tokens in the database.** If your `invitations` table leaks (logs, backups, SQL injection), an attacker can accept any pending invite. Always store `sha256(token)` and treat the raw token as a one-time password.

3. **Soft delete via `status = 'removed'` but still allowing logins.** A "removed" member with `status = 'removed'` should not be able to authenticate against the workspace. The guard already filters on `status = 'active'` — make sure every guard does.

4. **Forgetting to demote owners on transfer.** A naive "set new owner" leaves the old owner as an *owner* alongside the new one. There can be more than one owner, but it should be an intentional choice, not a bug.

5. **Re-using unique slugs across soft-deleted workspaces.** Either hard-delete workspaces (and cascade) or include deleted ones in the slug uniqueness check. The latter is what the schema above does.

6. **Putting `workspace_id` in the request body for create endpoints.** It must come from the URL or from the authenticated session. If the client can specify it, they can create data in someone else's workspace.

---

## 5. Exercise

**Build the workspace API end to end.**

1. Implement the schema in `src/db/schema/workspaces.ts` and run the migration.
2. Implement `workspaceService.createWorkspace` and the `POST /workspaces` route.
3. Implement `requireWorkspaceMember` and wire it onto `GET /workspaces/:workspaceId`.
4. Implement `createInvitation`, `acceptInvitation`, and `revokeInvitation` with the email-sending step stubbed (log to console for now).
5. Implement `setMembershipStatus` and `transferOwnership`. Write a test that asserts you cannot remove the last owner.
6. **Stretch:** Add a global "active workspace" concept: when a user logs in, they pick a workspace and the choice is stored in a signed cookie. The guard reads from the cookie if the URL doesn't specify one.

**Acceptance criteria:**
- An authenticated user can create a workspace and is automatically its owner.
- A user who is not a member of workspace X cannot read any route scoped to X (returns 403).
- An invitation token works exactly once; reuse returns 400.
- A user cannot be removed if they are the only active owner.
- Every `tasks` / `projects` / `files` query in your routes is scoped by `req.workspace.id`.

---

## 6. AI Industry Note (2026)

In 2026, AI coding assistants will happily generate the bulk of a multi-tenant CRUD API in seconds — and they will, just as happily, write a query that forgets the `WHERE workspace_id = ?` clause. The shape of the bug is consistent: confident code, missing tenant filter, data leakage. Modern review pipelines now include **automated multi-tenant linters** that scan query ASTs and flag any read or write that isn't parameterized by the request's tenant. Some teams go further and reject PRs that introduce raw SQL without a tenant context wrapper.

The lesson for you: when you delegate boilerplate to an agent, you keep the *invariants*. Tenant scoping, role checks, and the "last owner" rule are exactly the things a model will not invent unless you tell it to. Put them in your architecture as a middleware and a transaction-level assertion, not as comments and good intentions.
