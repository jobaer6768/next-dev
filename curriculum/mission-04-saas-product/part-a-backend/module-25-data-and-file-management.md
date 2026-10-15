# Module 25: Data & File Management

> "Soft delete, audit fields, idempotency keys. These three patterns are worth their weight in on-call hours."

---

## 1. Why this matters

Up to this point you've built the *shell* of a SaaS — auth, workspaces, members. Now we put the *substance* in: projects, tasks, comments, and the files people attach to them. This is where the product becomes useful and where the engineering decisions start compounding.

Three decisions in this module will quietly affect every feature you build for the rest of the product:

1. **Soft delete vs hard delete.** Delete is rarely delete. Users expect "undo", they expect compliance (GDPR, customer churn), and they expect analytics to survive deletions. We'll model `deletedAt` on every domain table.
2. **Audit fields.** Every row will tell you *who* created it, *when*, *who* last touched it, and *when*. This is non-negotiable in B2B. Customers will ask "who changed this?" and you will answer in SQL, not in guesswork.
3. **Idempotency.** A network blip during a file upload means the client retries. Without an idempotency key, you create two files. With one, the second request returns the first.

Then there's the file system itself. Files are not rows in your database — they're blobs in object storage — and the integration has to be careful, signed, and quota-aware.

---

## 2. Learning objectives

By the end of this module you can:

1. Model projects, tasks, and comments with Drizzle, including soft delete and audit fields.
2. Wire up Multer with appropriate storage strategies (memory vs disk) by file size.
3. Upload to Cloudinary or S3 with proper content-type, folder scoping, and metadata.
4. Serve files via **signed URLs** that expire — never expose the storage key directly.
5. Enforce per-workspace quotas for file size, file count, and total bytes.
6. Implement an idempotency-key contract for write endpoints.
7. Build CRUD endpoints for files attached to tasks, including safe deletion.

---

## 3. Conceptual model

### 3.1 The base pattern: every domain table looks like this

```ts
// src/db/schema/_common.ts
import { timestamp, uuid } from 'drizzle-orm/pg-core';

export const auditFields = {
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow().notNull(),
  createdBy: uuid('created_by'),
  updatedBy: uuid('updated_by'),
  deletedAt: timestamp('deleted_at', { withTimezone: true }),
};

export const softDeleteFilter = (table: { deletedAt: any }) => ({ isNull: (col: any) => col });
```

Then the actual models are short:

```ts
// src/db/schema/projects.ts
import { pgTable, uuid, text, index } from 'drizzle-orm/pg-core';
import { workspaces } from './workspaces';
import { auditFields } from './_common';

export const projects = pgTable('projects', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspaces.id, { onDelete: 'cascade' }),
  name: text('name').notNull(),
  description: text('description'),
  ...auditFields,
}, (t) => ({
  byWorkspace: index('projects_by_workspace').on(t.workspaceId),
}));
```

```ts
// src/db/schema/tasks.ts
import { pgTable, uuid, text, timestamp, pgEnum, index } from 'drizzle-orm/pg-core';
import { projects } from './projects';
import { auditFields } from './_common';

export const taskStatus = pgEnum('task_status', ['todo', 'in_progress', 'done', 'archived']);

export const tasks = pgTable('tasks', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull(),
  projectId: uuid('project_id').notNull().references(() => projects.id, { onDelete: 'cascade' }),
  title: text('title').notNull(),
  description: text('description'),
  status: taskStatus('status').notNull().default('todo'),
  dueAt: timestamp('due_at', { withTimezone: true }),
  ...auditFields,
}, (t) => ({
  byProject: index('tasks_by_project').on(t.projectId),
  byWorkspace: index('tasks_by_workspace').on(t.workspaceId),
}));
```

```ts
// src/db/schema/comments.ts
import { pgTable, uuid, text, index } from 'drizzle-orm/pg-core';
import { tasks } from './tasks';
import { auditFields } from './_common';

export const comments = pgTable('comments', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull(),
  taskId: uuid('task_id').notNull().references(() => tasks.id, { onDelete: 'cascade' }),
  body: text('body').notNull(),
  ...auditFields,
}, (t) => ({
  byTask: index('comments_by_task').on(t.taskId),
}));
```

### 3.2 Soft delete: the query convention

The *contract* is: every read filters `deleted_at IS NULL`. The *enforcement* is: a reusable helper.

```ts
// src/db/helpers.ts
import { sql, isNull } from 'drizzle-orm';

/** Append "WHERE deleted_at IS NULL" to a where builder */
export const notDeleted = (t: { deletedAt: any }) => isNull(t.deletedAt);

/** Standard `updatedAt` bump used everywhere we mutate a row */
export const touch = { updatedAt: new Date() };
```

