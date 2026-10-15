# Module 56: Streaming, Background Workers & Caching

## Why this module matters

The user's first impression of your AI feature is latency. The second is cost. The third is whether it works at 2am when one customer is uploading a 4,000-page manual and another is running a batch job. AI features are *slow* (LLM latency), *expensive* (per-token cost), and *bursty* (PDF uploads, batch summarizations). The standard request-response pattern that works for CRUD is the wrong tool.

This module gives you the three primitives that make AI products work in production: **streaming** for the user-perceived latency, **background workers** for long jobs, and **caching** for cost. We will wire all three into the Mission 4 SaaS.

## Learning objectives

1. Stream LLM tokens from server to browser over Server-Sent Events.
2. Build a BullMQ + Redis job queue for long-running AI jobs.
3. Implement status polling and webhook notifications for asynchronous work.
4. Cache LLM responses, embeddings, and idempotency keys in Redis.
5. Apply per-workspace rate limits and token budgets.
6. Sketch the production AI pattern: AI gateway, inference layer, eval layer.

## Conceptual model

The three primitives, mapped to the user journey:

```
[User clicks "Summarize 4000-page PDF"]
   ↓
POST /jobs/summarize → 202 Accepted + jobId    (background worker, not request-response)
   ↓
Worker chunks → embeds → calls LLM 100x → emits progress events
   ↓
Browser polls /jobs/{id} every 1.5s            (status, partial text)
   ↓
Final result cached in Redis                   (next request is 5ms instead of 8 minutes)
```

The opposite, naive pattern (one HTTP request that takes 8 minutes) is a timeout, a frustrated user, and a dead browser tab.

### 1. Streaming from server to browser

Server side (Next.js Route Handler):

```ts
// src/app/api/chat/stream/route.ts
import { ChatOpenAI } from "@langchain/openai";
import { ragPrompt, buildContext } from "@/lib/rag/prompt";
import { retrieve } from "@/lib/rag/retrieve";

export async function POST(req: Request) {
  const { question, workspaceId } = await req.json();
  const chunks = await retrieve(question, workspaceId);

  const llm = new ChatOpenAI({ model: "gpt-4o-mini", streaming: true, temperature: 0 });
  const stream = await ragPrompt.pipe(llm).stream({
    question,
    context: buildContext(chunks),
  });

  // Server-Sent Events: the simplest streaming wire format the browser supports natively.
  const encoder = new TextEncoder();
  const body = new ReadableStream({
    async start(controller) {
      try {
        for await (const token of stream) {
          controller.enqueue(encoder.encode(`data: ${JSON.stringify({ token })}\n\n`));
        }
        controller.enqueue(encoder.encode(`data: ${JSON.stringify({ done: true })}\n\n`));
      } catch (e) {
        controller.enqueue(encoder.encode(`data: ${JSON.stringify({ error: (e as Error).message })}\n\n`));
      } finally {
        controller.close();
      }
    },
  });

  return new Response(body, {
    headers: {
      "content-type": "text/event-stream",
      "cache-control": "no-cache",
      "connection": "keep-alive",
    },
  });
}
```

Client side (the `useChat` hook from the `ai` SDK does this for you, but here is the manual version so you understand the wire):

```tsx
// src/components/chat/manual-stream.tsx
export function useStreamingAnswer() {
  const [text, setText] = useState("");
  const [done, setDone] = useState(false);

  async function ask(question: string) {
    setText("");
    setDone(false);
    const res = await fetch("/api/chat/stream", {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ question }),
    });
    const reader = res.body!.getReader();
    const dec = new TextDecoder();
    let buf = "";
    while (true) {
      const { value, done } = await reader.read();
      if (done) break;
      buf += dec.decode(value, { stream: true });
      const events = buf.split("\n\n");
      buf = events.pop()!;
      for (const e of events) {
        const data = e.replace(/^data:\s*/, "");
        const parsed = JSON.parse(data);
        if (parsed.token) setText((t) => t + parsed.token);
        if (parsed.done) setDone(true);
      }
    }
  }

  return { text, done, ask };
}
```

