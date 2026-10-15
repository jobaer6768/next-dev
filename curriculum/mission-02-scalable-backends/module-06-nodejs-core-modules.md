# Module 06 — Node.js Core Modules & Your First Server

> The native APIs every backend engineer must know — and why you should keep using them even after Express arrives.

---

## Why this module matters

Express, Fastify, Nest — they're all nice. But underneath each of them is a hand-built `http.createServer`. The moment you need to debug a streaming upload, attach a WebSocket, or build a custom proxy, you'll be glad you know what `node:http`, `node:fs`, `node:path`, and `node:stream` actually do.

This module also teaches you **how to feel pain** — by building a server *without* a framework. That pain is what makes the framework ergonomics meaningful in Module 07.

---

## Learning objectives

- Use `fs`, `path`, `http`, `process`, `os`, `crypto`, and `stream` confidently.
- Read and write files both synchronously and asynchronously.
- Understand Buffers and Streams at a practical level.
- Build a complete HTTP server **without** Express.
- Set up `dotenv` for environment variables.
- Configure `tsx` / `nodemon` for development.
- Connect a TypeScript project end to end.

---

## 1. The core modules — the ones you'll touch

| Module | Purpose |
|--------|---------|
| `node:fs` / `node:fs/promises` | File system |
| `node:path` | Cross-platform path manipulation |
| `node:http` / `node:https` | HTTP server and client |
| `node:url` | URL parsing |
| `node:querystring` | Query string utilities |
| `node:os` | OS info (CPU, memory, hostname) |
| `node:crypto` | Hashing, HMAC, random bytes, ciphers |
| `node:stream` | Streaming abstraction |
| `node:buffer` | Binary data containers |
| `node:process` | The current process and env |
| `node:events` | EventEmitter base class |
| `node:zlib` | Gzip / deflate |
| `node:child_process` | Spawn sub-processes |
| `node:worker_threads` | CPU parallelism |

Always prefer the `node:` prefix (`import { readFile } from "node:fs/promises"`). It makes the source of the module explicit.

---

## 2. File system — sync vs async

```ts
import { readFile, writeFile, appendFile, unlink, mkdir, readdir, stat } from "node:fs/promises";
import { createReadStream, createWriteStream } from "node:fs";

const text = await readFile("./README.md", "utf8");
await writeFile("./out.txt", "hello", "utf8");

// Append-mode logging
await appendFile("./app.log", `${new Date().toISOString()} ready\n`);

// Recursive directory creation
await mkdir("./uploads/2026/01", { recursive: true });

// List
const entries = await readdir("./src", { withFileTypes: true });
for (const e of entries) {
  console.log(e.name, e.isDirectory() ? "(dir)" : "(file)");
}

// Stat
const s = await stat("./README.md");
console.log(s.size, "bytes");
```

**Decision rule:**

- **Async / `fs/promises`** — 99% of cases. Server code must not block.
- **Sync (`fs.readFileSync`)** — only in startup scripts, CLI tools, and one-off bootstrap.

---

## 3. Buffers — the binary building block

A `Buffer` is a chunk of raw bytes in memory.

```ts
const buf = Buffer.from("hello", "utf8");
console.log(buf);              // <Buffer 68 65 6c 6c 6f>
console.log(buf.toString());   // "hello"

const hex = Buffer.from("hello").toString("hex"); // "68656c6c6f"
```

You'll meet Buffers in:

- File I/O
- HTTP request/response bodies
- Network sockets
- Crypto (hashing)

### Alloc vs from

```ts
const empty = Buffer.alloc(1024);          // 1KB of zeros, safe
const dangerous = Buffer.allocUnsafe(1024); // 1KB uninitialized, may contain old data
```

**Never** use `allocUnsafe` unless you have measured the perf gain and immediately fill the buffer.

---

## 4. Streams — the secret to Node's speed

Streams let you process data **as it arrives**, without loading it all into memory.

```ts
import { createReadStream, createWriteStream } from "node:fs";
import { pipeline } from "node:stream/promises";
import { createGzip } from "node:zlib";

await pipeline(
  createReadStream("./big.log"),
  createGzip(),
  createWriteStream("./big.log.gz"),
);
```

This `pipeline` reads, gzips, and writes — for a 10GB log file, using only ~64KB of RAM at any moment. The **same** code that does it in 10MB of RAM also does it in 10GB.

### The four stream types

