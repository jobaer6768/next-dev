# Module 52: Advanced Agent Systems

## Why this module matters

A single agent with ten tools is impressive in a demo and miserable in production. The real work of an AI feature is not "the model called a tool." It is "the system remembered what the user told it last week, paused for approval before sending the email, refused the prompt-injection attempt in the user's PDF, and stayed under budget." That is what separates a 2026 production agent from a 2024 demo.

This module takes the LangGraph agent from Module 51 and surrounds it with everything it needs to live in the wild: memory, multi-agent collaboration, guardrails, cost caps, and the production deployment pattern.

## Learning objectives

1. Distinguish short-term (in-conversation) memory from long-term (vector store / DB) memory and pick the right one.
2. Build a human-in-the-loop checkpoint that pauses the graph, asks the client, and resumes.
3. Compose multiple specialized agents into a router/executor/synthesizer pipeline.
4. Add input and output guardrails to neutralize the most common jailbreak patterns.
5. Deploy an agent as a queue + worker with timeouts, retries, and per-workspace cost caps.
6. Implement semantic search and an "inbox-zero" email/task assistant.

## Conceptual model

The five advanced concerns, mapped to one diagram:

```
                ┌──────────────┐
                │   Client UI  │
                └──────┬───────┘
                       │ HTTPS + JWT
                ┌──────▼───────┐
                │  API Gateway │  (Module 55: per-workspace rate limits)
                └──────┬───────┘
                       │
                ┌──────▼───────┐    enqueue
                │ BullMQ Queue│◀────────────┐
                └──────┬───────┘             │
                       │                     │
                ┌──────▼───────┐             │
                │ Agent Runner │             │
                │  ┌────────┐  │             │
                │  │ Router │──┼─── tools   │ retry
                │  └────┬───┘  │             │
                │       │      │             │
                │  ┌────▼───┐  │   ┌─────┐   │
                │  │Special-│  │   │ HITL│◀──┘ (resume)
                │  │ists    │──┼──▶│Wait │
                │  └────┬───┘  │   └─────┘
                │       │      │
                │  ┌────▼───┐  │
                │  │Guard-  │  │
                │  │rails   │  │
                │  └────────┘  │
                └──────┬───────┘
                       │ SSE stream
                ┌──────▼───────┐
                │   pgvector   │  (memory + RAG corpus)
                └──────────────┘
```

The agent is no longer a function; it is a service.

### 1. Short-term vs. long-term memory

```ts
// src/lib/agents/memory.ts
import { PostgresCheckpointer } from "@langchain/langgraph/checkpoint/postgres";
import { OpenAIEmbeddings } from "@langchain/openai";
import { PGVectorStore } from "@langchain/community/vectorstores/pgvector";

// Short-term: the current thread. Resets when the user starts a new conversation.
export const checkpointer = PostgresCheckpointer.fromConnString(process.env.DATABASE_URL!);

// Long-term: facts the agent should remember across sessions.
// Store: "user:123 prefers concise answers", "user:123 is in EST timezone"
export const longTermStore = new PGVectorStore(new OpenAIEmbeddings(), {
  pool: { connectionString: process.env.DATABASE_URL! },
  tableName: "agent_long_term_memory",
});

export async function recallAbout(userId: string, query: string) {
  const docs = await longTermStore.similaritySearch(query, 5, { filter: { user_id: userId } });
  return docs.map((d) => d.pageContent).join("\n");
}

export async function remember(userId: string, fact: string) {
  await longTermStore.addDocuments([{ pageContent: fact, metadata: { user_id: userId, ts: Date.now() } }]);
}
```

The discipline: short-term lives in the graph state and the Postgres checkpointer; long-term lives in a separate vector table scoped to the user. Do not mix them.

### 2. Human-in-the-loop

```ts
// src/lib/agents/hitl.ts
import { interrupt } from "@langchain/langgraph";

export async function askApproval(state, action: { type: string; payload: unknown }) {
  // `interrupt` pauses the graph and yields control back to the caller.
  // The caller resumes with `Command({ resume: { approved: boolean } })`.
  const answer = await interrupt({
    type: "approval_required",
    message: `Approve this action: ${action.type}?`,
    details: action.payload,
  });

  if (!answer.approved) throw new Error("USER_REJECTED");
  return state;
}
```

Server side:

```ts
// src/app/api/agent/resume/route.ts
import { Command } from "@langchain/langgraph";

export async function POST(req: Request) {
  const { threadId, approved } = await req.json();
  const result = await agent.invoke(
    new Command({ resume: { approved } }),
    { configurable: { thread_id: threadId } }
  );
  return Response.json(result);
}
```

HILT is the difference between an "AI feature" and an "AI product." If the agent can send email, delete a record, or charge a card, it must pause for approval first.

### 3. Multi-agent: router + specialists + synthesizer

