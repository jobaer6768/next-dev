# Reward Course 03 — Testing & Quality

> Reward module. Builds on Mission 2, Mission 3 (databases), Mission 4, and Reward 02 (NestJS). Estimated effort: 12–15 hours.

---

## Why this module matters

In Mission 4 you shipped a working SaaS. You probably have *some* tests — the ones you wrote to make CI green. But the difference between "tests exist" and "the test suite actually catches bugs before users do" is enormous, and it is the difference between a junior and a senior engineer in 2026.

This module is not "how to write a unit test." You already know that. It is: what to test, what to skip, how to write tests that don't rot, how to debug something that fails only on a colleague's laptop, and how to know when you have *enough* coverage.

The hard truth: **100% coverage is a smell.** A test that calls a function without asserting anything is worse than no test, because it lies. We will replace the "more tests is better" myth with a working mental model.

---

## Learning objectives

By the end of this module you will be able to:

1. Apply the testing pyramid and decide which layer to invest in.
2. Write unit tests with Vitest, using matchers, mocks, and fixtures.
3. Write integration tests against a real database.
4. Test an HTTP API end-to-end with Supertest.
5. Use `vi.fn`, `vi.mock`, and the test-double vocabulary.
6. Run the TDD red-green-refactor loop.
7. Debug with breakpoints, structured logs, and the scientific method.
8. Read a coverage report and know what it actually tells you.
9. Keep a test suite maintainable as the codebase grows.
10. Set up a CI pipeline with pre-merge gates and smoke tests.

---

## 1. Testing Fundamentals

### The testing pyramid

```text
                ╱╲
               ╱  ╲           E2E (Playwright, Supertest)
              ╱    ╲          slow, brittle, expensive
             ╱──────╲         ← 5–10 tests, the critical paths
            ╱        ╲
           ╱          ╲       Integration (real DB, real queue)
          ╱            ╲      medium speed
         ╱──────────────╲     ← 20–50 tests, the boundaries
        ╱                ╲
       ╱                  ╲   Unit (Vitest, pure logic)
      ╱                    ╲  fast, cheap, deterministic
     ╱──────────────────────╲ ← hundreds, the rules of the system
```

The pyramid is a *recommendation*, not a law. The shape says: lots of unit tests, fewer integration tests, very few e2e. The reason is feedback loop time:

| Layer | Feedback | Speed | What it catches |
|---|---|---|---|
| Unit | 10ms | very fast | logic bugs |
| Integration | 100ms | fast | wiring bugs, SQL bugs |
| E2E | 5s | slow | user-flow regressions |

If your e2e suite takes 30 minutes, you will not run it before every commit. If your unit tests take 5 seconds, you will run them on every save.

### What to test

- **Business rules.** "Premium users get 10% off." "You cannot delete a project with active members." These change often and break silently.
- **Boundaries.** "The API returns 400 when the body is invalid." "Stripe webhook is idempotent."
- **Bugs that already happened.** When you fix a bug, write a test that fails on the old code and passes on the new. This is regression coverage.

### What NOT to test

- **Framework code.** Don't test that Express returns JSON when you call `res.json`. It does.
- **Trivial getters and setters.** They have no behavior.
- **Third-party SDK internals.** Trust Stripe to charge cards; you test your call to Stripe with a mock.
- **Implementation details.** "This function calls `bcrypt` 3 times" is not a test, it's a snapshot. Test the *behavior* (correct hash produced) not the *mechanism* (called 3 times).

---

## 2. Unit Testing with Vitest

Vitest is the modern Jest-compatible runner for Vite/ESM projects. The API is almost identical to Jest, with some improvements.

```bash
npm i -D vitest
```

```json
// package.json
"scripts": {
  "test": "vitest run",
  "test:watch": "vitest"
}
```

### A real test

