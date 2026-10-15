# Module 05 — Node.js Runtime & Architecture

> How the web actually works, and how Node turns JavaScript from "a browser language" into "the language that runs the back end of the internet."

---

## Why this module matters

Most TypeScript developers write backend code without understanding what happens between `req` and `res`. The moment something goes wrong in production — memory leak, event loop blocked, dropped connections — that lack of understanding becomes a 3am page.

By the end of this module, you will be able to:

- Draw the request/response cycle from socket to handler.
- Explain the Node event loop, microtasks, and the `libuv` thread pool.
- Know when code is *truly* parallel in Node (almost never by default).
- Choose between CommonJS and ESM in a TypeScript project.
- Structure a `package.json` like a senior.

---

## 1. How the web works

```
Browser                    Server
  │ ── TCP SYN ──────────► │   (kernel accepts socket)
  │ ── HTTP request ─────► │   (Node http.Server emits 'request')
  │                       │ ─► your handler runs
  │ ◄── HTTP response ─── │   (you call res.end())
  │ ◄── TCP FIN ──────────│   (connection closes or kept-alive)
```

**Key facts:**

- HTTP is **a text protocol over TCP**.
- A request is a method, a path, headers, and (optionally) a body.
- A response is a status code, headers, and a body.
- **Keep-alive** reuses the TCP connection for multiple requests — that's why Node's default HTTP server is fast.

---

## 2. Understanding the Node.js runtime

Node is **not** a JavaScript engine. It is a JavaScript **runtime** that bundles:

| Component | Role |
|-----------|------|
| **V8** | Google's JS engine — compiles JS to machine code |
| **libuv** | A C library that provides the event loop and a thread pool |
| **Node core (C++)** | File system, networking, crypto, etc. |
| **Node API (JS)** | `fs`, `http`, `path`, `process`, etc. — the parts you `import` |

When you `import { readFile } from "node:fs"`, you are calling into C++ code that eventually asks libuv to do I/O on a worker thread.

---

## 3. Event loop — non-blocking architecture

Node is **single-threaded for JavaScript execution** but uses libuv's thread pool for I/O.

```
   ┌───────────────────────────┐
┌─►│        timers             │   setTimeout / setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │  pending callbacks        │   some OS-level callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │  idle, prepare            │   internal
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │  poll                     │   retrieve new I/O events
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │  check                    │   setImmediate
│  └─────────────┬─────────────┘
│  └─────────────┘
```

Between every phase, the runtime **drains the microtask queue** (Promise callbacks, `process.nextTick`).

### Sync vs async

```ts
// Synchronous — blocks the event loop
import { readFileSync } from "node:fs";
const data = readFileSync("/etc/hosts");

// Asynchronous — non-blocking
import { readFile } from "node:fs/promises";
const data = await readFile("/etc/hosts");
```

**Mental model:** Node hands I/O work to libuv's thread pool (default 4 threads). Your JS keeps running. When the I/O is done, a callback is queued. The event loop picks it up at the next phase.

### Microtasks vs macrotasks

```ts
console.log("1");
setTimeout(() => console.log("2"), 0);
Promise.resolve().then(() => console.log("3"));
console.log("4");

// Output: 1, 4, 3, 2
// Why: 3 is a microtask (drained before the next event-loop phase)
//      2 is a macrotask (waits for the timers phase)
```

This is the source of 90% of "why is my code running in the wrong order" questions.

---

## 4. Callbacks and callback hell

The old way:

```ts
import { readFile } from "node:fs";

readFile("a.txt", "utf8", (err, a) => {
  if (err) throw err;
  readFile("b.txt", "utf8", (err, b) => {
    if (err) throw err;
    readFile("c.txt", "utf8", (err, c) => {
      if (err) throw err;
      console.log(a + b + c);
    });
  });
});
```

The modern way:

```ts
import { readFile } from "node:fs/promises";
const a = await readFile("a.txt", "utf8");
const b = await readFile("b.txt", "utf8");
const c = await readFile("c.txt", "utf8");
console.log(a + b + c);
```

**Rule of thumb in 2026:** use `fs/promises` everywhere. Treat `fs` callback APIs as legacy.

---

## 5. Promises and async/await

A `Promise` is a *placeholder* for a value that will arrive later.

```ts
function fetchUser(id: string): Promise<User> {
  return db.query("SELECT * FROM users WHERE id = $1", [id])
    .then((rows) => rows[0])
    .then((row) => ({
      id: row.id,
      email: row.email,
      // ...
    }));
}
```

With `async/await` (preferred):

```ts
async function fetchUser(id: string): Promise<User> {
  const rows = await db.query("SELECT * FROM users WHERE id = $1", [id]);
  const row = rows[0];
  if (!row) throw new UserNotFoundError(id);
  return { id: row.id, email: row.email };
}
```

### Error handling

