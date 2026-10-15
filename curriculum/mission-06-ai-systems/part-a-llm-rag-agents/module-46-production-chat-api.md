# Module 46: Building a Production Chat API

## Why this module matters

A demo that calls `openai.chat.completions.create` from a script is a homework assignment. A **production** chat API is what pays salaries: it streams tokens to the browser in under 300ms, returns validated JSON the frontend can render, never blows your budget when a user pastes a 200-page document, fails gracefully when the upstream is degraded, and bills usage back to the right workspace. Every one of those is engineering work, and every one of them is what separates a bootcamp project from a real product.

This module stitches together the SaaS you built in Mission 4 (multi-tenant, JWT auth, Drizzle ORM, Postgres) with the LLM client from Module 45. By the end you will have a single streaming chat endpoint that you can ship on Friday and not get paged on Saturday.

## Learning objectives

By the end of this module you will be able to:

1. Force an LLM to return a schema-validated JSON object (Zod + tool calling + `response_format`).
2. Stream tokens to an HTTP client using Server-Sent Events and the Node `ReadableStream` API.
3. Count and budget tokens, truncate context safely, and decide when to summarize instead of truncating.
4. Wrap an LLM call in retries with exponential backoff, a timeout, and a circuit breaker.
5. Implement per-workspace rate limiting with a sliding window in Postgres.
6. Stand up a complete `POST /api/chat` endpoint with auth, validation, streaming, and graceful errors.

## Conceptual model

A production chat endpoint is six concerns stacked on top of each other. The order matters: auth first, validation second, rate limiting third, then the LLM call, then streaming, then the post-write (logging, billing). Get the order wrong and you either leak data, get DDoS'd, or both.

```
┌──────────────────────────────────────────────────────────────────────┐
│ POST /api/chat                                                       │
│  1. Auth   →  verify JWT, attach workspaceId & userId                │
│  2. Validate → Zod-parse the body against ChatRequest                │
│  3. RateLimit → sliding-window check on (workspaceId, userId)        │
│  4. Budget → count tokens, truncate/summarize history                 │
│  5. Call → LLM with retries, timeout, circuit breaker                 │
│  6. Stream → SSE back to the client; on finish, log + bill            │
└──────────────────────────────────────────────────────────────────────┘
```

### 1. Structured output: three ways to constrain the model

Most production features are not "send the user a paragraph." They are "extract these fields," "classify this ticket," "decide which tool to call." You need the model to return data your code can parse, not prose. Three techniques, in order of preference.

**a. `response_format: { type: "json_schema" }`** — the cleanest. You give the model a JSON Schema; the provider guarantees the output parses against it. Supported by OpenAI and (with slight variations) by most OpenRouter models.

```ts
// src/lib/llm/structured.ts
import { z } from "zod";
import OpenAI from "openai";

export const TicketTriage = z.object({
  product:   z.enum(["Billing", "Auth", "Dashboard", "Other"]),
  severity:  z.enum(["low", "medium", "high"]),
  summary:   z.string().max(140),
  escalate:  z.boolean(),
});

type TicketTriage = z.infer<typeof TicketTriage>;

export async function triageTicket(
  ticket: string,
): Promise<TicketTriage> {
  const llm = new OpenAI();  // uses OPENAI_API_KEY
  const res = await llm.chat.completions.create({
    model: "gpt-4.1-mini",
    response_format: {
      type: "json_schema",
      json_schema: {
        name: "ticket_triage",
        strict: true,
        schema: zodToJsonSchema(TicketTriage),  // from `zod-to-json-schema`
      },
    },
    messages: [
      { role: "system", content: "You triage support tickets. Output JSON only." },
      { role: "user",   content: ticket },
    ],
  });
  const raw = res.choices[0].message.content!;
  return TicketTriage.parse(JSON.parse(raw));  // belt + braces
}
```

