# Module 02 — Advanced TypeScript Types

> The toolkit that turns TypeScript from "JavaScript with a linter" into a real design tool.

---

## Why this module matters

In Module 01 you learned the *grammar*. In this module you learn the *idioms* — the patterns senior engineers use to encode business rules in the type system so that bugs are caught at compile time, not at 3am.

The single most valuable skill you can take from this module is **generic functions with constraints**. Once you can write a function `<T extends { id: string }>(items: T[], id: string): T | undefined`, you can read 80% of the production codebases you will ever touch.

---

## Learning objectives

- Use **type assertion** deliberately and know when to avoid it.
- Choose between **interface** and **type alias** with confidence.
- Write **generics** with a clear mental model.
- Apply **constraints** (`extends`) and the `keyof` operator.
- Use **conditional types** and the `infer` keyword.
- Use **mapped types** to transform object shapes.
- Know when to use `enum` vs `as const` objects.
- Apply **utility types**: `Partial`, `Pick`, `Omit`, `Record`, `Required`, `Readonly`.

---

## 1. Type assertion — handle with care

```ts
const input = document.getElementById("email") as HTMLInputElement;
input.value;
```

A type assertion is a **lie you tell the compiler**: "trust me, I know better." Use it only when you have evidence the compiler doesn't (e.g. you just checked with `instanceof`).

**Never** do this:

```ts
const data = JSON.parse(text) as User;   // ❌ trusts the JSON
```

The JSON is *runtime* data. The assertion is *compile-time* fiction. Use Zod (Mission 4) instead.

### Double assertion — almost always a code smell

```ts
const v = (value as unknown) as User;  // ❌ if you need this, redesign
```

---

## 2. Interface vs type alias

```ts
interface User {
  id: string;
  email: string;
}

type User2 = {
  id: string;
  email: string;
};
```

For object shapes they are nearly identical. Differences that matter:

| Feature | `interface` | `type` |
|---------|-------------|--------|
| Can be extended with `extends` | ✅ | ✅ (`&`) |
| Can be merged (declaration merging) | ✅ | ❌ |
| Can represent unions/tuples/functions | ❌ | ✅ |
| Better error messages in some cases | ✅ | varies |
| Can be implemented by a class | ✅ | ✅ |

**Rule of thumb used in this course:**

- `interface` for **public, stable object contracts** (DTOs, domain models).
- `type` for **everything else** (unions, mapped types, function signatures, computed shapes).

---

## 3. Generics — the mental model

A generic is a **type-level variable**.

```ts
function identity<T>(value: T): T {
  return value;
}

const a = identity(42);          // T inferred as number
const b = identity("hi");        // T inferred as string
const c = identity<User>(user);  // T explicit
```

Mental model: think of `T` as a placeholder that gets filled in at the call site. The compiler verifies that the function body is valid for *any* valid `T`.

### A realistic generic — `firstOrNull`

```ts
function firstOrNull<T>(items: T[]): T | null {
  return items.length > 0 ? items[0] : null;
}

const u = firstOrNull(users);     // User | null
const n = firstOrNull([1, 2, 3]); // number | null
```

### Multiple type parameters

```ts
function map<T, U>(items: T[], fn: (item: T) => U): U[] {
  return items.map(fn);
}

const lengths = map(["hi", "world"], (s) => s.length); // number[]
```

---

## 4. Constraints — `extends`

What if `T` needs to have an `id`?

```ts
function findById<T extends { id: string }>(items: T[], id: string): T | undefined {
  return items.find((item) => item.id === id);
}
```

The constraint `T extends { id: string }` says: "T can be any type, as long as it has an `id: string` field."

```ts
findById([{ id: "1", name: "Rumi" }], "1");  // ✅
findById([{ name: "Rumi" }], "1");           // ❌ Property 'id' is missing
```

---

## 5. `keyof` — the operator that unlocks the standard library

```ts
type User = { id: string; email: string; name: string };
type UserKeys = keyof User;  // "id" | "email" | "name"
```

Combined with generics, `keyof` lets you build type-safe property accessors:

```ts
function pluck<T, K extends keyof T>(items: T[], key: K): T[K][] {
  return items.map((item) => item[key]);
}

const emails = pluck(users, "email");
//    ^? string[]

const names = pluck(users, "name");
//    ^? string[]
```

`pluck(users, "emial")` is a compile error. This is the kind of generic that *earns its keep* in production.

---

## 6. Conditional types

```ts
type IsString<T> = T extends string ? true : false;
type A = IsString<"hi">;  // true
type B = IsString<42>;    // false
```

Think of it as the type-level equivalent of a ternary. Used for:

- **Filtering unions:** `Extract<T, U>`, `Exclude<T, U>`.
- **Resolving API response shapes** based on input.
- **Library internals** (Zod, tRPC, Drizzle all rely on this).

### The `infer` keyword

```ts
type ReturnTypeOf<T> = T extends (...args: any[]) => infer R ? R : never;

type X = ReturnTypeOf<() => number>;   // number
type Y = ReturnTypeOf<string>;         // never
```

`infer` lets you **declare a new type variable** inside a conditional. This is how the built-in `ReturnType<typeof fn>` is defined.

A real example — extract the element type of an array:

```ts
type ElementOf<T> = T extends (infer E)[] ? E : never;

type N = ElementOf<number[]>;   // number
type U = ElementOf<User[]>;    // User
```

---

## 7. Mapped types — transform object shapes

```ts
type Readonly<T> = {
  readonly [K in keyof T]: T[K];
};

type Optional<T> = {
  [K in keyof T]?: T[K];
};
```

This is the exact shape of TypeScript's built-in utility types. You can write your own:

```ts
// Nullable — every field becomes T | null
type Nullable<T> = { [K in keyof T]: T[K] | null };

// Getters — turn values into () => value
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
```

### Mapping with `as` for renaming

```ts
type EventPayloads = {
  click: { x: number; y: number };
  submit: { formId: string };
};

type Handlers = {
  [E in keyof EventPayloads as `on${Capitalize<string & E>}`]: (p: EventPayloads[E]) => void;
};
// { onClick: (p: {x:number;y:number}) => void; onSubmit: ... }
```

This is the pattern that powers event-emitter libraries.

---

## 8. `enum` vs `as const` object

```ts
// enum — old-school, has runtime cost, can be surprising
enum Color { Red, Green, Blue }
const c: Color = Color.Green;

// as const object — modern, zero runtime cost, plays well with tree-shaking
const Color = {
  Red: "RED",
  Green: "GREEN",
  Blue: "BLUE",
} as const;

type Color = (typeof Color)[keyof typeof Color];
// type Color = "RED" | "GREEN" | "BLUE"
```

**Recommendation in 2026:** prefer `as const` objects. They are explicit, tree-shakeable, and work perfectly with discriminated unions.

---

## 9. Utility types you will use every day

Given:

```ts
type User = {
  id: string;
  email: string;
  name: string;
  passwordHash: string;
};
```

| Utility | What it does | Use case |
|---------|--------------|----------|
| `Partial<User>` | All fields optional | update endpoints |
| `Required<User>` | All fields required (no `?`) | config objects |
| `Pick<User, "id" \| "email">>` | Subset of fields | public DTOs |
| `Omit<User, "passwordHash">` | All except these | safe responses |
| `Record<"admin" \| "user", User[]>` | Object map | indexes, lookup tables |
| `Readonly<User>` | All fields readonly | immutability enforcement |
| `NonNullable<T>` | Strips `null` and `undefined` | post-validation |
| `ReturnType<typeof fn>` | Function's return type | derive types without re-declaring |

### `Pick` + `Omit` for DTO discipline

```ts
// Never leak the password hash to the client
type PublicUser = Omit<User, "passwordHash">;

// For PATCH /users/:id, every field is optional
type UpdateUser = Partial<Pick<User, "name" | "email">>;
```

This is the exact pattern we will use in Mission 4 for every endpoint.

---

## 10. Putting it together — a type-safe repository

```ts
type Entity = { id: string };

interface Repository<T extends Entity> {
  findById(id: string): Promise<T | null>;
  list(filter?: Partial<Omit<T, "id">>): Promise<T[]>;
  create(input: Omit<T, "id">): Promise<T>;
  update(id: string, patch: Partial<Omit<T, "id">>): Promise<T>;
  delete(id: string): Promise<void>;
}
```

This is a **generic contract** that any repository in our backend will satisfy. By the time you reach Mission 4 you will be writing this exact interface — and the compiler will guarantee every implementation matches it.

---

## Common mistakes

1. **Over-using `keyof` in mapped types** when a simple `Pick` would do.
2. **Conditional types that confuse readers.** If a junior cannot follow the type in 30 seconds, prefer a `type` alias that names each branch.
3. **`as any` to "escape" the type system.** Once you do this, the system stops helping you.
4. **Enums in new code.** Use `as const` objects.

---

## Exercise

1. Write a generic `groupBy<T, K extends keyof T>(items: T[], key: K): Record<string, T[]>`.
2. Define a `Patch<T>` type that makes all fields optional *except* `id`.
3. Write a `Result<T, E = Error>` discriminated union and an `isOk` / `isErr` type guard.
4. Using `as const`, define a `Permission` object with values like `"user:read"`, `"user:write"`, etc., and derive a `Permission` type.
5. Implement `DeepReadonly<T>` for arbitrary nested objects.

Verify everything with `tsc --noEmit` and a small test file.

---

## AI Industry Note (2026)

- **The Zod / Drizzle / tRPC pattern.** In 2026, the dominant pattern in the TypeScript ecosystem is: **runtime schema is the source of truth, TypeScript types are inferred from it.** Zod gives you a runtime validator whose output is a TypeScript type. Drizzle gives you a database schema whose table types are TypeScript. tRPC gives you an API contract that *is* TypeScript. We use all three in Missions 4 and 6.
- **Generics are how AI writes good code.** When you ask an AI assistant to "write a function that takes a list of users and returns a map of id → user," it will reach for generics and `Record` if your existing code uses them. **The patterns you use become the patterns the AI uses.** Train yourself on these idioms now; your future AI pair-programmer will be better for it.
- **The interview question that separates juniors from seniors in 2026:** *"How would you model an update endpoint where every field is optional but at least one must be present, in TypeScript?"* If your answer is `Partial<T> & { id: string }` plus a Zod refinement, you're senior. If your answer is "I'd just check in the function body," we're not ready yet.
