# Live Agentic AI Masterclass

> A 10-session live workshop series. Not a course module — a parallel track. Runs alongside Missions 5 and 6, and the Capstone. Total: ~30 hours of live + ~40 hours of practice.

---

## What this is

A 10-week (or 5-week intensive) live workshop where you learn to work *with* AI coding tools the way a senior engineer works with a junior: by giving it clear specs, reviewing its output, and knowing when to ignore it.

The premise of 2026: the engineers who use AI tools well outproduce the ones who don't by 3–10x. But the productivity gap is not "AI does it for you" — it's "you spend your time on the things only you can do: specs, reviews, architecture, taste, and the parts of the problem that don't have an obvious answer."

This masterclass teaches that skill.

---

## Who this is for

- Working developers in the program who have completed Missions 1–4 (TypeScript, backend, database, SaaS).
- Engineers who want to integrate AI into their daily workflow, not just "use Copilot for autocomplete."
- People who want to be dangerous with Claude Code, OpenCode, Cursor, and Continue.dev.

## Prerequisites

- A **paid LLM API key** (Anthropic, OpenAI, or both). The masterclass is API-driven — you need to see token costs and learn to manage them.
- A **GitHub account** (for actions, code review, and the live demos).
- **Node 22** installed locally.
- **Docker** installed (for sandboxing AI-generated code).
- Comfort with the command line.
- The Mission 1–4 codebases in your GitHub. We will work against real code.

## Schedule (10 weeks, 3 hours/week)

| Week | Topic | Live demo |
|---|---|---|
| 1 | Introduction to Agentic Coding | Demo: Claude Code vs. Copilot, same task |
| 2 | Setting Up Your AI Coding Environment | Live: install 4 tools, configure, benchmark |
| 3 | Context Engineering & AI Memory | Live: AGENT.md evolution on a real repo |
| 4 | AI Development Workflow | Live: spec → draft → review → test on a real feature |
| 5 | Specification-Driven Development (SDD) | Live: write a 1-page spec, AI implements |
| 6 | Advanced Agent Workflows | Live: multi-file refactor with sub-agents |
| 7 | MCP (Model Context Protocol) | Live: connect to a Postgres MCP, query the DB |
| 8 | Engineering Automation | Live: AI fixes lint errors and writes tests |
| 9 | Production Demo | Live: build a small feature end-to-end |
| 10 | Best Practices with AI | Live Q&A; review student submissions |

## Intensive format (5 weeks, 6 hours/week)

Same content, twice a week, with more live coding and less lecture. Used for the summer cohort.

---

## The 10 sessions

### 1. Introduction to Agentic Coding

**What "agentic" actually means.** The difference between "code completion" (Copilot, autocomplete) and "agentic" (Claude Code, OpenCode) is not the model — it's the loop. A code completer predicts the next 20 tokens. An agent runs a loop: read files → think → write → run → check → repeat. That loop is what makes it useful, and what makes it dangerous.

**Topics:**

- The agentic loop (observe → plan → act → verify).
- The difference between Copilot, Cursor, Continue.dev, Claude Code, OpenCode, Aider.
- Where each tool fits in your workflow.
- The cost: tokens, latency, and trust.

**Live demo:** the same feature (a regex email validator with tests) implemented in 4 different tools. Compare speed, correctness, and what the human had to review.

**Takeaway:** "agentic" is not a feature, it's a workflow.

---

### 2. Setting Up Your AI Coding Environment

You will use at least 4 tools. Each has a sweet spot.

| Tool | Sweet spot | Config files |
|---|---|---|
| **Claude Code** | Long-horizon refactors, multi-file, terminal-native | `CLAUDE.md`, `AGENT.md` |
| **OpenCode** | Open-source alternative to Claude Code, BYO model | `AGENT.md` |
| **Cursor** | IDE-integrated, tab-complete + chat + composer | `.cursorrules`, `.cursorignore` |
| **Continue.dev** | VS Code/JetBrains extension, BYO model, open | `.continue/config.json` |

**Topics:**

- Installing each tool.
- API key management (env vars, never in code).
- `.cursorrules` and `AGENT.md` — the "operating manual" for the AI.
- Per-repo config vs. global config.
- Cost observability: track tokens per session.

**Live demo:** install all 4 on a fresh laptop, configure for the Mission 4 SaaS repo, run the same prompt in each, compare results.

**Takeaway:** the tool is 30% of the value. The config is 70%.

---