```ts
// src/services/projectService.ts
import { eq, and } from 'drizzle-orm';
import { db } from '../db';
import { projects } from '../db/schema/projects';
import { notDeleted } from '../db/helpers';
import { HttpError } from '../lib/errors';

export async function softDeleteProject(workspaceId: string, projectId: string, userId: string) {
  const [row] = await db.update(projects)
    .set({ deletedAt: new Date(), updatedBy: userId, updatedAt: new Date() })
    .where(and(eq(projects.id, projectId), eq(projects.workspaceId, workspaceId), notDeleted(projects)))
    .returning({ id: projects.id });
  if (!row) throw new HttpError(404, 'project not found');
}
```

Hard delete is reserved for compliance erasure flows (GDPR right-to-be-forgotten), and is gated on a separate, audited, owner-only endpoint.

### 3.3 Idempotency keys

The contract: the client sends `Idempotency-Key: <uuid>` on a write request. If the server has seen the key recently with the same request fingerprint, it returns the original response without re-running the side effect.

```ts
// src/db/schema/idempotency.ts
import { pgTable, text, timestamp, jsonb, uuid } from 'drizzle-orm/pg-core';

export const idempotencyKeys = pgTable('idempotency_keys', {
  key: text('key').primaryKey(),
  workspaceId: uuid('workspace_id').notNull(),
  userId: uuid('user_id').notNull(),
  requestHash: text('request_hash').notNull(),   // sha256 of method+path+body
  response: jsonb('response'),
  statusCode: text('status_code'),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
  expiresAt: timestamp('expires_at', { withTimezone: true }).notNull(),
});
```

```ts
// src/middleware/idempotent.ts
import { createHash } from 'node:crypto';
import type { Request, Response, NextFunction } from 'express';
import { db } from '../db';
import { idempotencyKeys } from '../db/schema/idempotency';
import { eq } from 'drizzle-orm';

const TTL_MS = 1000 * 60 * 60 * 24; // 24h

export function idempotent() {
  return async (req: Request, res: Response, next: NextFunction) => {
    const key = req.header('Idempotency-Key');
    if (!key) return next();                            // optional — make it required for unsafe routes
    const fingerprint = createHash('sha256')
      .update(`${req.method}:${req.path}:${JSON.stringify(req.body)}`)
      .digest('hex');

    const existing = await db.query.idempotencyKeys.findFirst({ where: { key } } as any);
    if (existing) {
      if (existing.requestHash !== fingerprint) {
        return res.status(422).json({ error: 'Idempotency-Key reused with different payload' });
      }
      if (existing.expiresAt.getTime() > Date.now() && existing.response) {
        return res.status(Number(existing.statusCode)).json(existing.response);
      }
    }

    res.on('finish', async () => {
      if (res.statusCode >= 200 && res.statusCode < 300) {
        const body = (res as any).locals?.body ?? null;
        await db.insert(idempotencyKeys).values({
          key, workspaceId: req.workspace!.id, userId: req.user!.id,
          requestHash: fingerprint,
          response: body,
          statusCode: String(res.statusCode),
          expiresAt: new Date(Date.now() + TTL_MS),
        }).onConflictDoNothing();
      }
    });

    next();
  };
}
```

The route captures its own JSON response into `res.locals.body` before sending so the middleware can persist it.

### 3.4 File upload: Multer + Cloudinary

Two storage strategies:

- **Memory storage** for small files (≤ 8 MB). Fast, easy, no temp cleanup, but the whole file sits in RAM.
- **Disk storage** for larger files. Spill to a temp path, then stream to cloud, then `unlink`.

```ts
// src/lib/upload.ts
import multer from 'multer';
import path from 'node:path';
import os from 'node:os';

const TMP = path.join(os.tmpdir(), 'uploads');

export const memoryUpload = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: 8 * 1024 * 1024 }, // 8 MB
});

export const diskUpload = multer({
  storage: multer.diskStorage({
    destination: (_req, _file, cb) => cb(null, TMP),
    filename: (_req, file, cb) => cb(null, `${Date.now()}-${file.fieldname}${path.extname(file.originalname)}`),
  }),
  limits: { fileSize: 200 * 1024 * 1024 }, // 200 MB
});
```

The route decides which to use based on `Content-Length`:

```ts
// src/routes/files.ts
import { Router } from 'express';
import { memoryUpload, diskUpload } from '../lib/upload';
import { uploadToCloudinary } from '../services/storageService';
import { recordFile, enforceQuota } from '../services/fileService';

const router = Router({ mergeParams: true });

router.post(
  '/workspaces/:workspaceId/tasks/:taskId/files',
  (req, res, next) => {
    const handler = Number(req.header('content-length')) > 8 * 1024 * 1024 ? diskUpload.single('file') : memoryUpload.single('file');
    handler(req, res, next);
  },
  async (req, res, next) => {
    try {
      await enforceQuota(req.workspace!.id, req.file!.size);
      const stored = await uploadToCloudinary({
        buffer: req.file!.buffer,
        path: req.file!.path,
        originalName: req.file!.originalname,
        mimeType: req.file!.mimetype,
        workspaceId: req.workspace!.id,
      });
      const file = await recordFile({
        workspaceId: req.workspace!.id,
        taskId: req.params.taskId,
        uploadedBy: req.user!.id,
        stored,
        originalName: req.file!.originalname,
        mimeType: req.file!.mimetype,
        size: req.file!.size,
      });
      res.status(201).json({ file });
    } catch (e) { next(e); }
  }
);
```

### 3.5 Storage adapter (Cloudinary, with S3 shape)

We define an interface so the route handler doesn't care which provider is behind it:

```ts
// src/services/storage/types.ts
export type StoredFile = {
  provider: 'cloudinary' | 's3';
  key: string;          // provider's identifier
  url?: string;         // public url when the bucket is meant to be public
  bytes: number;
  mimeType: string;
};
```

```ts
// src/services/storage/cloudinary.ts
import { v2 as cloudinary } from 'cloudinary';
import { Readable } from 'node:stream';
import type { StoredFile } from './types';

cloudinary.config({
  cloud_name: process.env.CLOUDINARY_CLOUD_NAME!,
  api_key: process.env.CLOUDINARY_API_KEY!,
  api_secret: process.env.CLOUDINARY_API_SECRET!,
  secure: true,
});

export async function uploadToCloudinary(opts: {
  buffer?: Buffer; path?: string; originalName: string; mimeType: string; workspaceId: string;
}): Promise<StoredFile> {
  const folder = `saas/${opts.workspaceId}`;
  const resourceType = opts.mimeType.startsWith('image/') ? 'image' : 'raw';

  const upload = () => new Promise<any>((resolve, reject) => {
    const stream = cloudinary.uploader.upload_stream(
      { folder, resource_type: resourceType, type: 'authenticated', use_filename: true },
      (err, res) => err ? reject(err) : resolve(res)
    );
    if (opts.buffer) Readable.from(opts.buffer).pipe(stream);
    else require('fs').createReadStream(opts.path!).pipe(stream);
  });

  const result = await upload();
  return {
    provider: 'cloudinary',
    key: result.public_id,
    url: result.secure_url,
    bytes: result.bytes,
    mimeType: opts.mimeType,
  };
}
```

S3 looks almost identical but uses `PutObjectCommand` with a `private` ACL and a presigned `GetObjectCommand` URL on read.

### 3.6 The file table + signed URL access

```ts
// src/db/schema/files.ts
import { pgTable, uuid, text, bigint, timestamp, index } from 'drizzle-orm/pg-core';
import { tasks } from './tasks';
import { auditFields } from './_common';

export const files = pgTable('files', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull(),
  taskId: uuid('task_id').references(() => tasks.id, { onDelete: 'cascade' }),
  originalName: text('original_name').notNull(),
  mimeType: text('mime_type').notNull(),
  sizeBytes: bigint('size_bytes', { mode: 'number' }).notNull(),
  provider: text('provider').notNull(),
  providerKey: text('provider_key').notNull(),
  ...auditFields,
}, (t) => ({
  byWorkspace: index('files_by_workspace').on(t.workspaceId),
  byTask: index('files_by_task').on(t.taskId),
}));
```

Signed URL on read:

```ts
// src/services/fileService.ts
import { db } from '../db';
import { files } from '../db/schema/files';
import { and, eq, isNull, sql } from 'drizzle-orm';
import { HttpError } from '../lib/errors';
import { getSignedUrl } from './storageService';

export async function signFileDownload(workspaceId: string, fileId: string, userId: string) {
  const file = await db.query.files.findFirst({
    where: (f, { and, eq, isNull }) =>
      and(eq(f.id, fileId), eq(f.workspaceId, workspaceId), isNull(f.deletedAt)),
  });
  if (!file) throw new HttpError(404, 'file not found');

  const url = await getSignedUrl({ provider: file.provider, key: file.providerKey, ttlSeconds: 300 });
  return { url, expiresInSeconds: 300 };
}

export async function recordFile(opts: {
  workspaceId: string; taskId: string; uploadedBy: string;
  stored: { provider: string; key: string; bytes: number };
  originalName: string; mimeType: string; size: number;
}) {
  const [file] = await db.insert(files).values({
    workspaceId: opts.workspaceId,
    taskId: opts.taskId,
    uploadedBy: opts.uploadedBy,
    createdBy: opts.uploadedBy,
    provider: opts.stored.provider,
    providerKey: opts.stored.key,
    sizeBytes: opts.size,
    originalName: opts.originalName,
    mimeType: opts.mimeType,
  }).returning();
  return file;
}

/** Enforce per-workspace storage quota. */
export async function enforceQuota(workspaceId: string, additionalBytes: number) {
  const LIMITS = { maxFileSize: 200 * 1024 * 1024, totalBytes: 50 * 1024 * 1024 * 1024 }; // 50 GB
  if (additionalBytes > LIMITS.maxFileSize) throw new HttpError(413, 'file too large');

  const [{ sumBytes }] = await db.execute(sql`
    SELECT COALESCE(SUM(size_bytes), 0)::bigint AS "sumBytes"
      FROM files WHERE workspace_id = ${workspaceId} AND deleted_at IS NULL
  `);
  if (Number(sumBytes) + additionalBytes > LIMITS.totalBytes) {
    throw new HttpError(413, 'workspace storage quota exceeded');
  }
}
```

### 3.7 Deleting a file

Soft delete the row, then asynchronously destroy in cloud storage. We never hard-delete from storage immediately — that's how orphaned references in cached emails cause `404` images for users.

```ts
export async function softDeleteFile(workspaceId: string, fileId: string, userId: string) {
  const [row] = await db.update(files)
    .set({ deletedAt: new Date(), updatedBy: userId })
    .where(and(eq(files.id, fileId), eq(files.workspaceId, workspaceId)))
    .returning({ provider: files.provider, key: files.providerKey });
  if (!row) throw new HttpError(404, 'file not found');

  // Fire-and-forget; failure here shouldn't block the user.
  queueMicrotask(() => destroyInStorage(row.provider, row.key).catch((e) =>
    logger.warn({ err: e, key: row.key }, 'storage destroy failed; will retry on cron')
  ));
}
```

---

## 4. Common mistakes

1. **Storing the storage URL publicly in the database.** "It's already on a CDN!" — until someone screenshots the URL and shares it on Hacker News. Default to *authenticated* storage and serve through signed URLs.

2. **Trusting `req.file.size`.** Multer enforces it client-side too, but the source of truth is `Content-Length` and what you read from the upload result. Validate again on the server side.

3. **Forgetting `deletedAt` in *one* read path.** A single missed `isNull(deletedAt)` shows deleted files to users. Centralize the filter via `notDeleted(t)` so you can't forget it.

4. **Setting `updatedAt` to `now()` in a migration.** Migrations should never reference `now()`. Let the application set timestamps so they're consistent.

5. **Quotas that aren't transactional.** Two concurrent uploads can each pass the quota check and together bust the limit. Either use a row-level lock on the workspace or use a counter table with `SELECT ... FOR UPDATE`.

6. **Returning the entire file row including `providerKey`.** That key plus the storage credentials is enough to download the file. Strip it before sending.

---

## 5. Exercise

1. Add the `projects`, `tasks`, `comments`, and `files` tables to your schema and run the migration.
2. Implement CRUD for projects and tasks with soft delete.
3. Implement file upload via Multer → Cloudinary, with quota enforcement.
4. Implement `GET /files/:id/url` that returns a 5-minute signed URL.
5. Add an idempotency-key middleware to the `POST /tasks` route and prove it works (curl twice with the same key, get one row).
6. **Stretch:** Add a "trash" endpoint that lists all `deletedAt IS NOT NULL` rows for the current workspace, and a "restore" endpoint that sets `deletedAt = NULL`.

**Acceptance criteria:**
- Soft-deleting a project hides them from list endpoints but keeps the row.
- A user can't upload a 250 MB file (rejected by quota).
- A signed URL expires after 5 minutes; reusing it returns 403.
- The same idempotency key on two requests returns the *same* response body.

---

## 6. AI Industry Note (2026)

In 2026, AI code review routinely catches the "missing `isNull(deletedAt)`" class of bug by comparing every query against the project's tenant-table registry. The pattern: a meta-tool indexes every Drizzle table and flags reads that don't reference the `deletedAt` column.

The lesson: the patterns in this module — soft delete, audit fields, idempotency keys — are exactly the *boilerplate that is safe to delegate*. The hard part isn't writing them; it's being consistent. So centralize them: a `softDelete()` service, an `auditFields` mixin, an `idempotent()` middleware. Once they're a function call, an agent will use them correctly. If they're something an agent has to remember to type, the agent — and your future tired self at 11pm — will eventually forget.