The `ai` SDK's `useChat` does the same thing with one hook. Use that in production; understand this for debugging.

### 2. Background workers with BullMQ

```ts
// src/lib/queue/summarize.ts
import { Queue, Worker, QueueEvents } from "bullmq";
import { Redis } from "ioredis";
import { ChatOpenAI } from "@langchain/openai";
import { PGVectorStore } from "@langchain/community/vectorstores/pgvector";

const conn = new Redis(process.env.REDIS_URL!, { maxRetries: null });

export type SummarizeJob = {
  workspaceId: string;
  userId: string;
  documentId: string;
  pdfPath: string;
};

export const summarizeQueue = new Queue<SummarizeJob>("summarize", { connection: conn });

new Worker<SummarizeJob>(
  "summarize",
  async (job) => {
    const { workspaceId, documentId, pdfPath } = job.data;

    await job.updateProgress(5);
    const chunks = await loadAndSplit(pdfPath, ".pdf");
    await job.updateProgress(20);

    await vectorStore.addDocuments(chunks);
    await job.updateProgress(40);

    const llm = new ChatOpenAI({ model: "gpt-4o-mini", streaming: true });
    let summary = "";
    const total = chunks.length;
    for (let i = 0; i < total; i++) {
      const partial = await llm.invoke(`Summarize this section:\n\n${chunks[i].pageContent}`);
      summary += `\n## Section ${i + 1}\n${partial.content}\n`;
      await job.updateProgress(40 + Math.round(((i + 1) / total) * 55));
    }

    await db.query(
      "INSERT INTO kb_summaries (document_id, workspace_id, summary) VALUES ($1, $2, $3)",
      [documentId, workspaceId, summary]
    );
    await job.updateProgress(100);
    return { documentId, length: summary.length };
  },
  {
    connection: conn,
    concurrency: 2,                       // Don't melt the LLM budget
    timeout: 600_000,                    // 10 minutes max
    settings: { backoffStrategy: require("bullmq").defaultBackoff },
  }
);

new QueueEvents("summarize", { connection: conn });
```

The three settings that matter: **concurrency** (don't run 50 jobs at once), **timeout** (kill stuck jobs), **attempts** (BullMQ retries with exponential backoff automatically).

Status polling from the dashboard:

```ts
// src/app/api/jobs/[id]/route.ts
import { summarizeQueue } from "@/lib/queue/summarize";

export async function GET(_req: Request, { params }: { params: { id: string } }) {
  const job = await summarizeQueue.getJob(params.id);
  if (!job) return Response.json({ state: "unknown" }, { status: 404 });
  return Response.json({
    state: await job.getState(),
    progress: job.progress,
    failedReason: job.failedReason,
    returnvalue: job.returnvalue,
  });
}
```

For longer jobs, **webhooks** beat polling. The worker `POST`s `{ jobId, state, result }` to a callback URL the client registered.

### 3. Redis caching

Three cache layers, in increasing specificity:

```ts
// src/lib/cache/redis.ts
import { Redis } from "ioredis";
export const redis = new Redis(process.env.REDIS_URL!);
export const CACHE_TTL = {
  embedding: 60 * 60 * 24 * 7,    // 1 week — embeddings are content-addressable
  llmResponse: 60 * 60 * 24,      // 1 day — answers can become stale
  idempotency: 60 * 60 * 24,      // 1 day — Stripe-style
};
```

#### Embedding cache

```ts
// src/lib/rag/embeddings.ts
import { RedisCache } from "@langchain/core/caches";
import { redis } from "@/lib/cache/redis";

export const embeddings = new OpenAIEmbeddings({
  model: "text-embedding-3-small",
  cache: new RedisCache(redis),
});
```

Re-embedding the same PDF chunk is the most common waste. `OpenAIEmbeddings` will check the cache automatically when you pass a `cache`.

#### LLM response cache

```ts
// src/lib/cache/llm.ts
import { createHash } from "node:crypto";
import { redis, CACHE_TTL } from "./redis";