**b. Tool calling (function calling) as a structured-output tool** — the oldest, most portable trick. You tell the model about a function that takes your Zod schema as arguments. The model does not actually *call* a function; it returns a structured `tool_calls` array. This is the technique you will use when the provider does not support `json_schema` mode (Anthropic, most open-source models).

```ts
import Anthropic from "@anthropic-ai/sdk";
import { z } from "zod";

const Schema = z.object({
  sentiment: z.enum(["pos", "neu", "neg"]),
  confidence: z.number().min(0).max(1),
  topics: z.array(z.string()).max(5),
});

async function classify(review: string) {
  const anthropic = new Anthropic();
  const tool = {
    name: "emit_classification",
    description: "Emit the classification of the review.",
    input_schema: {
      type: "object",
      properties: {
        sentiment:  { type: "string", enum: ["pos", "neu", "neg"] },
        confidence: { type: "number" },
        topics:     { type: "array", items: { type: "string" } },
      },
      required: ["sentiment", "confidence", "topics"],
    },
  } as const;

  const res = await anthropic.messages.create({
    model: "claude-haiku-4-5",
    max_tokens: 256,
    tools: [tool],
    tool_choice: { type: "tool", name: "emit_classification" },
    messages: [{ role: "user", content: review }],
  });

  const block = res.content.find((b) => b.type === "tool_use")!;
  return Schema.parse((block as any).input);  // tool_use.input
}
```

**c. JSON mode + a parser** — `response_format: { type: "json_object" }` guarantees valid JSON but no schema. You then Zod-parse and retry on failure. Use this as a fallback, not a default.

### 2. Streaming with Server-Sent Events

Users will not wait 8 seconds for a 2,000-token reply. They want to see tokens the instant the model emits them. The protocol for this over HTTP is **Server-Sent Events**: a long-lived `text/event-stream` response where the server pushes `data: ...` lines and the client uses an `EventSource` (or `fetch` with a reader) to consume them.

```ts
// src/routes/chat.ts
import { Hono } from "hono";
import { stream } from "hono/streaming";
import OpenAI from "openai";
import { z } from "zod";

const app = new Hono();

const ChatRequest = z.object({
  conversationId: z.string().uuid(),
  message:        z.string().min(1).max(8000),
});

app.post("/api/chat", async (c) => {
  // (1) Auth — pretend this came from the JWT middleware in Mission 4.
  const { workspaceId, userId } = c.get("auth");

  // (2) Validate.
  const body = ChatRequest.parse(await c.req.json());

  // (3) Rate limit.
  await rateLimit({ workspaceId, userId, costEstimate: body.message.length });

  // (4) Build messages (history + system + new user msg).
  const messages = await buildMessages(workspaceId, body.conversationId, body.message);

  // (5) Call the LLM, streaming, and pipe to the client.
  const openai = new OpenAI();
  const completion = await openai.chat.completions.create({
    model: "gpt-4.1-mini",
    messages,
    stream: true,
    stream_options: { include_usage: true },  // get token counts at the end
  });

  c.header("Content-Type", "text/event-stream");
  c.header("Cache-Control", "no-cache, no-transform");
  c.header("Connection", "keep-alive");

  let inputTokens = 0;
  let outputTokens = 0;

  return stream(c, async (s) => {
    for await (const chunk of completion) {
      const delta = chunk.choices[0]?.delta?.content ?? "";
      if (delta) {
        await s.writeSSE({
          id: chunk.id,
          event: "delta",
          data: JSON.stringify({ text: delta }),
        });
      }
      if (chunk.usage) {
        inputTokens = chunk.usage.prompt_tokens;
        outputTokens = chunk.usage.completion_tokens;
      }
    }

    // Final event: usage + a "done" sentinel.
    await s.writeSSE({
      event: "done",
      data: JSON.stringify({ inputTokens, outputTokens }),
    });

    // Post-write: persist the assistant message and bill usage.
    await saveAssistantMessage(workspaceId, body.conversationId, outputTokens);
  });
});
```

Client side (the SaaS frontend from Mission 4):

