# 00 — Orientation: How to Think Like a Senior Engineer in 2026

> Before we write a single line of TypeScript, let's align on *how* you are going to learn for the next 6 months.

---

## Why this lesson exists

Most junior engineers fail not because they don't know syntax — they fail because:

1. They learn **tools**, not **mental models**.
2. They write code that works on **their machine**, not in production.
3. They don't know how to **debug** when things break.
4. They treat AI coding assistants as magic, not as a **force multiplier on existing skill**.

This short orientation is the operating system we are going to install in your head before we start Mission 1.

---

## 1. The Senior Engineer's Loop

Every senior engineer — including the one writing this — does the same five things in a loop:

```
Observe → Model → Implement → Verify → Operate
```

| Stage | What it means | What tool helps |
|-------|---------------|-----------------|
| **Observe** | Understand the real problem. Read tickets, ask "why" 5 times, draw the data flow. | Issue tracker, design doc |
| **Model** | Decide on data shape, contracts, and boundaries *before* coding. | ER diagrams, type signatures, OpenAPI |
| **Implement** | Write the smallest code that satisfies the model. | TypeScript, Drizzle, Next.js, AI assistant |
| **Verify** | Prove it works: types compile, tests pass, the user can do the task. | `tsc`, Vitest, Playwright, manual smoke |
| **Operate** | Watch it in production. Log, alert, fix, iterate. | NGINX logs, Sentry, health checks |

If you skip **Model**, you rewrite code.
If you skip **Verify**, you ship bugs.
If you skip **Operate**, you get paged at 3am.

**AI Industry Note (2026):** The most valuable engineers in 2026 are the ones who can stay in **Observe** and **Model** longer. AI assistants make **Implement** almost free, so the human edge moves up the stack into "deciding what to build" and "proving it works." That is exactly where this curriculum spends its time.

---

## 2. How to read code

When you see a new file, read it in this order:

1. **The imports / type signatures** — what does this module *depend on* and what does it *expose*?
2. **The exported function/class** — what is the public contract?
3. **The body, top to bottom** — implementation details last.

A junior reads line 1 → line 2 → line 3. A senior reads the **shape** first. AI assistants are particularly bad at this; they read left-to-right too. You can do better.

---

## 3. The "Three Question" debugging rule

When your code doesn't work, ask **exactly these three questions**, in order:

1. **What did I expect to happen?** (Be specific. "It works" is not an answer.)
2. **What actually happened?** (The exact error, the exact wrong value, the exact log line.)
3. **What is the smallest difference between those two?** (That is your bug.)

If you can't answer #1, you don't have a bug — you have a *specification problem*. Write the spec down before you keep debugging.

---

## 4. How to use AI assistants in this course

You are going to be given access to an **Agentic AI Masterclass** that teaches Claude Code and OpenCode. Here is the preview:

- **Never paste a vague prompt.** "Write the auth code" is useless. "Write a JWT-based registration endpoint in Express + TypeScript that takes `{ email, password }`, hashes with bcrypt, returns `{ accessToken, user }`" is a 10x prompt.
- **Always own the type signatures.** Let AI fill in the body, but you write the interface first.
- **Always read the diff.** If you can't explain every changed line, you didn't learn anything.
- **Always run the code.** AI lies. Your tests don't.

We will revisit AI-assisted workflows in every mission. By the end, you will treat your AI assistant like a very fast junior: useful for grunt work, dangerous unsupervised.

---

## 5. The 6-month contract with yourself

| Week | Mission | Deliverable you ship |
|------|---------|----------------------|
| 1–3 | TypeScript | Assignment 1: type-safe utility lib + small OOP domain |
| 4–6 | Scalable backends | Assignment 2: Express + PostgreSQL layered API |
| 7–9 | Databases | Assignment 3: 3NF production schema from a brief |
| 10–15 | SaaS product | Assignment 4 + 5: full multi-tenant SaaS (backend + frontend) |
| 16–17 | Production | Dockerized, NGINX-fronted, CI/CD deployed to a VPS |
| 18–23 | AI systems | RAG over your own docs + LangGraph agent, integrated into the SaaS |
| 24+ | Capstone | **Advanced Full-Stack Multi-Tenant AI SaaS Product** |

Each assignment is graded. Each is portfolio-worthy. Each is something you can show a hiring manager and say "I built this end to end."

---

## 6. Tools you will install this week

- **Node.js 22 LTS** — the runtime.
- **pnpm** — fast, disk-efficient package manager.
- **VS Code** — with the official TypeScript, ESLint, Prettier, and Drizzle extensions.
- **Git + GitHub** — every project is version-controlled from day 1.
- **Docker Desktop** — used in Mission 5.
- **PostgreSQL 16** (locally and in Docker) — your primary database.
- **A paid LLM API key** (OpenAI / Anthropic / OpenRouter) — used in Mission 6.

**AI Industry Note (2026):** Yes, the AI subscription is a real cost. Treat it like a textbook. A $20/month Claude or ChatGPT plan, used properly, will return 10x on the cost in productivity. The skill of *prompting* is now table-stakes — and the live masterclass is where we sharpen it.

---

## 7. Your first exercise (do this today)

Open a terminal. Run:

```bash
node --version
pnpm --version
git --version
docker --version
psql --version
```

If any of these fail, fix the install *before* Mission 1 starts. Then create a GitHub repo called `next-level-journal` and write a one-paragraph entry: **"What kind of engineer do I want to be in 6 months?"** Commit it. That commit is the first entry in a journal you will keep for the whole course.

---

## What you should take away

- You are learning a **loop**, not a list of tools.
- **Modeling** and **verification** are where the human edge lives in 2026.
- AI assistants are a force multiplier on existing skill, not a replacement for it.
- Show up daily. The 6-month contract is real.

When you are ready, open `mission-01-typescript/module-01-typescript-foundations.md`. The first line of code is waiting.
