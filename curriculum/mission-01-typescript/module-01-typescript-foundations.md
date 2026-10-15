# Module 01 — TypeScript Foundations

> From "plain JavaScript is a liability at scale" to "TypeScript is the default for backend, frontend, and AI in 2026."

---

## Why this module matters

JavaScript lets you shoot yourself in the foot. TypeScript hands you a holster.

In every production codebase I have worked on in the last five years, the moment we adopted TypeScript:

- **Runtime errors dropped** — type-related bugs become compile errors.
- **Refactors became fearless** — rename a property across 40 files, the compiler tells you every call site.
- **AI assistants got dramatically more useful** — TypeScript's type system gives them the same hints it gives you, so they hallucinate less.

If you skip this module and jump to "just write the code," you will spend Mission 4 fighting the type system instead of building product.

---

## Learning objectives

By the end of this module you can:

- Explain *why* TypeScript exists and what it does at compile time vs runtime.
- Set up a TypeScript project from scratch with `tsconfig.json`.
- Use the basic types: `string`, `number`, `boolean`, `null`, `undefined`, `bigint`, `symbol`.
- Annotate arrays, tuples, and objects correctly.
- Type functions, parameters, return values, and use optional / default parameters.
- Use union types, literal types, and type aliases.
- Apply the literal type, nullish coalescing, and optional chaining operators.
- Choose between `any`, `unknown`, `never`.

---

## 1. Why TypeScript exists: the problem with plain JavaScript

Consider this real bug, taken from a 2018 production incident:

```js
// user-service.js
function createUser(payload) {
  return db.insert({ name: payload.name, emial: payload.emial });
}

// caller
createUser({ name: "Rumi", email: "rumi@x.com" });
```

Look carefully. The function wrote `emial` instead of `email` **twice**. JavaScript happily stored the typo'd key. The caller's `email` field was silently dropped. The user got created with `email: undefined` and 6 months later we found a production table full of users without emails.

In TypeScript:

```ts
type CreateUserInput = { name: string; email: string };
function createUser(payload: CreateUserInput) {
  return db.insert({ name: payload.name, email: payload.email });
}
```

The typo is now a compile error. The bug is impossible.

**Mental model:** TypeScript is a *static analysis layer* over JavaScript. It compiles (or rather, **transpiles**) to plain JS. The runtime is unchanged. All the type annotations you write disappear at build time.

---

## 2. TypeScript setup and `tsconfig.json`

### Installing

```bash
mkdir ts-foundations && cd ts-foundations
pnpm init
pnpm add -D typescript @types/node tsx
npx tsc --init
```

`tsc --init` generates a `tsconfig.json`. For a modern Node 22 + ESM project, here is the minimum you need:

```json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2023"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "sourceMap": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### The strict flags that matter

| Flag | What it does |
|------|--------------|
| `"strict": true` | Turns on the bundle: `noImplicitAny`, `strictNullChecks`, etc. **Never turn it off.** |
| `"noUncheckedIndexedAccess": true` | `arr[0]` becomes `T \| undefined`. Catches 80% of off-by-one bugs. |
| `"noImplicitOverride": true` | Forces `override` keyword. Future-proof for classes. |
| `"forceConsistentCasingInFileNames": true` | Linux deployments will thank you. |

### Compiling and running

```bash
npx tsc          # type-check + emit
npx tsx src/index.ts   # run directly (great for dev)
```

`tsx` is the modern replacement for `ts-node`. It uses esbuild under the hood and is ~50x faster.

---

## 3. Basic data types

```ts
// Primitives
const name: string = "Rumi";
const age: number = 28;            // no separate int/float
const isPaid: boolean = true;
const nothing: null = null;
const notDefined: undefined = undefined;
const big: bigint = 9007199254740993n;  // note the 'n' suffix
const sym: symbol = Symbol("id");

// Arrays — two equivalent syntaxes
const tags: string[] = ["ai", "ts"];
const scores: Array<number> = [98, 95, 100];

// Tuples — fixed length, fixed types per position
const point: [number, number] = [10, 20];
const entry: [string, number, boolean] = ["login", 200, true];

// Object types — prefer interface or type alias, not inline
type User = {
  id: string;
  email: string;
  age: number;
};
```

**Common mistake:** Using `Number`, `String`, `Boolean` (capitalized) — these are boxed wrapper objects from JS and almost never what you want. Always use the lowercase primitive types.

---

## 4. Arrays and objects

### Readonly arrays

```ts
const roles: readonly string[] = ["admin", "editor"];
// roles.push("viewer"); // ❌ Property 'push' does not exist
```

We will use `readonly` heavily in Mission 4 to enforce immutability on data flowing through services.

### Index signatures

When the keys are dynamic:

```ts
type FeatureFlags = Record<string, boolean>;
const flags: FeatureFlags = { darkMode: true, betaAi: false };
```

---

## 5. Functions, parameters, return types

```ts
// Explicit return type — recommended for exported functions
function add(a: number, b: number): number {
  return a + b;
}

// Void return — function returns nothing on purpose
function log(msg: string): void {
  console.log(msg);
}

// Never return — function never returns (throws, infinite loop)
function fail(msg: string): never {
  throw new Error(msg);
}
```

### Optional parameters

```ts
function greet(name: string, greeting?: string): string {
  return `${greeting ?? "Hello"}, ${name}`;
}
```

`?` means the parameter can be `undefined` when called. Inside the function, the type is `string | undefined`.

### Default parameters

```ts
function greet(name: string, greeting: string = "Hello"): string {
  return `${greeting}, ${name}`;
}
```

Default parameters are *not* the same as optional. A default value means the parameter has a guaranteed type inside the function body.

---

## 6. Literal types

Sometimes you want a parameter to be *exactly* one of a small set of strings:

```ts
type Theme = "light" | "dark" | "system";
function setTheme(theme: Theme) { /* ... */ }