```ts
// src/lib/chat/stream.ts
export async function* streamChat(req: ChatRequest, signal: AbortSignal) {
  const res = await fetch("/api/chat", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(req),
    signal,
  });
  if (!res.ok || !res.body) throw new Error(`Chat failed: ${res.status}`);

  const reader = res.body.getReader();
  const decoder = new TextDecoder();
  let buffer = "";

  while (true) {
    const { value, done } = await reader.read();
    if (done) break;
    buffer += decoder.decode(value, { stream: true });

    // SSE messages are separated by a blank line.
    let idx;
    while ((idx = buffer.indexOf("\n\n")) !== -1) {
      const raw = buffer.slice(0, idx);
      buffer = buffer.slice(idx + 2);
      const event = parseSSE(raw);
      if (event.event === "delta") yield event.data.text;
      if (event.event === "done")  return event.data;
    }
  }
}
```

The `AbortSignal` is critical: when the user navigates away you must cancel the upstream call, or you will pay for tokens no one reads. Always thread it through.

### 3. Token budgeting and truncation

You have a 200K window. You have a 50-message history, a 30K-token RAG context, and a system prompt. Naively concatenating will eventually crash, will get expensive, and will quietly degrade quality long before it crashes. You need a budget.

```ts
// src/lib/llm/budget.ts
import { encode } from "gpt-tokenizer";

export type Msg = { role: "system" | "user" | "assistant"; content: string };

const SYSTEM_BUDGET   = 2_000;   // reserve for system prompt
const USER_BUDGET     = 2_000;   // reserve for the latest user message
const RAG_BUDGET      = 8_000;   // reserve for retrieved docs
const RESPONSE_BUDGET = 1_000;   // reserve for the answer
const MODEL_WINDOW    = 128_000; // gpt-4.1-mini window

export function budgetMessages(
  system: string,
  rag: string,
  history: Msg[],
  latest: string,
): Msg[] {
  const sysTokens  = encode(system).length;
  const ragTokens  = encode(rag).length;
  const lastTokens = encode(latest).length;

  const fixed = sysTokens + ragTokens + lastTokens + RESPONSE_BUDGET;
  const historyBudget = Math.max(0, MODEL_WINDOW - fixed);

  // Walk from the most recent turn backwards, keep what fits.
  const kept: Msg[] = [];
  let used = 0;
  for (let i = history.length - 1; i >= 0; i--) {
    const t = encode(history[i].content).length + 4;  // ~4 tokens of overhead
    if (used + t > historyBudget) break;
    used += t;
    kept.unshift(history[i]);
  }

  return [
    { role: "system", content: system },
    ...kept,
    { role: "user", content: rag ? `${rag}\n\n${latest}` : latest },
  ];
}
```

When history is long, two upgrades beat raw truncation:

1. **Summarize old turns.** Every N turns, ask the model to compress the prior N into a 200-token summary; store the summary as a `system` note and drop the originals.
2. **Sliding window with the last user goal.** Keep the most recent 6 turns verbatim and a 1-sentence summary of everything older.

### 4. Retries, timeouts, circuit breakers

LLM calls fail in three flavors: transient (5xx, rate limit), permanent (bad request, content filter), and slow (hung connection). You need a different response to each.

