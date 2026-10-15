# Module 03 — Object-Oriented Programming in TypeScript

> Classes, the four pillars, and the modern TypeScript take on OOP — including when *not* to use it.

---

## Why this module matters

Object-Oriented Programming has a bad reputation in the JavaScript world because, for a long time, JS didn't have `class`. It does now. And when you build real systems — long-lived domain models, framework integrations, plugin systems — OOP remains the cleanest way to organize code.

In Mission 4 you will build a SaaS with workspaces, teams, projects, tasks, and comments. Some of these map beautifully to OOP. Others (pure functions over data) are better expressed as plain functions and discriminated unions. **Knowing which is which** is the skill.

---

## Learning objectives

- Define classes with typed fields, constructors, and parameter properties.
- Apply inheritance correctly and avoid the classic "deep hierarchy" anti-pattern.
- Use type guards with `typeof` and `instanceof` for runtime narrowing.
- Apply access modifiers (`public`, `private`, `protected`).
- Use getters and setters thoughtfully.
- Implement the four pillars: inheritance, polymorphism, abstraction, encapsulation.
- Recognize **composition over inheritance** as the modern default.

---

## 1. Classes and objects

```ts
class User {
  id: string;
  email: string;
  createdAt: Date;

  constructor(id: string, email: string) {
    this.id = id;
    this.email = email;
    this.createdAt = new Date();
  }

  describe(): string {
    return `User<${this.id}>`;
  }
}

const u = new User("u1", "rumi@x.com");
```

### Parameter properties — the shorthand

```ts
class User {
  createdAt = new Date();   // default value, no constructor needed

  constructor(
    public id: string,           // auto-assigns to this.id
    public email: string,        // auto-assigns to this.email
  ) {}
}
```

`public`, `private`, `protected` in a constructor parameter do two things: type the parameter **and** assign it to `this`. Used everywhere in production code.

---

## 2. Inheritance

```ts
class Animal {
  constructor(public name: string) {}
  speak(): string { return "..."; }
}

class Dog extends Animal {
  override speak(): string {           // 'override' is required when noImplicitOverride is on
    return `${this.name}: woof`;
  }
}
```

**Mental model:** a `Dog` *is an* `Animal`. Anywhere you need an `Animal`, you can use a `Dog`.

### When to use inheritance vs composition

| Use inheritance when | Use composition when |
|----------------------|----------------------|
| "is-a" relationship is real and stable | "has-a" relationship |
| Subclass really does need all parent behavior | You only need *some* of the parent's behavior |
| Few levels deep (≤ 2) | You might want multiple "parents" |
| Framework mandates it (e.g. `BaseController`) | You want to swap behavior at runtime |

**The "favor composition" rule:** in 2026, prefer composition for almost everything. Inheritance is for narrow, stable hierarchies (e.g. error classes, framework extension points).

---

## 3. Access modifiers

```ts
class Account {
  public readonly id: string;            // anyone can read, no one can write
  private balance: number;               // only this class
  protected owner: string;               // this class and subclasses

  constructor(id: string, owner: string, initial: number) {
    this.id = id;
    this.owner = owner;
    this.balance = initial;
  }

  public deposit(amount: number) {
    if (amount <= 0) throw new Error("amount must be positive");
    this.balance += amount;
  }

  public getBalance(): number {
    return this.balance;
  }
}
```

`readonly` is *compile-time only*. For true immutability at runtime, use `Object.freeze` or a true immutable type from a library.

### The "private vs hash-private" question

```ts
class A {
  private x = 1;        // soft privacy — gone at runtime
  #y = 2;               // hard privacy — true ECMAScript private field
}
```

`#field` is *true* privacy enforced by the engine. Use it when external code must not touch the field even via `(obj as any).field`.

---

## 4. Getters and setters