export async function cachedLLM(promptKey: string, args: unknown, run: () => Promise<string>) {
  const key = `llm:${createHash("sha256").update(promptKey + JSON.stringify(args)).digest("hex")}`;
  const cached = await redis.get(key);
  if (cached) return { text: cached, cached: true };
  const text = await run();
  await redis.set(key, text, "EX", CACHE_TTL.llmResponse);
  return { text, cached: false };
}
```

Use sparingly: identical prompts to an LLM often mean a user typed the same thing twice, which *is* a cache hit, but identical questions across users are rare and a privacy risk. Cache per-user with a `userId` prefix.

#### Idempotency keys

```ts
// src/lib/cache/idempotency.ts
import { redis, CACHE_TTL } from "./redis";

export async function idempotent<T>(key: string, run: () => Promise<T>): Promise<T> {
  const cached = await redis.get(`idem:${key}`);
  if (cached) return JSON.parse(cached) as T;

  const result = await run();
  await redis.set(`idem:${key}`, JSON.stringify(result), "EX", CACHE_TTL.idempotency);
  return result;
}
```

A retry of a `POST /api/email/send` with the same `Idempotency-Key` returns the same response without sending a second email.

### 4. Per-workspace rate limiting & token budgets

```ts
// src/lib/billing/usage.ts
import { redis } from "@/lib/cache/redis";
import { db } from "@/lib/db";

export const usageTracker = {
  async record(workspaceId: string, usage: { total_tokens: number }) {
    const month = new Date().toISOString().slice(0, 7); // "2026-10"
    await db.query(
      `INSERT INTO usage (workspace_id, month, tokens) VALUES ($1, $2, $3)
       ON CONFLICT (workspace_id, month) DO UPDATE SET tokens = usage.tokens + $3`,
      [workspaceId, month, usage.total_tokens]
    );
  },

  async tokensThisMonth(workspaceId: string) {
    const month = new Date().toISOString().slice(0, 7);
    const row = await db.query(
      "SELECT tokens FROM usage WHERE workspace_id = $1 AND month = $2",
      [workspaceId, month]
    );
    return row.rows[0]?.tokens ?? 0;
  },

  async assertBudget(workspaceId: string, planLimit: number) {
    const used = await this.tokensThisMonth(workspaceId);
    if (used >= planLimit) throw new Error("WORKSPACE_OVER_BUDGET");
    return { used, remaining: planLimit - used };
  },
};
```

Enforce at every LLM call:

```ts
async function callLLM(workspaceId: string, args: ChatCompletionArgs) {
  await usageTracker.assertBudget(workspaceId, planLimitFor(workspaceId));
  const result = await openai.chat.completions.create(args);
  await usageTracker.record(workspaceId, { total_tokens: result.usage!.total_tokens });
  return result;
}
```

The 2026 lesson: **budgets are code, not policy docs**. If the code does not refuse the call, the budget does not exist.

### 5. The production AI pattern

The shape every 2026 AI product has settled on:

```
                 ┌────────────────────────────────────────────┐
                 │              API GATEWAY                    │
                 │  (Module 55: auth, rate limit, identity)    │
                 └───────────────────┬────────────────────────┘
                                     │
                 ┌───────────────────▼────────────────────────┐
                 │              AI GATEWAY                     │
                 │  prompt log, model fallback, cost tracking  │
                 │  (Portkey / Helicone / Cloudflare AI GW)     │
                 └───────────────────┬────────────────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
   ┌──────────▼─────────┐ ┌──────────▼─────────┐ ┌──────────▼─────────┐
   │  INFERENCE LAYER   │ │   QUEUE + WORKER   │ │    EVAL LAYER      │
   │  (streaming, RAG,  │ │ (BullMQ, long jobs,│ │ (RAGAS, DeepEval, │
   │   agent runner)    │ │  webhooks)         │ │  LangSmith)       │
   └────────────────────┘ └────────────────────┘ └────────────────────┘