| Type | Examples | Reads from / writes to |
|------|----------|------------------------|
| Readable | `fs.createReadStream`, `http.IncomingMessage` | source |
| Writable | `fs.createWriteStream`, `http.ServerResponse` | sink |
| Transform | `zlib.createGzip`, `crypto.createCipheriv` | both |
| Duplex | `net.Socket`, `tls.TLSSocket` | both, independent |

`pipeline` handles backpressure, errors, and cleanup. **Use it.** Don't hand-roll `.on("data")` chains.

### Backpressure in 30 seconds

When the source is faster than the destination, data piles up. Node's streams emit a `'drain'` event to tell you "OK, send more." `pipeline` handles this for you. Hand-rolled streams will silently OOM your process.

---

## 5. `process` — the current Node process

```ts
process.env.NODE_ENV        // "development" | "production" | "test"
process.env.PORT            // string, e.g. "3000"
process.argv                // [node, script.js, ...args]
process.cwd()               // current working directory
process.pid                 // OS process id
process.uptime()            // seconds
process.exit(0)             // 0 = success, 1+ = error
process.on("uncaughtException", err => { /* log + exit */ });
process.on("unhandledRejection", err => { /* log + exit */ });
```

### Reading environment variables

```ts
import { z } from "zod";

const Env = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  PORT: z.coerce.number().int().positive().default(3000),
  DATABASE_URL: z.string().url(),
});

export const env = Env.parse(process.env);
```

In 2026, **always** validate `process.env` at boot. The cost of one bug from `undefined` slipping through is higher than the cost of writing 5 lines of Zod.

### Graceful shutdown

```ts
async function shutdown(signal: string) {
  console.log(`Received ${signal}, closing...`);
  server.close();
  await db.end();
  process.exit(0);
}

process.on("SIGINT",  () => shutdown("SIGINT"));
process.on("SIGTERM", () => shutdown("SIGTERM"));
```

Docker, Kubernetes, and your future on-call rotation will thank you.

---

## 6. `path` — cross-platform paths

```ts
import path from "node:path";

path.join("src", "modules", "user.ts");        // "src/modules/user.ts"
path.resolve("/var", "log", "app.log");         // "/var/log/app.log"
path.basename("/var/log/app.log");              // "app.log"
path.dirname("/var/log/app.log");               // "/var/log"
path.extname("/var/log/app.log");               // ".log"

const root = path.resolve(process.cwd());
```

**Rule:** never concatenate paths with `+`. Always `path.join` or `path.resolve`.

---

## 7. `crypto` — the modules you'll use

```ts
import { randomBytes, createHash, createHmac, randomUUID, timingSafeEqual } from "node:crypto";

// Secure random
const id = randomUUID();
const token = randomBytes(32).toString("hex");

// Hashing
const hash = createHash("sha256").update("password").digest("hex");

// HMAC (for signing JWTs, webhooks, etc.)
const sig = createHmac("sha256", secret).update(body).digest("hex");

// Constant-time comparison (prevents timing attacks)
const a = Buffer.from(hash);
const b = Buffer.from(expected);
const ok = timingSafeEqual(a, b);
```

In Mission 4 you will use `crypto.randomUUID` for IDs and `bcrypt` for password hashing.

---

## 8. Building your first HTTP server (no framework)

```ts
// src/server.ts
import { createServer, IncomingMessage, ServerResponse } from "node:http";
import { readFile, writeFile } from "node:fs/promises";
import { randomUUID } from "node:crypto";

type Todo = { id: string; title: string; done: boolean };

let todos: Todo[] = [];

async function readJson<T>(req: IncomingMessage): Promise<T> {
  const chunks: Buffer[] = [];
  for await (const chunk of req) chunks.push(chunk as Buffer);
  const raw = Buffer.concat(chunks).toString("utf8");
  return raw ? (JSON.parse(raw) as T) : ({} as T);
}

function send(res: ServerResponse, status: number, body: unknown) {
  res.statusCode = status;
  res.setHeader("content-type", "application/json; charset=utf-8");
  res.end(JSON.stringify(body));
}

const server = createServer(async (req, res) => {
  const url = new URL(req.url ?? "/", `http://${req.headers.host}`);
  const { pathname, searchParams } = url;
  const method = req.method ?? "GET";

  try {
    if (method === "GET" && pathname === "/todos") {
      return send(res, 200, todos);
    }

    if (method === "POST" && pathname === "/todos") {
      const body = await readJson<{ title?: string }>(req);
      if (!body.title) return send(res, 400, { error: "title required" });
      const todo: Todo = { id: randomUUID(), title: body.title, done: false };
      todos.push(todo);
      return send(res, 201, todo);
    }

    if (method === "DELETE" && pathname.startsWith("/todos/")) {
      const id = pathname.split("/")[2];
      const before = todos.length;
      todos = todos.filter((t) => t.id !== id);
      if (todos.length === before) return send(res, 404, { error: "not found" });
      return send(res, 204, null);
    }

    send(res, 404, { error: "not found" });
  } catch (err) {
    console.error(err);
    send(res, 500, { error: "internal" });
  }
});