```ts
// pricing.test.ts
import { describe, it, expect } from 'vitest';
import { applyDiscount } from './pricing';

describe('applyDiscount', () => {
  it('returns 0 for empty cart', () => {
    expect(applyDiscount(0, 'PREMIUM')).toBe(0);
  });

  it('applies 10% for premium', () => {
    expect(applyDiscount(100, 'PREMIUM')).toBe(90);
  });

  it('applies 20% for premium over $1000', () => {
    expect(applyDiscount(2000, 'PREMIUM')).toBe(1600); // 10% + extra 10%
  });

  it('is case-insensitive on plan', () => {
    expect(applyDiscount(100, 'premium')).toBe(90);
  });

  it.each([
    [100, 'FREE', 100],
    [100, 'BASIC', 100],
    [100, 'PREMIUM', 90],
  ])('returns %i for plan %s', (amount, plan, expected) => {
    expect(applyDiscount(amount, plan)).toBe(expected);
  });
});
```

### Matchers cheat sheet

```ts
expect(x).toBe(3)            // strict ===
expect(x).toEqual({ a: 1 })  // deep
expect(x).toBeNull()
expect(x).toBeTruthy()
expect(x).toBeGreaterThan(2)
expect(arr).toContain(3)
expect(arr).toHaveLength(3)
expect(fn).toThrow('bad input')
expect(obj).toMatchObject({ a: 1 }) // partial
```

### Fixtures

Reusable test data. Don't repeat yourself inside a test file.

```ts
const user = (over = {}) => ({ id: 1, email: 'a@b.c', role: 'MEMBER', ...over });
```

The function returns a *fresh* object so tests can't share mutable state.

---

## 3. Integration Testing

Unit tests lie if your SQL is wrong. Integration tests catch that.

The pattern: spin up a *real* Postgres (often via Docker or testcontainers), run migrations, and have the test use it.

```ts
// users.integration.test.ts
import { PrismaClient } from '@prisma/client';
import { beforeAll, afterAll, beforeEach, describe, it, expect } from 'vitest';

const prisma = new PrismaClient({ datasources: { db: { url: process.env.TEST_DATABASE_URL! } } });

beforeAll(async () => { await prisma.$connect(); });
afterAll(async () => { await prisma.$disconnect(); });
beforeEach(async () => { await prisma.user.deleteMany(); });

describe('UsersRepository', () => {
  it('creates and finds a user by email', async () => {
    await prisma.user.create({ data: { email: 'a@b.c', name: 'A' } });
    const found = await prisma.user.findUnique({ where: { email: 'a@b.c' } });
    expect(found?.name).toBe('A');
  });
});
```

**Use a separate test database.** Never run tests against your dev DB. A common pattern: `docker compose up -d test-db` in CI, then `npm test`.

---

## 4. API Testing with Supertest

Supertest boots the Express app in-process and fires HTTP requests against it without a real port.

```ts
// app.test.ts
import request from 'supertest';
import { app } from './app';

describe('POST /auth/login', () => {
  it('returns 200 and a token for valid credentials', async () => {
    const res = await request(app)
      .post('/auth/login')
      .send({ email: 'a@b.c', password: 'password' });

    expect(res.status).toBe(200);
    expect(res.body.accessToken).toMatch(/^eyJ/);
  });

  it('returns 401 for bad credentials', async () => {
    const res = await request(app)
      .post('/auth/login')
      .send({ email: 'a@b.c', password: 'wrong' });

    expect(res.status).toBe(401);
  });
});

describe('GET /projects', () => {
  it('returns 401 without a token', async () => {
    const res = await request(app).get('/projects');
    expect(res.status).toBe(401);
  });

  it('returns the user projects', async () => {
    const token = await loginAs('a@b.c');
    const res = await request(app)
      .get('/projects')
      .set('Authorization', `Bearer ${token}`);
    expect(res.status).toBe(200);
    expect(Array.isArray(res.body)).toBe(true);
  });
});
```