### 3. Context Engineering & AI Memory

The model is stateless. Every prompt is the only thing it sees. The art of *context engineering* is giving it the right information at the right time.

**Topics:**

- `AGENT.md` vs `CLAUDE.md` — what each is for.
- The prompt cache trick: put stable info at the *start* of the prompt; Claude caches the prefix, you pay 10% of the cost for cached tokens.
- File-based context: `@file`, `@folder`, `/init`.
- The hierarchy: AGENT.md (project) → system prompt (tool) → user message.
- What to *exclude* — a `.cursorignore` / `.aiignore` for sensitive files.
- The "context rot" problem: a 200k-token window is useless if 150k of it is irrelevant.

**Live demo:** start with no context; add an AGENT.md; watch the model's output quality change on the same task.

**Takeaway:** you are not writing prompts, you are curating context.

---

### 4. AI Development Workflow

The human-AI loop. The four phases every feature goes through.

```text
SPEC   →   DRAFT   →   REVIEW   →   TEST
(spec is  →  (AI     →  (human   →  (human
human)     drafts)    reviews)   tests, AI
                                    helps)
```

**Topics:**

- The spec is the most important artifact. If the spec is bad, the code is bad.
- The review is non-negotiable. Every PR, AI-generated or not, gets reviewed.
- The test is your safety net. AI helps write tests; you write the assertions.
- Time budgets: 10 min spec, 2 min draft review, 5 min code review, 5 min test review.

**Live demo:** a real feature from the Mission 4 SaaS, run through the loop, timed.

**Takeaway:** the bottleneck is your judgment, not the model's speed.

---

### 5. Specification-Driven Development (SDD)

The most important session in the masterclass.

The thesis: **write the spec first, AI implements.** The spec is a 1–2 page document with: goal, non-goals, API contract, data model, edge cases, acceptance criteria. You hand it to the AI. The AI produces a PR.

**Topics:**

- Anatomy of a good spec.
- Spec templates (we provide one).
- How to write a spec that the AI can actually implement.
- The 80/20 of "what changed and why" — the AI's job is the 80, yours is the 20.
- When the spec is wrong, you don't blame the AI; you fix the spec.

**Live demo:** the instructor writes a spec for a new feature (project search) in 10 minutes. The AI implements it in 15 minutes. The class reviews.

**Takeaway:** if you can write a good spec, you can ship 5x as fast.

---

### 6. Advanced Agent Workflows

Single-file edits are the easy part. The hard part: a refactor across 30 files.

**Topics:**

- Multi-file edits with Claude Code: `Edit`, `Write`, `MultiEdit`.
- Sub-agents: spawn a sub-agent to explore, summarize, or implement in parallel.
- Parallel work: git worktrees, multiple sessions on different branches.
- The "long task" pattern: split a big task into checkpoints, run, review, continue.
- The "lost in the middle" problem: long sessions drift. Use `/compact` or `/clear`.

**Live demo:** refactor the Mission 4 SaaS to use a new error-handling pattern across 20 files. The AI does the work; the human reviews.

**Takeaway:** the model is fast, the bottleneck is *you reading*.

---

### 7. MCP (Model Context Protocol)

MCP is the standard for AI-to-tool integration. Instead of every tool writing a custom adapter, MCP defines a protocol: tools are servers, models are clients, they speak JSON-RPC.

**Topics:**