```ts
try {
  const u = await fetchUser("missing");
} catch (err) {
  if (err instanceof UserNotFoundError) {
    return null;
  }
  throw err;   // unknown — rethrow
}
```

`async` functions automatically wrap thrown values in a rejected Promise.

### `Promise.all` / `Promise.allSettled` / `Promise.race`

```ts
// Run in parallel, fail fast
const [a, b, c] = await Promise.all([fetchA(), fetchB(), fetchC()]);

// Run in parallel, never fail
const results = await Promise.allSettled([fetchA(), fetchB(), fetchC()]);
results.forEach((r) => r.status === "fulfilled" ? use(r.value) : log(r.reason));
```

Use `Promise.allSettled` whenever you want to *partially* succeed (e.g. a batch import).

---

## 6. CommonJS vs ES Modules

| | CommonJS | ESM |
|--|---------|-----|
| Syntax | `require()` / `module.exports` | `import` / `export` |
| Resolution | synchronous | asynchronous, static |
| Top-level `await` | ❌ | ✅ |
| Tree-shaking | poor | excellent |
| File extension | `.cjs` or implicit | `.mjs` or implicit via `"type": "module"` |

**Recommendation in 2026:** use **ESM** for everything new. Configure `"type": "module"` in `package.json` and use `.js` (or `.ts` via `tsx`).

```json
// package.json
{
  "name": "my-backend",
  "type": "module",
  "engines": { "node": ">=22" }
}
```

```ts
// src/index.ts
import { createServer } from "node:http";
export const server = createServer(handler);
```

---

## 7. npm ecosystem and package management

### `package.json` — the parts that matter

```json
{
  "name": "my-api",
  "version": "0.1.0",
  "type": "module",
  "engines": { "node": ">=22" },
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc -p tsconfig.json",
    "start": "node dist/index.js",
    "test": "vitest run",
    "lint": "eslint .",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {},
  "devDependencies": {}
}
```

### `dependencies` vs `devDependencies`

- **`dependencies`** — packages your **runtime** needs (express, drizzle-orm, openai).
- **`devDependencies`** — packages only needed for **build/test/lint** (typescript, vitest, eslint).

In production, `node_modules` will only contain `dependencies` (if you use `pnpm deploy` or Docker with `pnpm install --prod`).

### Why pnpm over npm

- **Disk-efficient** — uses a content-addressable store, shared across projects.
- **Strict** — won't let you import a package you didn't declare as a dependency.
- **Fast** — ~2x faster installs than npm.

```bash
npm i -g pnpm
pnpm init
pnpm add express
pnpm add -D typescript @types/node tsx vitest
```

### `package.json` scripts — the convention

| Script | Purpose |
|--------|---------|
| `dev` | Start with hot reload (`tsx watch` or `nodemon`) |
| `build` | Compile TypeScript to `dist/` |
| `start` | Run compiled code in production |
| `test` | Run all tests once |
| `test:watch` | Run tests in watch mode |
| `lint` | Lint with ESLint |
| `format` | Format with Prettier |
| `typecheck` | `tsc --noEmit` |
| `db:migrate` | Run Drizzle migrations |
| `db:seed` | Seed the database |

When you apply for a job, a clean `scripts` block is a small but real signal of senior hygiene.

---

## Exercise

1. Draw the event loop and explain why `setTimeout(fn, 0)` runs *after* a resolved Promise's `.then`.
2. Convert the callback-hell example to `async/await` and `Promise.all` where appropriate.
3. Bootstrap a new project with `pnpm init`, set up `type: "module"`, add `tsx` and `typescript`, and write a script that reads three files in parallel with `Promise.all`.
4. Explain in your own words why `fs/promises` is preferred over the callback `fs` API in 2026.
5. Add a `pretest` script that runs `tsc --noEmit` — break your types on purpose and watch it fail.

---

## Common mistakes

1. **CPU-heavy work on the event loop.** A 500ms JSON.stringify of a giant array will block every other request. Use `worker_threads` or a queue.
2. **Forgetting `await`.** A `Promise` is not a value. Logging it shows `Promise { <pending> }`.
3. **`await` inside `.map()`.** This serializes what should be parallel. Use `Promise.all(items.map(async ...))`.
4. **Catching everything.** A bare `catch {}` swallows real bugs. Catch specific errors.

---

## AI Industry Note (2026)

- **Node's async model is its superpower for AI workloads.** LLM API calls are network I/O. A single Node process can hold open hundreds of in-flight requests, each awaiting a streaming LLM response, while the event loop stays responsive. This is why every AI product you use in 2026 is almost certainly Node, Bun, or Deno on the server.
- **The "I don't need to understand the event loop because async/await hides it" myth.** It hides it well — until you hit a memory leak, a `UnhandledPromiseRejection`, or a CPU spike. The senior who can read `clinic flame` and find the blocking call is the one who gets the production on-call rotation.
- **What to learn next:** read the [Node.js docs on the event loop](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick) once a quarter. Every time, you notice something new.