const PORT = Number(process.env.PORT ?? 3000);
server.listen(PORT, () => console.log(`Listening on http://localhost:${PORT}`));
```

### What this teaches you (and why the pain is the point)

- **Routing by hand is painful.** The `if/else` chain will not scale. This is why Express exists.
- **Body parsing is a chore.** Every endpoint has to `await readJson(req)`. The pattern gets old fast.
- **Status codes and headers** are *your* responsibility. Frameworks automate this.
- **CORS, compression, request logging** are missing. Frameworks provide them as middleware.

You now understand *why* Express and Fastify exist. You will appreciate them in Module 07, and you will be able to debug them when they break.

---

## 9. Environment variables with `dotenv`

```bash
pnpm add dotenv
```

```ts
// src/config.ts
import "dotenv/config";   // loads .env into process.env

export const config = {
  port: Number(process.env.PORT ?? 3000),
  nodeEnv: process.env.NODE_ENV ?? "development",
  dbUrl: process.env.DATABASE_URL ?? "postgres://localhost:5432/dev",
} as const;
```

**`.env` files:** add `.env` to `.gitignore`. Commit `.env.example` with safe placeholders.

---

## 10. `nodemon` / `tsx` for dev

```bash
pnpm add -D tsx
```

```json
"scripts": {
  "dev": "tsx watch src/server.ts"
}
```

`tsx` runs TypeScript directly (esbuild-powered) and restarts on file changes. **Use this in 2026, not `ts-node`.** It is 50x faster and supports ESM out of the box.

---

## 11. Debuggable structure

```
project/
├── src/
│   ├── server.ts          # entry, http.createServer
│   ├── config.ts          # env loading
│   ├── routes/
│   │   └── todos.ts
│   ├── lib/
│   │   ├── http.ts        # send(), readJson()
│   │   └── errors.ts
│   └── data/
│       └── todos.ts       # in-memory store
├── package.json
├── tsconfig.json
├── .env.example
├── .gitignore
└── README.md
```

This is roughly the layout we will evolve into a layered Express app in Module 09.

---

## Exercise

1. Refactor the todo server above to use **streams** for body parsing (use `Readable.toArray()` or hand-rolled buffer concat).
2. Add a `GET /todos/:id` route that returns 404 when missing.
3. Add a graceful-shutdown handler for `SIGINT` and `SIGTERM`.
4. Add a `crypto`-backed request-id middleware that sets `X-Request-Id` on every response.
5. Add a `?q=search` query-string filter to `GET /todos`.
6. Validate the request body with **Zod** — return 400 on bad input.

Run with `pnpm dev`. Test with `curl` or **Bruno** (we will use Bruno for all API testing in this course).

---

## Common mistakes

1. **Reading the body twice.** `req` is a stream; once consumed, it's gone.
2. **Forgetting `await` on `readJson`.** The body hasn't arrived yet.
3. **Not handling `JSON.parse` errors.** Wrap it in try/catch and return 400.
4. **In-memory `let todos = []` survives until process restart only.** In Mission 2 you will replace it with a real database.
5. **Mixing `path` strings with `/` and `\\` on Windows.** Always `path.join`.

---

## AI Industry Note (2026)

- **The "raw Node first" tradition is timeless.** Every framework's docs eventually say *"and here's how to do it with the raw `http` module."* Knowing the raw version means you can debug the framework. Express hides the body parsing; one day you'll need to know it's just a stream read.
- **Streams are the secret of LLM integration.** When you call `openai.chat.completions.create({ stream: true })`, the response is a Node `Readable`. The first time you try to wrap that in a framework that doesn't understand streams, you'll wish you'd internalized `pipeline` here.
- **What to read next:** [Node's official stream guide](https://nodejs.org/en/learn/asynchronous-work/streams) — specifically the section on backpressure. It's short, and it's the difference between code that scales and code that crashes at 1000 RPS.