- What MCP is, in 5 minutes.
- The MCP server registry (filesystem, git, postgres, github, slack, …).
- How to write your own MCP server (it's a JSON-RPC endpoint).
- Security: an MCP server can read your DB. Treat it like a 3rd-party integration.

**Live demo:** connect Claude Code to a Postgres MCP server, run a migration via the model, query the DB through natural language. Then write a 30-line custom MCP server that exposes a "create project" tool.

**Takeaway:** MCP is the "USB-C for AI tools." Learn it once, use it everywhere.

---

### 8. Engineering Automation

The AI is not just for writing code. It's for running the boring parts of engineering.

**Topics:**

- AI that fixes lint errors (`npm run lint | claude "fix these"`)
- AI that writes tests from a diff (`git diff main | claude "write tests"`)
- AI that triages issues (read the issue, label it, ask for more info)
- AI that does PR review (run on every PR via GitHub Action)
- AI that writes release notes from commits

**Live demo:** a GitHub Action that runs Claude on every PR, posts a review comment. A workflow that, when an issue is opened, asks Claude to label and prioritize it.

**Takeaway:** the boring 80% of engineering is now automatable. The interesting 20% is still yours.

---

### 9. Production Demo — Live Build of a Small Feature

The capstone of the masterclass: a 90-minute live build.

**The feature:** "Add a 'duplicate project' button to the Mission 4 SaaS, with confirmation modal, server endpoint, and tests."

**The flow:**

1. 5 min: spec
2. 30 min: AI implements (frontend + backend + tests)
3. 15 min: instructor reviews live, explains what's good and what's bad
4. 10 min: AI fixes the issues
5. 5 min: deploy to staging
6. 5 min: smoke test
7. 20 min: Q&A

**Takeaway:** you have seen the entire loop, end-to-end, in real time.

---

### 10. Engineering Best Practices with AI

The session that prevents you from becoming a "vibe coder."

**Topics:**

- Security review with AI: prompt-injection, secrets in code, OWASP top 10.
- Code review with AI: what to look for in AI-generated code (subtle bugs, over-engineering, security holes).
- When to NOT use AI:
  - Anything with a personal-data risk you can't audit.
  - Anything you don't understand.
  - Production credentials, even briefly.
  - Decisions that are ethical or legal, not technical.
- The "AI as a junior" mental model: trust but verify.
- Cost management: how to keep your monthly AI bill under $50.

**Live Q&A:** students submit their worst AI-generated code, we review it together.

**Takeaway:** AI is a tool. Tools can be used well or poorly. The engineer is responsible for the result.

---

## Graduation requirement

To graduate from the masterclass, you must:

1. **Attend** at least 8 of the 10 sessions (live or recording within 7 days).
2. **Complete the spec-driven exercise** in session 5: write a spec, have the AI implement it, ship a PR.
3. **Complete the automation exercise** in session 8: ship a GitHub Action that does something useful in your repo.
4. **Ship one of the assignments using AI assistance.** Pick one of:
   - The Mission 5 deployment to AWS (Reward 04)
   - A new feature in your Capstone
   - A 3rd-party library integration
   - A refactor of an existing module
5. **Submit a 5-minute Loom** walking through: what you used AI for, what you didn't, what went wrong, and what you'd do differently.

The graduation is the proof. You can show a hiring manager: "I shipped this with AI, here's the spec, here's the PR, here's the Loom."

---

## What you will leave with

- A working `.cursorrules` / `AGENT.md` for your repos.
- A personal workflow you can apply on day 1 at your next job.
- A vocabulary for talking about agentic coding (no more "I just use Copilot").
- The muscle memory of the spec → draft → review → test loop.
- A graduate badge for LinkedIn that means something.
- 3–5 PRs in your GitHub that were built the new way, with the spec, the AI output, and your review.

---

## Anti-goals (what this masterclass is NOT)

- **Not a "vibe coding" course.** You will write specs. You will review code. You will not paste a half-formed idea into a chat and ship the result.
- **Not a "10x engineer in 30 days" promise.** The tools change. The discipline (spec, review, test) does not.
- **Not a replacement for engineering fundamentals.** AI does not absolve you of knowing how the system works. If you can't review the code, you can't ship it.
- **Not model worship.** Different models are good at different things. We will use multiple.

---

## Tools we will use (the only ones you need to pay for)

- **Anthropic API key** (Claude Sonnet 4.5 / Opus 4.5) — primary workhorse. ~$20/month for the masterclass if you use it carefully.
- **OpenAI API key** (optional) — for comparison. ~$20/month.
- **GitHub** — free tier.
- **All the IDE tools** (Cursor, Continue.dev) — free or have free tiers.

**Total cost of the masterclass: $0–$60**, depending on how much you use the models.

---

## Resources

- [Claude Code docs](https://docs.claude.com/claude-code)
- [MCP specification](https://modelcontextprotocol.io)
- [Specification-Driven Development — a working pattern](https://martinfowler.com/articles/exploring-gen-ai.html) (Martin Fowler, 2024)
- [AGENT.md template](https://github.com/buildermethods/agent-md-template) (we provide our own)
- The class Discord for async questions.

---

## Faculty

- **Lead instructor:** a senior engineer who has shipped AI-assisted features in production for 2+ years. Not an AI researcher; a practitioner.
- **TAs:** 2–3 engineers who use these tools daily, available on Discord for async help.

---

*Next: the Capstone. The masterclass pays off here — you will use spec-driven AI workflows to ship the multi-tenant AI SaaS in 4 weeks instead of 8.*
