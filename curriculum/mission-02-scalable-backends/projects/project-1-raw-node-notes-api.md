# Project 1 — Raw Node.js Notes / Todo API

> A complete API built on `node:http` only. **No Express, no framework, no excuses.**

---

## Goal

Build a working notes / todo API using only Node's built-in modules. By the end you will have proven you understand the layers Express is hiding from you.

---

## Features

- `GET /notes` — list all notes (with optional `?q=search`)
- `POST /notes` — `{ title, body }` → create
- `GET /notes/:id` — read one
- `PATCH /notes/:id` — partial update
- `DELETE /notes/:id` — remove

---

## Suggested file layout

```
nl-m2-p1-raw-node/
├── src/
│   ├── server.ts        # createServer, routing by hand
│   ├── lib/
│   │   ├── http.ts      # sendJson, readJson, parseUrl
│   │   ├── store.ts     # JSON file persistence with lock
│   │   ├── logger.ts
│   │   └── errors.ts
│   └── types.ts
├── data/                # auto-created, notes.json lives here
├── package.json
├── tsconfig.json
└── README.md
```

---

## The store — JSON file with a simple lock

```ts
// src/lib/store.ts
import { readFile, writeFile, mkdir } from "node:fs/promises";
import { existsSync } from "node:fs";
import path from "node:path";
import { randomUUID } from "node:crypto";

const DATA_DIR = path.resolve(process.cwd(), "data");
const FILE = path.join(DATA_DIR, "notes.json");

export type Note = { id: string; title: string; body: string; createdAt: string; updatedAt: string };

let writeChain: Promise<void> = Promise.resolve();

async function ensureFile() {
  if (!existsSync(DATA_DIR)) await mkdir(DATA_DIR, { recursive: true });
  if (!existsSync(FILE)) await writeFile(FILE, "[]", "utf8");
}

async function readAll(): Promise<Note[]> {
  await ensureFile();
  const text = await readFile(FILE, "utf8");
  return JSON.parse(text) as Note[];
}

async function writeAll(notes: Note[]): Promise<void> {
  await ensureFile();
  await writeFile(FILE, JSON.stringify(notes, null, 2), "utf8");
}

export const store = {
  async list(q?: string): Promise<Note[]> {
    const all = await readAll();
    if (!q) return all;
    const needle = q.toLowerCase();
    return all.filter((n) => n.title.toLowerCase().includes(needle) || n.body.toLowerCase().includes(needle));
  },
  async findById(id: string): Promise<Note | null> {
    const all = await readAll();
    return all.find((n) => n.id === id) ?? null;
  },
  async create(input: { title: string; body: string }): Promise<Note> {
    const note: Note = {
      id: randomUUID(),
      title: input.title,
      body: input.body,
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
    };
    const next = writeChain.then(async () => {
      const all = await readAll();
      all.push(note);
      await writeAll(all);
    });
    writeChain = next.catch(() => undefined);
    await next;
    return note;
  },
  async update(id: string, patch: Partial<{ title: string; body: string }>): Promise<Note | null> {
    let updated: Note | null = null;
    const next = writeChain.then(async () => {
      const all = await readAll();
      const i = all.findIndex((n) => n.id === id);
      if (i === -1) return;
      all[i] = { ...all[i]!, ...patch, id, updatedAt: new Date().toISOString() };
      updated = all[i]!;
      await writeAll(all);
    });
    writeChain = next.catch(() => undefined);
    await next;
    return updated;
  },
  async delete(id: string): Promise<boolean> {
    let ok = false;
    const next = writeChain.then(async () => {
      const all = await readAll();
      const next = all.filter((n) => n.id !== id);
      if (next.length === all.length) return;
      ok = true;
      await writeAll(next);
    });
    writeChain = next.catch(() => undefined);
    await next;
    return ok;
  },
};
```