If you're on Nest, the equivalent is `INestApplication`:

```ts
const moduleRef = await Test.createTestingModule({ imports: [AppModule] }).compile();
const app = moduleRef.createNestApplication();
await app.init();
return request(app.getHttpServer()).post('/auth/login')...
```

---

## 5. Mocking & Test Doubles

Gerard Meszaros's vocabulary (worth learning):

| Double | What it is | When |
|---|---|---|
| **Dummy** | Passed but never used | Filling required args |
| **Stub** | Returns canned answers | Controlling downstream |
| **Spy** | Records how it was called | Verifying behavior |
| **Mock** | Stub + spy + assertions | "Expected to be called twice" |
| **Fake** | Working implementation, simplified | In-memory DB |

In Vitest:

```ts
import { vi } from 'vitest';

// stub
const sendEmail = vi.fn().mockResolvedValue({ id: 'msg_1' });

// spy on existing
const spy = vi.spyOn(console, 'log');

// mock a module
vi.mock('./mailer', () => ({
  sendEmail: vi.fn().mockResolvedValue({ id: 'msg_1' }),
}));

// fake
class InMemoryUserRepo {
  users = new Map();
  create(u) { this.users.set(u.id, u); return u; }
  findById(id) { return this.users.get(id); }
}
```

**Mocking rule of thumb:** mock at the *boundary* (HTTP, DB, queue, time), not deep inside the unit. Mocking internal helpers creates brittle tests that fail on every refactor.

### Time

Always mock time when testing rate limiters, TTLs, scheduling.

```ts
vi.useFakeTimers();
vi.setSystemTime(new Date('2026-01-15T10:00:00Z'));
// run code
vi.advanceTimersByTime(60_000);
vi.useRealTimers();
```

---

## 6. Test-Driven Development (TDD)

The discipline: write the test *first*, watch it fail, write the minimum code to pass, refactor.

```text
RED    — write a failing test
GREEN  — write the minimum code to pass
REFACTOR — clean up while green
```

A worked example:

```ts
// 1. RED
it('formats a duration as 1m 30s', () => {
  expect(formatDuration(90_000)).toBe('1m 30s');
});

// 2. GREEN
export function formatDuration(ms: number): string {
  const s = Math.floor(ms / 1000);
  const m = Math.floor(s / 60);
  return `${m}m ${s % 60}s`;
}

// 3. REFACTOR + more tests
it('handles 0', () => expect(formatDuration(0)).toBe('0m 0s'));
it('handles 1h', () => expect(formatDuration(3_600_000)).toBe('60m 0s'));
```

**TDD's value** is not the tests (you'd write them anyway). It's the *pause*. You think about the API *before* the implementation, which usually means a better design.

**When NOT to use TDD:** exploratory code, throwaway prototypes, UI layout, anything where you don't yet know what "correct" means.

---

## 7. Debugging Strategies

### The scientific method

1. **Reproduce.** Find a test or command that fails 100% of the time.
2. **Hypothesize.** "I think the bug is in `applyDiscount` when amount is 0."
3. **Experiment.** Log, breakpoint, or write a unit test for that hypothesis.
4. **Confirm or refute.** If the experiment shows the bug elsewhere, update your hypothesis.
5. **Fix the root cause**, not the symptom.

### Breakpoints in VS Code

Set a breakpoint in the line, hit F5 (or Run → Start Debugging). The execution pauses, you can inspect variables, the call stack, and step line by line. Better than `console.log` because you can see the state without modifying the code.

### Structured logging

```ts
logger.debug({ userId, amount, plan }, 'computing discount');
// { "level":"debug","userId":1,"amount":100,"plan":"PREMIUM","msg":"computing discount" }
```

When a bug is reported, you can grep for the user's `userId` and see the entire story.

### The bisection method

For "it worked yesterday" bugs: `git bisect` walks you through commits and you mark each as good/bad. After ~5 marks, you have the offending commit.