```ts
class Temperature {
  #celsius: number;

  constructor(celsius: number) { this.#celsius = celsius; }

  get fahrenheit(): number {
    return this.#celsius * 9/5 + 32;
  }

  set fahrenheit(value: number) {
    this.#celsius = (value - 32) * 5/9;
  }
}

const t = new Temperature(0);
console.log(t.fahrenheit);   // 32
t.fahrenheit = 212;
console.log(t.fahrenheit);   // 412 ❌ — set was wrong, also should be 212 input
```

Use getters/setters when:
- The value is **computed** (like `fahrenheit` above).
- You need **validation** on assignment.
- You want to maintain a **public read-only** view of internal state.

**Don't** use them for fields that are just storage — a regular `public` field is clearer.

---

## 5. The four pillars of OOP

### 5.1 Inheritance — *is-a*

```ts
class Entity {                     // base
  constructor(public id: string) {}
}

class Workspace extends Entity {   // workspace IS-AN entity
  constructor(id: string, public name: string) {
    super(id);
  }
}
```

### 5.2 Polymorphism — *many forms*

```ts
interface Shape { area(): number; }

class Circle implements Shape {
  constructor(public r: number) {}
  area() { return Math.PI * this.r ** 2; }
}

class Square implements Shape {
  constructor(public s: number) {}
  area() { return this.s ** 2; }
}

function totalArea(shapes: Shape[]): number {
  return shapes.reduce((sum, s) => sum + s.area(), 0);
}
```

`Shape` doesn't know or care which concrete class it gets. **This is the most-used OOP pattern in real codebases.**

### 5.3 Abstraction — *hide the complexity*

```ts
abstract class PaymentProcessor {
  constructor(protected apiKey: string) {}

  // Template method — defines the skeleton
  async charge(amount: number): Promise<void> {
    await this.validate(amount);
    const result = await this.callProvider(amount);
    await this.recordResult(result);
  }

  protected abstract validate(amount: number): void;
  protected abstract callProvider(amount: number): Promise<{ ok: boolean }>;
  protected abstract recordResult(r: { ok: boolean }): Promise<void>;
}

class StripeProcessor extends PaymentProcessor {
  protected validate(amount: number) {
    if (amount <= 0) throw new Error("invalid amount");
  }
  protected async callProvider(amount: number) {
    // real Stripe call
    return { ok: true };
  }
  protected async recordResult(r: { ok: boolean }) {
    // write to our DB
  }
}
```

The base class controls the *flow*; subclasses control the *details*. This is the strategy pattern.

### 5.4 Encapsulation — *protect the invariant*

```ts
class Stack<T> {
  readonly #items: T[] = [];

  push(item: T): void {
    this.#items.push(item);
  }

  pop(): T | undefined {
    return this.#items.pop();
  }

  get size(): number {
    return this.#items.length;
  }
}
```

External code cannot bypass `push`/`pop` and corrupt internal state. The **invariant** (LIFO order) is protected.

---

## 6. Type guards with `instanceof`

When you receive an `unknown` or a union, you often need to narrow at runtime:

```ts
class ValidationError extends Error { constructor(public field: string) { super(...); } }
class AuthError extends Error { constructor() { super("auth"); } }

function handle(err: unknown) {
  if (err instanceof ValidationError) {
    console.log("Bad field:", err.field);
  } else if (err instanceof AuthError) {
    redirectToLogin();
  } else {
    throw err;
  }
}
```

`instanceof` is the runtime companion to the class hierarchy. Combine it with discriminated unions for the cleanest result.

---

## 7. Static members

```ts
class ID {
  static next(): string {
    return ID._counter++;
  }
  private static _counter = 1;
}
```

Static fields and methods belong to the class itself, not instances. Used for utilities, factories, and constants.

For a modern alternative, consider module-level functions:

```ts
// id.ts
let counter = 1;
export const nextId = (): string => `id_${counter++}`;
```

---

## 8. A small OOP domain — Library system (Assignment 1 prep)

This is the model you'll use in Assignment 1.

