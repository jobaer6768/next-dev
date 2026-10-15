# Assignment 1 — Type-Safe Utilities + Small OOP Domain

> Two conceptual sessions anchor this assignment: **"How the JS runtime actually runs your code"** and **"Thinking in types: reading and designing type signatures."**

---

## Why this assignment matters

This is your **first portfolio piece**. After Module 01–03 you have the type system and the OOP toolkit. Now you will prove — to yourself and to a future employer — that you can turn them into clean, tested, well-documented code.

Two things will be graded:

1. **Correctness** — does it compile, run, and pass the tests?
2. **Discipline** — did you actually use TypeScript's type system to *prevent* bugs, or did you sprinkle `any` everywhere to make the compiler shut up?

---

## Submission rules

- A single GitHub repo `nl-m01-assignment-1`.
- `pnpm install && pnpm test` must work on a fresh clone.
- TypeScript `strict: true`, `noUncheckedIndexedAccess: true`, `noImplicitOverride: true`.
- A `README.md` that explains the design in your own words.
- All code in `src/`. Tests in `tests/`. Use **Vitest** (`pnpm add -D vitest`).
- **No `any` and no `as any` in your final code.** Use `unknown` + narrowing, or fix the type.

---

## Part A — Type-safe utility functions

Implement the following in `src/utils/`. Every function must be **fully typed** and **exhaustively tested**.

### Required functions

```ts
// 1. The classic pick — but with a constraint
export function pick<T extends object, K extends keyof T>(obj: T, keys: K[]): Pick<T, K>;

// 2. A type-safe groupBy
export function groupBy<T, K extends keyof T>(items: T[], key: K): Record<string, T[]>;

// 3. A deep-partial updater — returns a new object with the path set
export function setIn<T>(obj: T, path: string, value: unknown): T;

// 4. A coalesce that takes any number of (T | null | undefined)
export function coalesce<T>(...values: (T | null | undefined)[]): T | undefined;

// 5. A Result helper, generic and discriminated
export type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

export function ok<T>(value: T): Result<T, never>;
export function err<E>(error: E): Result<never, E>;

// 6. A type-safe EventEmitter (the hard one — use generics + mapped types)
export class TypedEmitter<Events extends Record<string, unknown[]>> {
  on<K extends keyof Events>(event: K, listener: (...args: Events[K]) => void): this;
  off<K extends keyof Events>(event: K, listener: (...args: Events[K]) => void): this;
  emit<K extends keyof Events>(event: K, ...args: Events[K]): boolean;
}
```

### Conceptual session 1: "How the JS runtime actually runs your code"

In your `README.md`, write a 1–2 page essay (with code snippets) that explains:

- The **call stack** and the **heap**.
- **Synchronous vs asynchronous** execution — what the event loop actually is.
- How `Promise.then` / `await` interact with **microtasks** vs **macros** (`setTimeout`).
- Why `async/await` is *just* Promise syntax — it does not magically make code parallel.
- A short diagram (ASCII is fine) of how a Node request flows: socket → parser → handler → response.

This is the mental model you need to debug the backends we build in Mission 2.

### Conceptual session 2: "Thinking in types — reading and designing type signatures"

In your `README.md`, add another 1–2 page essay:

- Show three production-style function signatures from real codebases (you can use the ones above, plus any open-source library — Prisma, Zod, Drizzle are great).
- For each, explain **what the type is telling you** about the contract, the failure modes, and the intent.
- Then pick **one** of your own utility functions and explain how you would have written the signature wrong (loose types) and what bugs that would have allowed.
- End with: *"The signature is the API. If the signature lies, the function lies."*

---

## Part B — A small OOP-modelled domain

Pick **one** of the three domains:

### Option 1: Library

- `LibraryItem` (abstract): `id`, `title`, `isAvailable`, `borrow`, `return`.
- `Book`, `DVD`, `Magazine` extending `LibraryItem`.
- `Library`: holds items, can search, list available, lend.
- A custom `LibraryError` hierarchy: `ItemNotFoundError`, `ItemUnavailableError`.
- Polymorphic `describe(): string` on each subclass.

### Option 2: Vehicle

- `Vehicle` (abstract): `vin`, `miles`, `fuelEfficiency(): number`.
- `GasCar`, `ElectricCar`, `HybridCar`.
- `Fleet`: holds vehicles, returns aggregate fuel efficiency.
- Domain-specific errors.
- Polymorphic `serviceInfo(): string`.

### Option 3: Student

- `Student`: `id`, `name`, `courses`, validated `gpa` (0..4) via setter.
- `Course`: `code`, `title`, `credits`.
- `Transcript` service: GPA, courses by semester, honor roll.
- Errors: `CourseFullError`, `InvalidGPAError`.
- Polymorphic `printSummary`.

### Required across all options

- Use `readonly` on every immutable field.
- Use `#private` or `private` for internal state.
- Use `override` (and turn on `noImplicitOverride`).
- Use `instanceof` for runtime narrowing in error handling.
- **No** `any`. **No** `as any`.
- One unit test per public method.
- One integration test that exercises the whole domain.

---

## Grading rubric (out of 100)

| Area | Points |
|------|--------|
| Code compiles and tests pass | 20 |
| Every function fully and correctly typed | 20 |
| OOP model uses inheritance + polymorphism + encapsulation + abstraction correctly | 15 |
| Custom error hierarchy used consistently | 5 |
| `noUncheckedIndexedAccess` clean | 5 |
| README essays (conceptual sessions) — depth and clarity | 20 |
| Test coverage (Vitest) | 10 |
| Code review rubric: naming, file layout, no dead code, no comments-that-lie | 5 |

---

## Deadline and how to submit

- **Deadline:** 7 days from the start of Mission 1.
- **Submit:** push to GitHub, open a PR titled `[NL-M1-A1] <your-name>`, paste the PR link into the submission form.
- **Plagiarism:** none tolerated. You may discuss *concepts* with peers; you may not share code. All submissions are checked with MOSS.

---

## What to read after you submit

- The TypeScript handbook on [Type Compatibility](https://www.typescriptlang.org/docs/handbook/type-compatibility.html).
- [Total TypeScript](https://www.totaltypescript.com/) — Matt Pocock's premium tutorials. The single best TS learning resource in 2026.
- [Type-Level TypeScript](https://type-level-typescript.com/) — when you are ready to be humbled.

---

## AI Industry Note (2026)

The pattern you used in Part A — small, pure, fully-typed utility functions — is **the** shape of modern AI-assisted development. When you ask an AI to "write a function that does X," it will reach for the *exact* shape you have been writing. The same is true of the OOP model in Part B. Train yourself on **disciplined, fully-typed, tested** code, and your AI pair-programmer becomes a 10x multiplier. Train on loose, untested code, and the AI amplifies the mess.