```ts
// src/lib/llm/resilience.ts
import pRetry from "p-retry";
import { CircuitBreaker } from "opossum";

const classifyError = (err: any) => {
  const s = err?.status ?? err?.response?.status;
  if (s === 429 || (s >= 500 && s < 600)) return "transient";
  if (s === 400 || s === 401 || s === 403) return "permanent";
  return "transient";
};

export async function callWithResilience<T>(
  fn: () => Promise<T>,
  opts: { timeoutMs?: number; maxAttempts?: number; key?: string } = {},
) {
  const { timeoutMs = 30_000, maxAttempts = 5, key = "default" } = opts;
  const breaker = getBreaker(key);  // per-(model, endpoint) breaker

  return pRetry(
    async (attempt) => {
      const work = breaker.fire(fn);
      return await withTimeout(work, timeoutMs);
    },
    {
      retries: maxAttempts - 1,
      minTimeout: 500,
      maxTimeout: 8_000,
      factor: 2,
      onFailedAttempt: (e) => {
        if (classifyError(e) === "permanent") throw e;  // no retry
        // jittered backoff handled by p-retry + factor
      },
    },
  );
}

const withTimeout = <T,>(p: Promise<T>, ms: number) =>
  new Promise<T>((_, rej) => setTimeout(() => rej(new Error("timeout")), ms));

const breakers = new Map<string, CircuitBreaker>();
function getBreaker(key: string) {
  let b = breakers.get(key);
  if (!b) {
    b = new CircuitBreaker(() => Promise.resolve(), {
      timeout: 30_000,
      errorThresholdPercentage: 50,
      resetTimeout: 30_000,
    });
    breakers.set(key, b);
  }
  return b;
}
```

Three rules:

- **Always set a timeout.** A model that hangs for 90s will tie up your event loop and burn money. 20–30s for chat, 60s for long-form generation.
- **Retry only transient errors.** Retrying a 400 is a waste and a footgun.
- **Use a circuit breaker per upstream.** When OpenAI is having a bad day, your breaker should open in 30s and stop making it worse. Route to a fallback model in the meantime.

### 5. Rate limiting per workspace

A naive in-memory limiter is wrong for two reasons: it does not survive restarts and it does not work across pods. Use Postgres. One table, one function, atomic.

```sql
-- migrations/0014_rate_limits.sql
CREATE TABLE rate_limit_events (
  workspace_id   uuid        NOT NULL,
  user_id        uuid        NOT NULL,
  occurred_at    timestamptz NOT NULL DEFAULT now(),
  cost           int         NOT NULL DEFAULT 1
);
CREATE INDEX rate_limit_events_workspace_time
  ON rate_limit_events (workspace_id, occurred_at DESC);
```

```ts
// src/lib/rate-limit.ts
import { db } from "@/db";
import { sql } from "drizzle-orm";

const LIMITS = {
  free:       { perMinute: 10,  perDay:  500 },
  pro:        { perMinute: 60,  perDay:  5000 },
  enterprise: { perMinute: 300, perDay: 50000 },
};

export async function rateLimit(args: {
  workspaceId: string;
  userId: string;
  costEstimate: number;
  plan: keyof typeof LIMITS;
}) {
  const limit = LIMITS[args.plan];
  const cost = Math.max(1, Math.ceil(args.costEstimate / 1000));

  // Atomic: count, decide, insert, return result.
  const [row] = await db.execute(sql`
    WITH counted AS (
      SELECT
        SUM(CASE WHEN occurred_at > now() - interval '1 minute'  THEN cost END) AS last_min,
        SUM(CASE WHEN occurred_at > now() - interval '1 day'     THEN cost END) AS last_day
      FROM rate_limit_events
      WHERE workspace_id = ${args.workspaceId}
    ),
    ins AS (
      INSERT INTO rate_limit_events (workspace_id, user_id, cost)
      VALUES (${args.workspaceId}, ${args.userId}, ${cost})
      RETURNING 1
    )
    SELECT last_min, last_day FROM counted
  `);

  if ((row.last_min ?? 0) + cost > limit.perMinute) {
    const err: any = new Error("rate_limited_minute");
    err.retryAfter = 60;
    throw err;
  }
  if ((row.last_day ?? 0) + cost > limit.perDay) {
    const err: any = new Error("rate_limited_day");
    err.retryAfter = 86400;
    throw err;
  }
}
```

Mission 6 Part B will move this to a dedicated Redis or Upstash layer. For now, Postgres is fine and saves you an ops dependency.

### 6. The full endpoint

Putting it together — the only file the frontend ever talks to.

