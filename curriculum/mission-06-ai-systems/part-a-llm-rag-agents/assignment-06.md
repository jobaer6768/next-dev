# Assignment 6: Build a Production AI Application

## Overview

This is the capstone for Mission 6. You will build a **production-grade AI application** on top of the SaaS from Mission 4, integrating everything you have learned in Modules 50–58. The deliverable is a working app that another developer can clone, run, and use — not a notebook, not a prototype, a *product*.

You will be graded on functionality, code quality, evaluation, cost analysis, and the three conceptual sessions at the end of this document. Read the whole assignment before you start coding.

**Due:** End of Mission 6.
**Weight:** 30% of Mission 6 grade.
**Group size:** 1–3 developers. Solo is fine; teams of 3 must show a clear split of ownership in the PR history.

---

## What you are building

A **"Document Intelligence"** product for your SaaS workspace. It is the union of:

1. A **PDF knowledge base** — users upload documents, the system chunks and embeds them, and the user can chat with the corpus (Module 50).
2. A **research agent** that can search the web, run calculations, and call internal APIs (Modules 51–52).
3. A **streaming chat interface** in the browser (Module 57).
4. A **dashboard** for managing documents, viewing agent runs, and tracking usage (Module 58).
5. The **service-to-service plumbing** that makes it survive a Friday deploy (Modules 54–56).

The reference stack:

- **Frontend:** Next.js 15 (App Router), React 19, Tailwind, shadcn/ui, `ai-sdk`, TanStack Query, Recharts.
- **Backend:** Next.js Route Handlers + three Node services (`agent`, `kb`, `billing`) deployed independently.
- **Data:** Postgres (Supabase or Neon), pgvector, Redis, BullMQ.
- **AI:** LangChain v0.3+, LangGraph, OpenAI (`gpt-4o-mini`, `text-embedding-3-small`) and/or Anthropic (`claude-3.5-sonnet`).
- **Observability:** LangSmith, OpenTelemetry → Jaeger, pino logs.

You may substitute equivalent tools (Anthropic for OpenAI, Turbopuffer for pgvector, etc.) but document the choice.

---

## Requirements

### 1. LangChain pipeline (Module 50)

- A `POST /api/kb/upload` endpoint that accepts a PDF, chunks it, embeds it, and stores it in pgvector.
- A `POST /api/chat` endpoint that performs RAG: retrieve top-k=20, optionally re-rank, build the prompt, stream the response.
- Citations: every answer includes a `citations` array with `{ source, chunkIndex, score }`.
- Refusal: when the highest retrieved score < 0.65, the system returns the exact string "I don't have that information in the provided documents."

### 2. LangGraph agent with tool calling (Modules 51–52)

- A `POST /api/agent/run` endpoint that invokes a LangGraph agent with at least three tools: `web_search`, `calculator`, and `kb_search` (your own RAG).
- The graph has typed state, conditional routing, a step cap of 8, and a token cap of 50k per run.
- A `POST /api/agent/approve` endpoint that resumes a paused run via `interrupt` (HILT for the `send_email` tool).
- One multi-agent pipeline: a router that classifies the question, dispatches to a specialist, and synthesizes the answer.

### 3. RAG over user-uploaded PDFs (Module 50, again)

- Documents are scoped to the workspace. A test must prove that user A in workspace 1 cannot retrieve user B's chunks in workspace 2.
- Re-ranking via Cohere (or a cross-encoder of your choice) on the top-20 candidates.
- Embedding cache: re-uploading the same file is a no-op for the embedding step.

### 4. pgvector

- HNSW index on the embedding column.
- Schema migration is versioned (use `drizzle-kit` or `prisma migrate`).
- A `kb_chunks_workspace_idx` index for tenant-filtered queries.

### 5. Streaming chat interface (Module 57)

- Uses the `ai` SDK's `useChat` hook.
- Renders tokens as they arrive (latency from "send" to "first token" < 1.5s on the reference machine).
- Stop, retry, and send buttons all work.
- Citation chips are clickable; clicking opens a side panel with the source PDF preview.

### 6. Evaluation notes (Module 50 + Module 56)

- A `pnpm run eval` script that runs RAGAS on a 20-question test set you author.
- The script reports `faithfulness`, `answer_relevance`, `context_recall`, `context_precision`.
- Results are written to `eval/results.json` and committed (or to a CI artifact).
- Include a 1-page writeup (`docs/eval.md`) explaining how you would improve each metric.

### 7. Cost analysis (Module 56 + Module 58)