```ts
// src/lib/agents/multi.ts
import { StateGraph, END } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { Annotation } from "@langchain/langgraph";
import { searchTool } from "./tools";

const router = new ChatOpenAI({ model: "gpt-4o-mini" }).withStructuredOutput(
  z.object({ route: z.enum(["research", "code", "general"]), reason: z.string() })
);

const MultiState = Annotation.Root({
  question: Annotation<string>(),
  route: Annotation<string>(),
  researchNotes: Annotation<string>({ default: () => "" }),
  codeDraft: Annotation<string>({ default: () => "" }),
  finalAnswer: Annotation<string>({ default: () => "" }),
});

async function pickRoute(state) {
  const { route, reason } = await router.invoke([
    { role: "system", content: "Classify the user question into one of the routes." },
    { role: "user", content: state.question },
  ]);
  return { route };
}

async function researchAgent(state) {
  const llm = new ChatOpenAI({ model: "gpt-4o-mini" }).bindTools([searchTool]);
  const r = await llm.invoke([{ role: "user", content: state.question }]);
  return { researchNotes: r.content as string };
}

async function codeAgent(state) {
  const llm = new ChatOpenAI({ model: "gpt-4o-mini" });
  const r = await llm.invoke([{ role: "system", content: "You write production TypeScript." }, { role: "user", content: state.question }]);
  return { codeDraft: r.content as string };
}

async function synthesize(state) {
  const llm = new ChatOpenAI({ model: "gpt-4o-mini" });
  const prompt = `Synthesize a final answer from: ${JSON.stringify(state)}\nQuestion: ${state.question}`;
  const r = await llm.invoke(prompt);
  return { finalAnswer: r.content as string };
}

const graph = new StateGraph(MultiState)
  .addNode("route", pickRoute)
  .addNode("research", researchAgent)
  .addNode("code", codeAgent)
  .addNode("synthesize", synthesize)
  .addEdge("__start__", "route")
  .addConditionalEdges("route", ({ "route": r }) => r, {
    research: "research",
    code: "code",
    general: "synthesize",
  })
  .addEdge("research", "synthesize")
  .addEdge("code", "synthesize")
  .addEdge("synthesize", END);

export const multiAgent = graph.compile();
```

The PM pattern: **planner → executor → reviewer**. Each role has narrow tools, narrow prompts, narrow evaluators. Three small agents beat one large agent on every benchmark that matters.

### 4. Guardrails

```ts
// src/lib/agents/guardrails.ts
import { ChatOpenAI } from "@langchain/openai";

const INJECTION_PATTERNS = [
  /ignore previous instructions/i,
  /disregard (all|any) prior/i,
  /you are now/i,
  /system:\s*you/i,
];

export function inputGuardrail(text: string): { ok: true } | { ok: false; reason: string } {
  for (const re of INJECTION_PATTERNS) {
    if (re.test(text)) return { ok: false, reason: "prompt_injection_detected" };
  }
  if (text.length > 50_000) return { ok: false, reason: "input_too_long" };
  return { ok: true };
}

const judge = new ChatOpenAI({ model: "gpt-4o-mini", temperature: 0 }).withStructuredOutput(
  z.object({
    isSafe: z.boolean(),
    reason: z.string(),
  })
);

export async function outputGuardrail(answer: string): Promise<{ ok: true } | { ok: false; reason: string }> {
  const { isSafe, reason } = await judge.invoke([
    { role: "system", content: "Flag if the assistant (a) leaked credentials, (b) produced instructions for weapons/drugs, (c) violated PII policy." },
    { role: "user", content: answer },
  ]);
  return isSafe ? { ok: true } : { ok: false, reason };
}
```

Defense layer order: cheap regex → model-as-judge for the gray area → human review for high-stakes actions.

### 5. Queue + worker + cost cap

```ts
// src/workers/agent.worker.ts
import { Worker, Queue } from "bullmq";
import { Redis } from "ioredis";
import { multiAgent } from "@/lib/agents/multi";
import { usageTracker } from "@/lib/billing/usage";

const conn = new Redis(process.env.REDIS_URL!, { maxRetries: null });

export const agentQueue = new Queue("agent-runs", { connection: conn });

const HARD_STEP_CAP = 12;
const HARD_TOKEN_CAP = 200_000;

new Worker(
  "agent-runs",
  async (job) => {
    const { question, workspaceId, userId } = job.data;

    // Cost cap check BEFORE we even start.
    const used = await usageTracker.tokensThisMonth(workspaceId);
    if (used > HARD_TOKEN_CAP) throw new Error("WORKSPACE_OVER_BUDGET");

    const result = await multiAgent.invoke(
      { question },
      {
        configurable: { thread_id: job.id },
        recursionLimit: HARD_STEP_CAP,
        callbacks: [
          {
            handleLLMEnd: async (out) => {
              await usageTracker.record(workspaceId, out.llmOutput?.tokenUsage);
            },
          },
        ],
      }
    );
    return result;
  },
  { connection: conn, concurrency: 4, timeout: 120_000 }
);
```