```

Three layers, three responsibilities:

- **AI gateway** owns the prompt log, model fallback ("if OpenAI 503s, try Anthropic"), per-tenant cost. If you do nothing else, ship this.
- **Inference layer** is the synchronous LLM calls. Streaming, RAG, tool calling. This is Modules 50–52.
- **Eval layer** runs continuously: every prompt and answer is graded by an LLM-as-judge; alerts fire when quality drops. This is the bit most teams skip and regret.

## Common mistakes

1. **Synchronous long jobs.** A `POST /summarize` that blocks for 8 minutes *will* time out at the load balancer. Enqueue it.
2. **No progress events.** A 10-minute job with no progress bar is a user who closed the tab. Emit progress every iteration.
3. **Global rate limit only.** A single bad actor can starve every other customer. Per-workspace *and* per-IP.
4. **Caching across tenants.** An LLM response cache scoped globally leaks answers across customers. Always include `workspaceId` and `userId` in the cache key.
5. **No timeout on the worker.** A job that hangs forever will pile up. `timeout: 600_000` is non-negotiable.
6. **Streaming that buffers.** `await llm.invoke(...)` defeats the purpose. Use `.stream()`.
7. **Forgetting OpenTelemetry on the queue.** A job that ran 8 minutes looks like nothing in your dashboards. Instrument `BullMQ` events the same way you instrument HTTP requests.

## Exercise

Add streaming chat + a background PDF summarization job to the SaaS.

**Acceptance criteria**

1. The chat endpoint streams tokens to the browser via SSE; the UI shows them as they arrive.
2. A user can submit a PDF for summarization. They receive a `jobId` immediately and can poll `/api/jobs/:id` for progress.
3. The worker caps at concurrency 2, kills any job that runs > 10 minutes, and retries transient failures 3 times.
4. Embedding calls are content-cached in Redis (uploading the same file twice embeds once).
5. The workspace has a monthly token budget of 500k tokens; exceeding it returns `429 WORKSPACE_OVER_BUDGET` with a friendly UI message.
6. Every LLM call is logged with `workspaceId`, `userId`, `model`, `tokens`, `latencyMs`. The dashboard can filter and aggregate.

**Stretch**

- Add a webhook: when the job completes, the worker POSTs to a URL the user registered. Re-sign the webhook with HMAC and verify on receipt.
- Add an eval job: every 100th LLM response is graded by a judge LLM on faithfulness and relevance; an alert fires when the rolling 24h mean drops below 0.7.

## AI Industry Note (2026)

The streaming/workers/caching trio is no longer "advanced." It is the default. In 2026:

- **Streaming is mandatory.** A non-streaming LLM response feels broken. Users notice 200ms-of-silence differences. The `ai` SDK, Vercel's AI Chat, and every competitor do this out of the box.
- **BullMQ is the standard.** Sidekiq/Celery holdouts have largely moved to BullMQ for any greenfield Node/TypeScript project. Temporal is gaining for very-long-running workflows.
- **Redis is the AI cache.** Valkey (the open-source continuation of Redis) has kept the same API. The decision is no longer "Redis or Memcached" — it is "Redis OSS or Redis Cloud or Upstash."
- **Per-tenant budgets are now a security control.** The first big AI-company scandal of 2026 was a customer who found a loop that spent $400k in a weekend. After that, every enterprise procurement checklist includes "per-tenant budget enforcement."
- **AI gateways are a product category.** Portkey, Helicone, Cloudflare AI Gateway, and OpenRouter all ship caching, fallback, and cost tracking as a service. You can buy what Module 56 teaches you to build. We still teach it because (a) the marginal cost of running it yourself is small once you have Redis, and (b) the off-the-shelf gateways do not give you the audit log your enterprise customers will demand.
- **The "eval layer" is the new moat.** Models are commoditizing. The team that runs a continuous eval loop and improves prompts against it is the team that ships a better product, not just a faster wrapper.