```ts
abstract class LibraryItem {
  readonly id: string;
  readonly title: string;
  protected borrowedBy: string | null = null;

  constructor(id: string, title: string) {
    this.id = id;
    this.title = title;
  }

  isAvailable(): boolean { return this.borrowedBy === null; }

  borrow(userId: string): void {
    if (!this.isAvailable()) throw new Error("already borrowed");
    this.borrowedBy = userId;
  }

  return(): void {
    if (this.isAvailable()) throw new Error("not borrowed");
    this.borrowedBy = null;
  }
}

class Book extends LibraryItem {
  constructor(id: string, title: string, public author: string) {
    super(id, title);
  }
}

class DVD extends LibraryItem {
  constructor(id: string, title: string, public runtimeMin: number) {
    super(id, title);
  }
}

class Magazine extends LibraryItem {
  constructor(id: string, title: string, public issue: number) {
    super(id, title);
  }
}
```

### Vehicle system (alternative)

```ts
abstract class Vehicle {
  constructor(public readonly vin: string, public miles: number) {}
  abstract fuelEfficiency(): number;   // mpg or kWh/mile
}

class GasCar extends Vehicle {
  constructor(vin: string, miles: number, public tankGallons: number) { super(vin, miles); }
  fuelEfficiency() { return this.miles / this.tankGallons; }
}

class ElectricCar extends Vehicle {
  constructor(vin: string, miles: number, public kwh: number) { super(vin, miles); }
  fuelEfficiency() { return this.miles / this.kwh; }
}
```

### Student system (alternative)

```ts
class Course {
  constructor(public code: string, public title: string) {}
}

class Student {
  private _gpa: number = 0;
  constructor(public id: string, public name: string, public courses: Course[] = []) {}

  enroll(c: Course) { this.courses.push(c); }

  get gpa(): number { return this._gpa; }
  set gpa(v: number) {
    if (v < 0 || v > 4) throw new RangeError("gpa out of range");
    this._gpa = v;
  }
}
```

Pick one. Build it out. Add the polymorphism: write a function `describeItem` (or `describeVehicle` / `printTranscript`) that works on any subclass.

---

## 9. Common beginner mistakes

1. **Deep inheritance trees.** Three levels is usually a sign you needed composition.
2. **God classes.** A class with 30 fields and 50 methods is doing 5 jobs. Split it.
3. **Public mutable state.** Make fields `readonly` by default; expose `private` with controlled setters.
4. **Forgetting `super()` in subclass constructors.** Always call `super()` first.
5. **Treating `private` as security.** It is not. It is documentation + compile-time discipline. For real security, use closures or the `#` private fields.
6. **Forgetting `override`.** With `noImplicitOverride: true` (recommended), the compiler will refuse code that overrides without the keyword. Use it.

---

## Exercise — Mission recap

Build the **Library system** above (or the Vehicle or Student system) and add:

1. A `Library` class that holds items, can `add`, `findById`, `listAvailable`, and `borrow(itemId, userId)`.
2. A type guard `isBook(x: unknown): x is Book` and equivalents.
3. A `printSummary(items: LibraryItem[])` that uses polymorphism.
4. A custom error `ItemNotFoundError` extending `Error`, and use it inside `findById`.
5. Use `readonly` and `private`/`#` aggressively. State your invariants in a comment block at the top of each class.

Run `tsc --noEmit` and `tsx src/practice.ts` and verify behavior. **Read your own code and ask: would a senior engineer be proud of this?**

---

## AI Industry Note (2026)

- **The "OOP is dead" myth.** Functional-leaning JS codebases (think Effect, fp-ts) are growing, but most real production codebases in 2026 are still class-based for the long-lived domain layer and function-based for request handlers. Master both.
- **AI assistants default to classes when you use them.** If your repository, your Drizzle schema, and your DI container all use classes, the AI will follow that pattern. If your codebase is purely functional, it will mirror that. **You set the style; the AI amplifies it.**
- **What hiring managers look for in 2026:** can you model a real domain (e.g. "a multi-tenant SaaS with workspaces, teams, projects, tasks, comments, and file attachments") in a way that the **invariants are protected by the type system**? If you can show that in a 30-minute system-design interview, you are hired. The Library / Vehicle / Student exercises are warmups for that conversation.