```bash
git bisect start
git bisect bad
git bisect good v1.0
# test, then git bisect good or bad
# repeat until you find the commit
git bisect reset
```

### The "rubber duck"

Explain the bug out loud, line by line, to an inanimate object (or to Claude). Half the time the act of explaining makes the answer obvious.

---

## 8. Test Coverage & Best Practices

### The 80% myth

"There should be 80% test coverage" is the most repeated lie in software. Coverage is a *tool*, not a goal. A line covered means the line was executed during a test. It does not mean it was *asserted* upon.

```ts
// 100% coverage, 0% value
it('runs', () => {
  applyDiscount(100, 'PREMIUM');
});
```

What coverage *does* tell you:

- **Untested code is suspect.** If line 47 is never hit, you have untested code.
- **Coverage dropping over time** is a signal that PRs are sneaking in untested code.

What to track instead:

- **Mutation score** (StrykerJS) — a tool that mutates your code (flips `>` to `>=`, removes a line) and runs the tests. If tests still pass, the tests are weak. Stryker is slow but the only coverage metric that actually predicts bug-finding ability.
- **The change-failure rate** in CI — what % of releases cause a bug in production? (DORA metrics.)
- **Mean time to recover (MTTR).**

### A pragmatic coverage policy

1. New code must come with tests. Enforce with a CI check on diff coverage (e.g., "lines added in this PR must be 80% covered").
2. No specific overall number. 60% well-tested is better than 95% poorly-tested.
3. Critical paths (auth, payments, billing) must be near 100% with meaningful assertions.

---

## 9. Writing Maintainable Test Suites

### The AAA pattern

```ts
it('does X', () => {
  // ARRANGE — set up the world
  const repo = new InMemoryRepo();
  const svc = new UserService(repo);

  // ACT — do the thing
  const result = await svc.create({ email: 'a@b.c' });

  // ASSERT — check the outcome
  expect(result.email).toBe('a@b.c');
  expect(repo.users).toHaveLength(1);
});
```

Three blocks, three blank lines, no mixing. The pattern that makes tests skimmable.

### Naming

A test name is documentation. Use the form: `methodName_doesWhat_underWhatCondition`.

```ts
it('createUser_throws_when_email_already_exists', ...);
it('applyDiscount_applies_extra_tier_above_$1000', ...);
```

Or, if you prefer the BDD style:

```ts
describe('UserService', () => {
  describe('create', () => {
    it('throws when email already exists', ...);
  });
});
```

Both are fine. The rule: someone reading the test list (not opening the file) should know what each test does.

### Organization

```
src/
  users/
    users.service.ts
    users.service.test.ts       ← co-located
    users.controller.ts
    users.controller.test.ts
e2e/
  auth.e2e.test.ts
  projects.e2e.test.ts
test/
  helpers/
    factories.ts
    db.ts
```

Co-located unit tests are easier to find when you change the file. E2E tests live separately because they spin up the whole app.

### Avoid the test smell checklist