```ts
// src/routes/chat.ts (final version, abbreviated)
app.post("/api/chat", async (c) => {
  const { workspaceId, userId, plan } = c.get("auth");
  const body = ChatRequest.parse(await c.req.json());

  await rateLimit({ workspaceId, userId, costEstimate: body.message.length, plan });

  const { system, history, rag } = await loadContext(workspaceId, body.conversationId);
  const messages = budgetMessages(system, rag, history, body.message);

  const stream$ = callWithResilience(
    () => openai.chat.completions.create({
      model: pickModel(body.message, plan),  // cheap model for short msgs
      messages,
      stream: true,
      stream_options: { include_usage: true },
    }),
    { key: "openai-chat", timeoutMs: 30_000 },
  );

  c.header("Content-Type", "text/event-stream");
  return stream(c, async (s) => pipeChatToSse(s, await stream$));
});

app.onError((err, c) => {
  if (err.message === "rate_limited_minute") {
    return c.json({ error: "rate_limited", retryAfter: 60 }, 429);
  }
  if (err.message === "timeout") {
    return c.json({ error: "upstream_timeout" }, 504);
  }
  console.error(err);
  return c.json({ error: "internal" }, 500);
});
```

## Common mistakes

- **Forgetting `stream_options: { include_usage: true }`.** Without it, the final chunk has no token counts, and you cannot bill accurately. OpenAI does not enable it by default; Anthropic's SDK reports usage in the final message; check the docs per provider.
- **Not threading the `AbortSignal`.** When the user closes the tab, the server keeps streaming and the LLM keeps generating. You pay for tokens no one reads.
- **Trusting `response_format: { type: "json_object" }` to enforce a schema.** It only enforces *valid JSON*. Use `json_schema` mode or tool calling for shape.
- **Retrying on `400`.** A 400 from the model API almost always means your prompt is bad, not the network. Retrying will not help and may trigger a rate limit ban.
- **Building the rate limiter in memory.** Fine for one pod, broken behind a load balancer. Use Postgres or Redis.
- **Logging the full prompt at `info` level.** Prompts often contain user PII. Log only token counts, model, and a redacted prompt hash. Save the full prompt to a separate, encrypted audit log.

## Exercise

Add a `POST /api/chat` to the Mission 4 SaaS repo. Acceptance criteria:

1. Requires a valid JWT; rejects with `401` otherwise.
2. Body is validated with Zod: `{ conversationId: uuid, message: string (1–8000 chars) }`.
3. Calls `rateLimit()` from this module; on limit, returns `429` with `Retry-After`.
4. Streams SSE to the client with `event: delta` per chunk and a final `event: done` carrying token counts.
5. Persists both the user message and the assistant message to the `messages` table (Drizzle) with the right `conversation_id` and `workspace_id`.
6. Wraps the LLM call in `callWithResilience()` with a 30s timeout and 3 retries on transient errors only.
7. Adds a `--dry-run` flag for local dev that returns the built `messages` array without calling the model.
8. Includes an integration test that posts to the endpoint with a mocked OpenAI client and asserts that SSE events are emitted in order.

Stretch: implement the summarization pass — every 20 turns, summarize the oldest 10 turns into a 200-token system note and drop the originals.

## AI Industry Note (2026)

In 2026 the boring engineering of production AI — retries, budgets, observability, rate limits — is the differentiator. Three things are true now that were not in 2024:

1. **Every product is a router.** The "smart" product picks the right model for the right request, falls back when one provider is degraded, and never lets a single upstream failure become a customer-visible incident. The resilience patterns in this module are the floor, not the ceiling.
2. **Latency is a feature.** A 1.2s first-token is the difference between a delightful product and an abandoned one. Streaming, speculative decoding, prompt caching, and edge inference (Workers AI, Vercel AI Gateway, Cloudflare) are not exotic anymore. You are expected to know them.
3. **Cost attribution is a board-level question.** "How much did each workspace cost us last month?" must be answerable from your database in 30 seconds. That is why this module puts `workspace_id` on every usage row, and why you should never aggregate it away.