The two production rules every senior knows: (1) every LLM call has a token budget enforced by code, not by prompt; (2) every agent run has a timeout enforced by the queue, not by the LLM client.

### 6. Semantic search & recommendations

```ts
// src/lib/search/semantic.ts
import { vectorStore } from "@/lib/rag/embeddings";

export async function personalizedSearch(workspaceId: string, query: string, userId: string) {
  // Mix: 0.7 * semantic similarity + 0.3 * personalization
  const base = await vectorStore.similaritySearchWithScore(query, 20, { filter: { workspace_id: workspaceId } });
  const userHistory = await recallAbout(userId, query);
  const rerankPrompt = `Given the user's interests:\n${userHistory}\n\nRank the candidates by relevance to "${query}".`;
  // ...re-rank as in Module 50
  return base;
}
```

### 7. Inbox-zero email assistant

```ts
// src/lib/agents/inbox.ts
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const triageSchema = z.object({
  emailId: z.string(),
  category: z.enum(["urgent", "reply", "fyi", "spam"]),
  draftReply: z.string().optional(),
});

export const emailAgent = (graph) =>
  graph.compileNode("triage", async (state) => {
      const draft = await llm.invoke([
        { role: "system", content: "Categorize the email and draft a reply if needed. Never send — always HITL." },
        ...state.messages,
      ]);
      const parsed = triageSchema.parse(JSON.parse(draft.content));
      // Anything marked "urgent" or with a non-empty reply must be human-approved.
      return { requiresApproval: parsed.category === "urgent" || !!parsed.draftReply, triage: parsed };
    });
```

The pattern is universal: classify, draft, **pause**, send. Never send.

## Common mistakes

1. **HILT only for "dangerous" actions.** If a draft sends, the user must approve. Period. The cost of an unprompted email is bigger than the cost of a button.
2. **Long-term memory without a TTL.** Memories without an "I stopped being true" path become a graveyard of stale facts. Add a `confidence` and `expires_at`.
3. **Multi-agent without observability.** Three agents means three traces per run. If you are not using LangSmith's multi-graph view, you are flying blind.
4. **Guardrails that are pure prompts.** `"Be safe in your response"` does nothing. Use a classifier model or a regex; treat safety as code.
5. **No cost cap.** An agent in a loop will drain the LLM budget before you finish your coffee. The cap is the safety net.
6. **Forgetting memory scoping.** Long-term memory scoped by `user_id` and `workspace_id` is the difference between a feature and a GDPR violation.

## Exercise

Add an "AI email triage" feature to your Mission 4 SaaS.

**Acceptance criteria**

1. When a new email arrives (mock the inbox), the agent categorizes it as `urgent`, `reply`, `fyi`, or `spam`.
2. For `urgent` and `reply`, the agent drafts a response but does NOT send it.
3. The draft appears in the dashboard with **Approve** and **Reject** buttons. Approving sends; rejecting discards.
4. Every run is stored with its full LangGraph state and is viewable as a timeline.
5. The workspace has a per-month token cap; exceeding it pauses the queue and emails the workspace owner.

**Stretch**

- Add a second specialist agent that summarizes long email threads; route to it only when `state.threadLength > 5`.
- Add a long-term memory recall: the agent's draft reply references the user's *previous* preferences.

## AI Industry Note (2026)

The frontier of agent systems in 2026 is no longer "can the model reason." The correct question is "can the *system* remember, refuse, escalate, and stay within budget?" That is a different kind of engineering, and it is the one that pays.

- **Memory is now a first-class product surface.** ChatGPT's "Memory", Claude's "Projects", and every vertical AI tool now treats cross-session recall as a feature, not an implementation detail.
- **HITL is shifting left.** In 2024, HITL was a debugging tool. In 2026, it is the *default mode* for any agent that touches money, identity, or external communication.
- **Multi-agent is winning.** Single-agent tool-calling tops out around 15–20 tools before reliability collapses. Multi-agent decompositions (router + specialists) routinely hit 30+ tool calls with high success rates.
- **Cost caps are now a security control.** A prompt-injection that loops an agent is a denial-of-wallet attack. The queue's `HARD_TOKEN_CAP` is as important as your firewall.
- **Guardrails as a market.** Lakera, Guardrails AI, and Rebuff are now billion-dollar-adjacent companies because every enterprise AI deployment has a CISO asking "how do we know it is safe?"
- **The "agent from day one" pattern is the new SaaS shape.** Linear, Notion, and HubSpot now ship agents that act on the user's data inside the same product. The agent *is* the UI.