- ❌ Tests that depend on each other (test 3 fails if test 1 didn't run)
- ❌ Tests that use real network calls (real Stripe, real Slack)
- ❌ Tests that take >5 seconds at the unit level
- ❌ Tests that don't assert anything
- ❌ Tests with the same name (silently overwrites)
- ❌ Snapshots you never review (`expect(tree).toMatchSnapshot()` becomes a 4000-line blob)
- ❌ Tests with no `Arrange` — you can't tell what state they started from

---

## 10. Production Testing Workflow

### CI pipeline (GitHub Actions example)

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
        options: --health-cmd pg_isready --health-interval 5s
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci
      - run: npx prisma migrate deploy
      - run: npm test -- --coverage
      - run: npm run lint
      - run: npm run typecheck
```

### Pre-merge gates

Branch protection rules on `main`:

- [x] Require status checks: `test`, `lint`, `typecheck`, `build`
- [x] Require 1+ review
- [x] No force pushes
- [x] No skipping CI

A PR cannot be merged unless CI is green. This is non-negotiable.

### Smoke tests

After deployment, hit the live URL with a known good request. If it 500s, roll back.

```ts
// scripts/smoke.ts
const res = await fetch(`${process.env.URL}/healthz`);
if (!res.ok) {
  await fetch(process.env.SLACK_WEBHOOK, {
    method: 'POST',
    body: JSON.stringify({ text: `smoke failed: ${res.status}` }),
  });
  process.exit(1);
}
```

Run from your deploy job, *after* the deploy, *before* declaring success.

### The test pyramid in CI

```text
PR opened →
  unit + integration (fast feedback)   ~30s
  e2e (slow, only on main)            ~5m
  deploy preview
  smoke test
PR merged →
  deploy to staging
  smoke test
  manual approval
  deploy to production
  smoke test
  rollback if smoke fails
```

---

## Common mistakes

1. **Testing implementation, not behavior.** "We changed the function name, all 50 tests broke." → tests are coupled to internals.
2. **Snapshot tests as a substitute for thought.** A 3000-line snapshot tells you nothing.
3. **Mocks everywhere.** "Mock the DB, mock the queue, mock the date, mock the function" — you're not testing the system, you're testing your mocks.
4. **Coverage as a goal.** Pushing from 80 to 90% by adding tests that don't assert anything.
5. **E2E tests that hit a live service.** "This e2e test failed because Stripe is down." → mock the boundary.
6. **Tests that flake.** A flaky test is worse than no test. Fix or delete on first sight.
7. **Skipping tests in CI.** `--no-test` "just this once" is the first step to never running them.
8. **No test for the bug fix.** The bug will come back.
9. **One giant `it('does everything')`.** When it fails you have no idea what broke.
10. **Tests that require 5 minutes of setup.** If setup is hard, the design is hard. Simplify the code.

---

## Exercise

**Add a test suite to the Mission 4 SaaS.**

1. Set up Vitest. Add `npm test` and `npm run test:watch` scripts.
2. Add a `TEST_DATABASE_URL` and a Docker compose service for the test DB.
3. Write **5 unit tests** for the pricing/discount logic in the SaaS.
4. Write **3 integration tests** for a repository (e.g., the `ProjectsRepository`) using the real test DB.
5. Write **5 Supertest API tests** for the auth + projects endpoints, including:
   - 200 with valid input
   - 400 on invalid input
   - 401 without a token
   - 403 with wrong role
   - 404 on missing resource
6. Mock Stripe in the billing tests. Do not hit the real Stripe.
7. Add a CI workflow on GitHub Actions: install, migrate, test, lint, typecheck.
8. Run coverage. Identify the 3 lowest-covered files. Add tests to bring them above 70%.
9. **Stretch:** add Stryker for mutation testing. Aim for a mutation score above 50%.

Submit: the test output, a coverage report, the CI link, and a 1-paragraph reflection on what surprised you.

---

## AI Industry Note (2026)

In 2026, the single highest-leverage use of an AI coding tool is **test generation** — not feature code. The reason: tests are explicit, formal, and well-suited to a model. A senior engineer using Claude Code or Cursor to write 80% of a test suite is twice as productive as one writing it by hand.

The catch: AI-generated tests are biased toward *the implementation they just saw*. They tend to be tautological — testing the code, not the behavior. The senior move is to **write the test name and the assertion yourself, let the model fill in the arrange and the act.** That gives you behavioral coverage with the model's speed.

In 2026 interviews, "I use AI to write my tests" is a green flag. "I use AI to write my tests *and I review them* " is a stronger one. "I never write tests because AI does it" is a red flag.

---

*Next: Reward Course 04 — AWS Deployment, where we take the tested SaaS and ship it to the cloud.*