- A `pnpm run cost-report` script that queries the `llm_call_log` table and produces a per-workspace cost summary for the last 30 days.
- A `docs/cost.md` with a one-page table: input token cost, output token cost, embedding cost, total per 1,000 questions.
- A `docs/cost-optimization.md` listing 5 concrete optimizations you implemented and their estimated savings.

### 8. Polish (Module 58)

- The dashboard's KB tab, Agent Runs tab, and Usage tab all work.
- The chat has loading, error, retry, and empty states.
- Keyboard shortcuts: `Cmd+K` for command palette, `Cmd+/` to focus composer, `Esc` to stop.
- The layout is responsive to 360px width.
- Lighthouse accessibility score ≥ 90.

---

## Submission

1. A GitHub repo (or your bootcamp's Git server) with the code.
2. A 5-minute screen recording walking through: upload a PDF → chat with it → run a research agent that uses web search + KB + calculator → approve the HITL step → see the trace in the dashboard → check the usage chart.
3. A `README.md` with: architecture diagram, setup instructions, env vars, deployment guide.
4. The `docs/eval.md`, `docs/cost.md`, and `docs/cost-optimization.md` files.
5. A PR description with: what's done, what's not, what you would do next.

---

## Grading rubric (out of 100)

| Category | Weight | What we're looking for |
|---|---|---|
| **Functionality** | 30 | All required endpoints work. Streaming, RAG, agent, HITL, dashboard all demonstrable. |
| **Code quality** | 15 | TypeScript strict mode, Zod at boundaries, no `any` without a comment, sensible file structure, tests for the hard parts. |
| **Evaluation** | 15 | A 20-question RAGAS run with results; a 1-page analysis identifying the weakest metric and how to fix it. |
| **Cost analysis** | 10 | A real cost report from the `llm_call_log`; a credible optimization list. |
| **Production hygiene** | 15 | Service-to-service auth, timeouts, retries, circuit breaker, health checks, per-workspace rate limits, per-workspace token budget. |
| **UX polish** | 10 | Streaming UI works smoothly; citations are clickable; dashboard is responsive; keyboard shortcuts work; accessibility score ≥ 90. |
| **Documentation** | 5 | README is honest about what's done and what's not; architecture diagram is accurate; the three conceptual-session writeups are thoughtful. |

Deductions:
- **−20** for a critical security issue (e.g., tenant data leak, no auth on a destructive endpoint).
- **−15** for an unbounded cost path (e.g., no token cap, no rate limit on a per-user basis).
- **−10** for hallucinated citations (the LLM cites a chunk that does not exist in the top-k).
- **−5** for each missing required deliverable.

---

## Three conceptual sessions

These are not coding deliverables. They are 1-page writeups (`docs/session-1.md`, `docs/session-2.md`, `docs/session-3.md`) that show you can think about AI the way a senior engineer in 2026 thinks about it. Each is worth 5 points; we read them.

### Session 1: Prompt engineering that survives production

> A prompt that works in a notebook is not a prompt that survives 10,000 users.

Cover:

- **Versioning.** How do you track prompt changes? Git? A `prompts/` directory with semantic-versioned YAML? A prompt-management system (LangSmith, PromptLayer, Helicone)?
- **Evaluation.** How do you know a new prompt is *better* and not just *different*? A/B traffic split? An offline eval harness? A/B in shadow mode?
- **A/B testing.** How do you route 10% of traffic to a new prompt and measure the lift on faithfulness and user satisfaction?
- **Failure modes.** What does "stale prompt" look like in production? How do you detect a regression before customers do?

**Guiding question:** If your prompt changes next week, how does the team know it got better?

### Session 2: RAG in the real world — what breaks at scale

> A RAG demo works on five documents. A RAG product breaks on five thousand.

Cover four failure modes from your own experience building this assignment:

- **Chunking.** When does fixed-size chunking fail? How do you handle a 200-page legal document where a clause spans three pages? Describe a concrete strategy (e.g., "section-aware chunking with a max chunk size of 1500 chars and a 200-char overlap aligned on sentence boundaries").
- **Freshness.** When a user updates a document, how do you invalidate the embeddings? Delete-and-re-embed? Incremental re-embedding? Versioned embeddings? What happens if a citation points to a chunk that no longer exists?
- **Multilingual.** How does your system behave when the user asks in Spanish and the corpus is in English? Do you embed multilingual? Do you translate before retrieval? Cite the embeddings model you chose (`text-embedding-3-small` is English-only; `multilingual-e5-large` is not).
- **Citations at scale.** When the corpus grows, naive "show all 5 citations" overwhelms the UI. How do you pick the *best* 3 citations to show? How do you handle the case where the LLM cites a chunk the retriever did not return?

**Guiding question:** Pick the failure mode that surprised you most. What did you change in your code to handle it?

### Session 3: Designing agents that don't loop forever

> An agent in a loop is not an AI feature. It is a billing event.

Cover:

- **Max-steps.** Why is a hard cap (e.g., 8 steps) better than a soft one? Where in the graph is the cap enforced? What happens to state when the cap is hit?
- **Timeouts.** A `timeout: 60_000` on the worker, an `AbortController` on every tool call, a request deadline on every fetch. Why all three?
- **Cost caps.** Per-workspace and per-day. Where in the code is the budget checked? What does the user see when they hit it?
- **Human-in-the-loop.** Which tool calls *must* require approval? (`send_email`, `delete_record`, `charge_card`.) Which can be auto-approved? (`web_search`, `calculator`.) Why?
- **Observability.** How do you know an agent is in a loop *before* the bill arrives? What metric do you alert on? (Token spend per minute, number of tool calls per run, latency p99.)

**Guiding question:** Show the moment in your code where an agent's run is hard-stopped. What does the user see? What does the dashboard show? What does the next run inherit?

---

## Suggested schedule

| Day | Focus | Deliverable |
|---|---|---|
| 1 | Repo setup, contracts package, three services, gateway | `pnpm dev` brings up all four |
| 2 | RAG: load → split → embed → store → retrieve | First end-to-end PDF chat works |
| 3 | LangGraph agent with three tools | Agent runs and returns a final answer |
| 4 | Streaming UI + dashboard skeleton | Tokens appear in the browser; tabs render |
| 5 | Multi-agent + HILT + memory | Router/specialist/synthesizer works; interrupt/resume works |
| 6 | BullMQ workers + caching + cost caps | Long jobs run in the background; quotas enforced |
| 7 | Evaluation, cost report, polish, docs | RAGAS results, cost.md, three session writeups |
| 8 | Submission prep | Recording, README, PR description |

---

## Stretch goals (5 bonus points each, capped at 10)

- **Voice in / voice out.** Whisper.cpp in the browser for input, OpenAI/Anthropic TTS for output. A mic button next to the composer.
- **Multimodal RAG.** Embed images with CLIP, retrieve text + images together, render inline.
- **Cross-workspace search.** A user with the right permission can ask a question across multiple workspaces; citations show which workspace each came from.
- **Agent-as-API.** A second customer can call your agent over an OAuth2-authenticated public API with a per-API-key rate limit.
- **Webhooks.** When a long job completes, the worker POSTs to a user-registered URL signed with HMAC.

---

## What "production" means in this assignment

You are not asked to deploy to AWS. You are asked to build something that *could* be deployed to AWS with a config change. The difference:

- **Yes:** typed contracts, timeouts on every fetch, retries on retryable errors only, a circuit breaker, per-workspace rate limits, per-workspace token budgets, health and readiness endpoints, OpenTelemetry traces, structured logs with correlation IDs, a Postgres checkpointer for the agent.
- **No:** Kubernetes manifests, Terraform, a CI pipeline (one is welcome, not required), multi-region failover, on-call rotation.

The line is "could ship Monday" not "shipped Monday."

---

## AI Industry Note (2026)

The assignment you have in front of you is a representative week of work for a senior AI engineer in 2026. The tooling shifts; the disciplines do not.

- **The stack is converging.** LangChain + LangGraph + the `ai` SDK + Postgres + Redis is the default 2026 stack for a TypeScript team. The alternatives (`llamaindex`, `haystack`, raw OpenAI) are losing ground in the JS/TS world.
- **Models are commoditizing.** `gpt-4o-mini` and `claude-3.5-sonnet` are the workhorses; the frontier is irrelevant to your product. The value is in the *system around the model*: the retrieval, the eval loop, the cost caps, the user trust.
- **The differentiator is the eval loop.** Teams that ship a continuous eval pipeline (RAGAS + LLM-as-judge) ship better products, faster. Teams that don't ship vibes.
- **Multi-agent is the default for non-trivial work.** The PM pattern (planner → executor → reviewer) is the LLM equivalent of a CI pipeline. Single-agent tool-calling is a thing of the past for any product that needs reliability.
- **Per-tenant everything.** Per-tenant rate limits, per-tenant token budgets, per-tenant memory, per-tenant cost dashboards. The era of "one global limit" is over; the era of "treat every workspace as a customer" is here.
- **The senior engineer is the one who says no.** No, we are not adding another tool to the agent. No, we are not skipping the eval. No, we are not deploying on Friday. The senior's job is the discipline; the LLM's job is the work.

Build it. Show it. Ship it.
