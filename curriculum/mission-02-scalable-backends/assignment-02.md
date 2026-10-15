# Assignment 2 — Small Backend Project: Layered, Modular, Validated

> Two small backend projects anchor this assignment: **Project 1 — Raw Node.js Notes API** (after Module 06) and **Project 2 — Express + PostgreSQL Mini API** (after Module 09).

---

## Why this assignment matters

You have learned how to *use* each tool — Node, Express, pg, Zod, layered architecture. Now you must prove you can **combine them** into a real, shippable API.

The grader looks for:

- Does the layered structure hold? (Route → Controller → Service → Repository)
- Is every request validated?
- Are errors centralized and consistent?
- Is the README so clear a junior could `git clone` and run it?

---

## Project 1 — Raw Node Notes / Todo API

**Stack:** `node:http`, `node:fs`, `node:path`, `node:crypto` — *no* framework, *no* Express.

**Features:**

- `GET /notes` — list all notes
- `POST /notes` — create a note `{ title, body }`
- `GET /notes/:id` — read
- `PATCH /notes/:id` — update partial
- `DELETE /notes/:id` — delete
- `GET /notes/search?q=...` — case-insensitive title search

**Storage:** `./data/notes.json` (a single JSON file). Read on startup, write on every mutation (with file lock to prevent corruption).

**Required:**

- Request IDs (use `randomUUID`).
- Request logging (method, path, status, duration) — plain `console.log` is fine.
- Manual body parsing (you already wrote this in Module 06).
- Manual routing (no Express).
- Consistent `{ data, error }` response shape.
- Status codes: 200, 201, 204, 400, 404, 500.
- Graceful shutdown on `SIGINT` and `SIGTERM`.
- README explaining how to run and test.

**Anti-requirements:**

- Do NOT add Express.
- Do NOT use a database.

---

## Project 2 — Express + PostgreSQL Layered Mini API

**Stack:** Express, TypeScript, `pg`, Zod.

**Pick ONE of the two domains:**

### Option A: Expense Tracker

- Tables: `categories`, `expenses`.
- `expenses` has `amount NUMERIC(10,2)`, `category_id`, `note`, `spent_at DATE`.
- Endpoints:
  - `GET /api/v1/categories` (list)
  - `POST /api/v1/categories` (create)
  - `GET /api/v1/expenses?from=...&to=...&category=...&page=1&limit=20`
  - `POST /api/v1/expenses` (create)
  - `PATCH /api/v1/expenses/:id`
  - `DELETE /api/v1/expenses/:id`
  - `GET /api/v1/expenses/summary?month=2026-01` — group by category with totals

### Option B: Book Catalogue

- Tables: `authors`, `books`.
- `books` has `title`, `author_id`, `isbn UNIQUE`, `published_year`, `genre`.
- Endpoints:
  - `GET /api/v1/authors` (search by name)
  - `POST /api/v1/authors`
  - `GET /api/v1/books?author=...&genre=...&q=...&page=...`
  - `POST /api/v1/books`
  - `PATCH /api/v1/books/:id`
  - `DELETE /api/v1/books/:id`
  - `GET /api/v1/books/:id/author` — joined author

**Required structure:**

```
src/
├── modules/
│   ├── expenses/        (or books/)
│   │   ├── *.controller.ts
│   │   ├── *.service.ts
│   │   ├── *.repository.ts
│   │   ├── *.routes.ts
│   │   ├── *.schema.ts
│   │   ├── *.types.ts
│   │   └── *.test.ts
├── lib/
├── db/
├── config/
├── app.ts
├── server.ts
└── migrations/
```

**Required functionality:**

- Full CRUD with Zod validation on every body and query.
- Centralized error handling with `HttpError` subclasses.
- `pino` logger.
- Pagination on every list endpoint.
- A search endpoint (case-insensitive, partial match).
- A summary endpoint (uses `GROUP BY`).
- Migrations in plain SQL, run by a script `pnpm db:migrate`.
- `docs/API.md` for every endpoint.
- A Bruno collection under `bruno/`.
- Vitest unit tests for the service layer.
- A README with run instructions.

---

## Grading rubric (out of 100)

| Area | Points |
|------|--------|
| Project 1 — works without Express, persists to JSON, has graceful shutdown | 15 |
| Project 2 — layered architecture is clean (route → controller → service → repo) | 20 |
| Project 2 — Zod validation on every endpoint | 10 |
| Project 2 — centralized error handling | 10 |
| Project 2 — pagination + search + summary endpoint work | 10 |
| Project 2 — migrations runnable via `pnpm db:migrate` | 5 |
| Project 2 — README + Bruno collection + API docs | 10 |
| Project 2 — unit tests for the service | 10 |
| Code review rubric: no `any`, no dead code, no comments-that-lie | 10 |

---

## Submission

- Two repos: `nl-m2-p1-raw-node` and `nl-m2-p2-express-pg`.
- Each must `pnpm install && pnpm dev` (or `pnpm start`) on a fresh clone with Postgres available.
- Open a PR per project. Title `[NL-M2-P1] <name>` / `[NL-M2-P2] <name>`.
- **Deadline:** 7 days from the start of Mission 2.

---

## AI Industry Note (2026)

- **The raw-Node project is your debugging superpower.** When the framework breaks (and it will, at 3am), the engineer who can read a raw `http` request and know exactly which middleware dropped the body is the one who fixes the outage. Most of your peers will not have this skill.
- **The layered project is your portfolio centerpiece for backend roles.** Hiring managers in 2026 *immediately* recognize route/controller/service/repo. Show it once, clearly, and you have signaled senior thinking. Show it twice across two different domains, and you have demonstrated *transferable* senior thinking.
- **What to learn next:** read the [Express docs on error handling](https://expressjs.com/en/guide/error-handling.html) and the [`pg` docs on connection pooling](https://node-postgres.com/features/pooling). Both are short. Both will make you a better engineer this week.