`writeChain` serializes writes. In a single-process app this prevents the read-modify-write race. (For multi-process you'd use a real file lock like `proper-lockfile`.)

---

## The HTTP helpers

```ts
// src/lib/http.ts
import type { IncomingMessage, ServerResponse } from "node:http";

export async function readJson<T>(req: IncomingMessage): Promise<T> {
  const chunks: Buffer[] = [];
  for await (const c of req) chunks.push(c as Buffer);
  const text = Buffer.concat(chunks).toString("utf8");
  if (!text) throw Object.assign(new Error("empty body"), { status: 400 });
  try { return JSON.parse(text) as T; }
  catch { throw Object.assign(new Error("invalid JSON"), { status: 400 }); }
}

export function sendJson(res: ServerResponse, status: number, body: unknown) {
  res.statusCode = status;
  res.setHeader("content-type", "application/json; charset=utf-8");
  res.end(JSON.stringify(body));
}

export function sendError(res: ServerResponse, status: number, message: string) {
  sendJson(res, status, { error: { code: status, message } });
}
```

---

## The server

```ts
// src/server.ts
import { createServer } from "node:http";
import { randomUUID } from "node:crypto";
import { store } from "./lib/store.js";
import { readJson, sendJson, sendError } from "./lib/http.js";

const server = createServer(async (req, res) => {
  const requestId = (req.headers["x-request-id"] as string) ?? randomUUID();
  res.setHeader("x-request-id", requestId);
  const start = Date.now();
  const url = new URL(req.url ?? "/", `http://${req.headers.host ?? "localhost"}`);
  const method = req.method ?? "GET";

  try {
    if (method === "GET" && url.pathname === "/notes") {
      const items = await store.list(url.searchParams.get("q") ?? undefined);
      return sendJson(res, 200, { data: items });
    }
    if (method === "POST" && url.pathname === "/notes") {
      const body = await readJson<{ title?: string; body?: string }>(req);
      if (!body.title) return sendError(res, 400, "title required");
      const note = await store.create({ title: body.title, body: body.body ?? "" });
      return sendJson(res, 201, { data: note });
    }

    const idMatch = url.pathname.match(/^\/notes\/([\w-]+)$/);
    if (idMatch) {
      const id = idMatch[1]!;
      if (method === "GET") {
        const n = await store.findById(id);
        return n ? sendJson(res, 200, { data: n }) : sendError(res, 404, "not found");
      }
      if (method === "PATCH") {
        const body = await readJson<{ title?: string; body?: string }>(req);
        const n = await store.update(id, body);
        return n ? sendJson(res, 200, { data: n }) : sendError(res, 404, "not found");
      }
      if (method === "DELETE") {
        const ok = await store.delete(id);
        return ok ? sendJson(res, 204, null) : sendError(res, 404, "not found");
      }
    }

    sendError(res, 404, "not found");
  } catch (err: unknown) {
    const e = err as { status?: number; message?: string };
    const status = e.status ?? 500;
    const message = e.message ?? "internal";
    if (status >= 500) console.error({ err, requestId }, "unhandled");
    sendError(res, status, message);
  } finally {
    console.log(`[${requestId}] ${method} ${url.pathname} ${res.statusCode} ${Date.now() - start}ms`);
  }
});

const PORT = Number(process.env.PORT ?? 3000);
server.listen(PORT, () => console.log(`Notes API on http://localhost:${PORT}`));

function shutdown(signal: string) {
  console.log(`Got ${signal}, closing`);
  server.close(() => process.exit(0));
  setTimeout(() => process.exit(1), 5000).unref();
}
process.on("SIGINT",  () => shutdown("SIGINT"));
process.on("SIGTERM", () => shutdown("SIGTERM"));
```

---

## README template

```md
# Notes API (raw Node)

A notes API built on `node:http` only.

## Run
\`\`\`
pnpm install
pnpm dev
\`\`\`

## Endpoints
- GET /notes?q=...
- POST /notes  { title, body }
- GET /notes/:id
- PATCH /notes/:id { title?, body? }
- DELETE /notes/:id

## Storage
A single JSON file at `data/notes.json`.

## Design notes
- Manual body parsing (no framework).
- Manual routing.
- Single-process write lock to prevent races.
- Graceful shutdown on SIGINT/SIGTERM.
```

---

## Acceptance criteria

- All five endpoints work.
- `data/notes.json` is created on first write.
- Survives a process restart.
- A `DELETE` of a non-existent id returns 404.
- Search is case-insensitive.
- Every response has an `X-Request-Id` header.
- README has run + test instructions.

When you finish, move on to **Project 2 — Express + PostgreSQL Mini API** (see `assignment-02.md`).