setTheme("dark");    // ✅
setTheme("purple");  // ❌ Argument of type '"purple"' is not assignable
```

This is one of the highest-leverage TypeScript features. Use it everywhere you have a closed set of values (statuses, roles, HTTP methods, etc.).

---

## 7. Union and intersection types

### Union — "either A or B"

```ts
type Result =
  | { kind: "success"; data: string }
  | { kind: "error";   error: Error };

function handle(r: Result) {
  if (r.kind === "success") {
    r.data.toUpperCase();  // TS narrows to the success branch
  } else {
    r.error.message;       // TS narrows to the error branch
  }
}
```

This pattern (a **discriminated union**) is *the* pattern you will use for the entire backend. Status fields, state machines, API responses — all of them.

### Intersection — "A and B"

```ts
type WithTimestamps = { createdAt: Date; updatedAt: Date };
type User = { id: string; email: string };
type UserWithTimestamps = User & WithTimestamps;
```

---

## 8. Type aliases

```ts
type UserId = string;            // semantic alias
type Status = "active" | "banned";
type Handler = (req: Request) => Promise<Response>;
```

**`type` vs `interface`?** For object shapes, both work. Convention used in this course:

- `type` for unions, intersections, primitives, function types.
- `interface` for object shapes that may be **extended** or **implemented**.

We will go deep on this in Module 02.

---

## 9. Ternary, nullish coalescing, optional chaining

```ts
const score = isFinal ? 100 : calculate();

const displayName = user.nickname ?? user.name ?? "Anonymous";

const city = user?.address?.city;  // undefined if any link is null
```

### The three operators cheat sheet

| Operator | Null / undefined | Empty string | 0 | false |
|----------|------------------|--------------|---|-------|
| `??`     | fallback         | keeps        | keeps | keeps |
| `\|\|`   | fallback         | fallback     | fallback | fallback |
| `?.`     | returns undefined | keeps     | keeps | keeps |

`??` is what you almost always want for default values. `||` lies when the value is a valid `0` or `""`.

---

## 10. `null`, `undefined`, `unknown`, `never`

| Type | What it means | When to use |
|------|---------------|-------------|
| `null` | explicitly "no value" | when you want a sentinel |
| `undefined` | "not yet set" | function params not provided, missing object keys |
| `unknown` | "I don't know the type yet" | external input — parse before use |
| `never` | "this code can never produce a value" | exhaustive checks, functions that throw |

### The `unknown` escape hatch — done right

```ts
function parseJSON(text: string): unknown {
  return JSON.parse(text);
}

const raw = parseJSON(input);
// raw.email  // ❌ Object is of type 'unknown'

if (typeof raw === "object" && raw !== null && "email" in raw) {
  // raw is now narrowed — but still use a validator
}
```

For real safety, pair `unknown` with a runtime validator (Zod, which we use in Mission 4).

### The `never` exhaustive check

```ts
type Status = "pending" | "done";
function label(s: Status): string {
  switch (s) {
    case "pending": return "Waiting";
    case "done":    return "Finished";
    default:
      const _exhaustive: never = s;  // compile error if a case is missing
      return _exhaustive;
  }
}
```

If you ever add `"failed"` to `Status` and forget to handle it, this line screams at you. I put this pattern in every state machine.

---

## Common mistakes I see every cohort

1. **Using `any` to silence errors.** `any` is contagious — it spreads. Prefer `unknown` and narrow.
2. **Forgetting `strict: true`.** A non-strict tsconfig is worse than no TypeScript at all.
3. **Annotating everything.** Let inference do the work for locals; annotate at *boundaries* (function signatures, exported APIs, configs).
4. **`== null` vs `=== undefined`.** `==` with `null` is one of the few "loose equals" patterns that is actually useful and conventional in TS — it matches both `null` and `undefined`. Use it deliberately.

---

## Exercise — do this before Module 02

Create a file `src/practice.ts` and implement the following without using `any`:

1. A function `formatName(first: string, last?: string)` that returns `"Last, First"` or just `"First"` if no last name.
2. A function `coalesce<T>(...values: (T | null | undefined)[]): T | undefined` that returns the first non-null/non-undefined value.
3. A type `ApiResult<T>` as a discriminated union of `{ status: "ok"; data: T }` and `{ status: "error"; message: string }`. Write a function `unwrap` that returns `T` or throws.
4. A literal type `LogLevel = "debug" | "info" | "warn" | "error"` and a function `log(level: LogLevel, message: string)` that no-ops for `"debug"`.

Verify with `npx tsc --noEmit`.

---

## AI Industry Note (2026)

- **TypeScript is now the default for AI code generation.** When you ask Claude or GPT-4o to write a function, the prompt *"in TypeScript, with explicit return types"* produces measurably better code than *"in JavaScript"*. Why? Because the type system constrains the model's search space. The first thing you should do in any AI session is establish the type contracts.
- **Zod + TypeScript = "types that don't lie."** TypeScript types exist at compile time only. If a value comes from a database, an HTTP request, or an LLM JSON output, the runtime type is `unknown`. Zod (we use it heavily in Mission 4) lets you *infer* TypeScript types from runtime schemas. By 2026, this is the standard pattern for "data that crosses a trust boundary."
- **Interview signal:** in 2026, a TypeScript portfolio with discriminated unions, exhaustive `never` checks, and `readonly` discipline will be ranked above one that uses clever generics but has sloppy null handling. Show the boring discipline; that is what gets you hired.
