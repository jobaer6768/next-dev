# Module 47: LangChain Application Development

## Why this module matters

By the end of Module 46 you can call OpenAI, stream tokens, validate JSON, and budget tokens — all in about 200 lines of TypeScript. So why does LangChain exist, and why is it a 50-million-line install footprint instead of a one-file SDK? Because the second you ship a real product, the same patterns show up over and over: prompt templates, output parsers, message history, tool use, retrieval, evaluation. LangChain is a **composability library** that gives you the pieces and a single `Runnable` interface so you can wire them together without rewriting the glue.

The reason this module is here is the same reason React is here in the frontend mission: not because you cannot write it yourself, but because the abstraction is genuinely good and the ecosystem it unlocks (LangSmith, LangServe, the 600+ integrations) is what your future team will already know. The reason the module also teaches you when *not* to use it is the same reason the React mission also teaches you vanilla DOM: abstractions are paid for in cognitive load, and you should be the one deciding when to take the loan.

## Learning objectives

By the end of this module you will be able to:

1. Decide when LangChain earns its keep and when raw SDKs are simpler.
2. Compose prompts, models, and parsers with the LangChain Expression Language (LCEL).
3. Explain the `Runnable` interface — `invoke`, `stream`, `batch` — and use it consistently.
4. Build a chat-with-memory endpoint using `BaseChatMessageHistory` and trimming.
5. Debug a chain with LangSmith, verbose mode, and manual logging.
6. Replace a 150-line hand-rolled pipeline with a 30-line LCEL pipeline — and know when not to.

## Conceptual model

LangChain v0.3 (late 2025) has settled on a single mental model. Everything is a **`Runnable`**. A `Runnable` is anything that takes an input and returns an output and supports three methods:

- `invoke(input)` — call once, await result.
- `stream(input)` — call once, get an async iterator of partial outputs.
- `batch([inputs])` — call N times in parallel, return N results.

Prompt templates, chat models, output parsers, retrievers, tools — all `Runnable`. You compose them with the pipe operator (or `.pipe()` for the visually cautious), and the composition itself is a `Runnable`. This is the same idea as a Unix pipeline and the same idea as a Promise chain: small, composable, every step is a function.

```
input → [prompt] → [model] → [parser] → output
```

The most common LCEL pipeline you will ever write:

```ts
import { ChatPromptTemplate } from "@langchain/core/prompts";
import { ChatOpenAI } from "@langchain/openai";
import { StringOutputParser } from "@langchain/core/output_parsers";
import { RunnablePassthrough, RunnableSequence } from "@langchain/core/runnables";

const prompt = ChatPromptTemplate.fromMessages([
  ["system", "You are a concise assistant. Reply in {language}."],
  ["human", "{question}"],
]);

const model = new ChatOpenAI({ model: "gpt-4.1-mini", temperature: 0 });
const parser = new StringOutputParser();

const chain = prompt.pipe(model).pipe(parser);

// All three work on the same chain:
const a = await chain.invoke({ language: "French", question: "What is RAG?" });
const b = await chain.stream({ language: "French", question: "What is RAG?" });
const c = await chain.batch([
  { language: "French", question: "What is RAG?" },
  { language: "Spanish", question: "What is RAG?" },
]);
```

That is the entire mental model. The rest of this module is the building blocks you slot into the slots.

### When NOT to use LangChain

I will not pretend the abstraction is free. Use raw SDKs when:

- You are calling one model, one prompt, one output. `openai.chat.completions.create()` is shorter than `ChatPromptTemplate.fromMessages(...).pipe(ChatOpenAI).pipe(StringOutputParser)`.
- You are shipping a single-file script.
- You need bleeding-edge model features the LangChain adapter has not caught up to.
- You are evaluating "is the wrapper faster than the raw call?" — sometimes it is, sometimes it is not. Measure.

Use LangChain when:

- You have a multi-step pipeline (retrieve → rerank → prompt → generate → parse → validate).
- You want to swap models without rewriting code.
- You want LangSmith tracing for free.
- You want a single interface for `invoke`, `stream`, `batch` across providers, retrievers, and tools.
- You are building agents (Module 52).

### Prompt templates

The `ChatPromptTemplate` API is the reason most teams standardize on LangChain. It supports variables, message roles, message placeholder injection (for history), and partial variables (for things you know at startup time).

```ts
import { ChatPromptTemplate, MessagesPlaceholder } from "@langchain/core/prompts";

const triagePrompt = ChatPromptTemplate.fromMessages([
  ["system", `
    You triage support tickets for the "Mission" SaaS.
    Products: {products}
    Severities: low | medium | high
    Always reply with JSON matching the schema.
  `],
  new MessagesPlaceholder("history"),       // <-- chat memory gets injected here
  ["human", "{ticket}"],
]);

// Partial: lock in a value at startup.
const boundPrompt = await triagePrompt.partial({ products: "Billing, Auth, Dashboard, Other" });

// Render to inspect what will actually be sent (great for debugging).
const rendered = await boundPrompt.invoke({
  ticket: "Charged me twice.",
  history: [],   // you can pass a real history later
});
console.log(rendered.toChatMessages());
```

### Output parsers

Output parsers are the right side of the pipe. They take whatever the model emits and turn it into something your code can use. The four you will use 95% of the time:

```ts
import { StringOutputParser } from "@langchain/core/output_parsers";
import { JsonOutputParser } from "@langchain/core/output_parsers";
import { StructuredOutputParser } from "langchain/output_parser";
import { z } from "zod";

const stringParser  = new StringOutputParser();
const jsonParser    = new JsonOutputParser();

const TicketSchema = z.object({
  product:  z.enum(["Billing", "Auth", "Dashboard", "Other"]),
  severity: z.enum(["low", "medium", "high"]),
  summary:  z.string(),
});
const structuredParser = StructuredOutputParser.fromZodSchema(TicketSchema);

// The structured parser injects format instructions into the prompt for you.
const prompt2 = ChatPromptTemplate.fromMessages([
  ["system", "Triage this support ticket."],
  ["human", "{ticket}\n{format_instructions}"],
]);
const chain2 = prompt2.pipe(model).pipe(structuredParser);
const result = await chain2.invoke({ ticket: "Charged me twice on May 3." });
// result is a typed object — no JSON.parse, no try/catch, no Zod dance.
```

The mistake I see constantly: using `JsonOutputParser` and then Zod-parsing the result. The `StructuredOutputParser` does both. Pick the right one.

### LCEL composition patterns you will write a hundred times

```ts
import { RunnablePassthrough, RunnableMap, RunnableBranch } from "@langchain/core/runnables";

// 1. Parallel branches — send one input to two models, merge results.
const parallel = RunnableMap.from({
  summary:    summaryChain,
  sentiment:  sentimentChain,
});
const merged = await parallel.invoke({ review: "..." });

// 2. Branching — pick a sub-chain based on the input.
const router = RunnableBranch.from([
  [{ route: "code" },  codeChain  ],
  [{ route: "chat" },  chatChain  ],
  defaultChain,
]);

// 3. Pass-through — keep the original input alongside a derived value.
const enriched = RunnablePassthrough.assign({
  translated: translationChain,   // adds `translated` to the output
});
```

### Message history and chat memory

Memory is where LangChain earns its keep for chat products. The interface is `BaseChatMessageHistory`; the implementation is usually `PostgresChatMessageHistory` or `RedisChatMessageHistory`. In the bootcamp we use Postgres — same database as the SaaS.

```ts
// src/lib/chat/history.ts
import { BaseChatMessageHistory } from "@langchain/core/chat_history";
import { AIMessage, HumanMessage, BaseMessage } from "@langchain/core/messages";
import { db } from "@/db";
import { messages } from "@/db/schema";
import { eq, and, asc } from "drizzle-orm";

export class PostgresChatMessageHistory extends BaseChatMessageHistory {
  lc_namespace = ["bootcamp", "PostgresChatMessageHistory"];

  constructor(
    private readonly workspaceId: string,
    private readonly conversationId: string,
  ) {
    super();
  }

  async getMessages(): Promise<BaseMessage[]> {
    const rows = await db
      .select()
      .from(messages)
      .where(
        and(
          eq(messages.workspaceId, this.workspaceId),
          eq(messages.conversationId, this.conversationId),
        ),
      )
      .orderBy(asc(messages.createdAt));

    return rows.map((r) =>
      r.role === "user"
        ? new HumanMessage(r.content)
        : new AIMessage(r.content),
    );
  }

  async addMessage(message: BaseMessage): Promise<void> {
    await db.insert(messages).values({
      workspaceId:    this.workspaceId,
      conversationId: this.conversationId,
      role:           message._getType() === "ai" ? "assistant" : "user",
      content:        message.content as string,
    });
  }

  async clear(): Promise<void> {
    await db.delete(messages).where(
      and(
        eq(messages.workspaceId, this.workspaceId),
        eq(messages.conversationId, this.conversationId),
      ),
    );
  }
}
```

The trimming problem: a 500-message history will not fit any model. LangChain ships a `trimMessages` helper for the standard "keep system + last N turns" pattern.

```ts
import { trimMessages } from "@langchain/core/messages";

const history = await history$.getMessages();
const trimmed = await trimMessages(history, {
  maxTokens: 4_000,
  strategy: "last",
  startOn: "human",                 // never end on an assistant turn
  tokenCounter: new ChatOpenAI({ model: "gpt-4o-mini" }),  // any model works
});
```

### Chat-with-memory: the complete endpoint

```ts
// src/routes/chat-with-memory.ts
import { ChatPromptTemplate, MessagesPlaceholder } from "@langchain/core/prompts";
import { ChatOpenAI } from "@langchain/openai";
import { StringOutputParser } from "@langchain/core/output_parsers";
import { RunnableWithMessageHistory } from "@langchain/core/runnables";
import { PostgresChatMessageHistory } from "@/lib/chat/history";
import { z } from "zod";
import { Hono } from "hono";
import { stream } from "hono/streaming";

const prompt = ChatPromptTemplate.fromMessages([
  ["system", "You are Atlas, a concise support assistant. Reply in {language}."],
  new MessagesPlaceholder("history"),
  ["human", "{input}"],
]);

const model = new ChatOpenAI({ model: "gpt-4.1-mini", temperature: 0 });
const baseChain = prompt.pipe(model).pipe(new StringOutputParser());

const withHistory = new RunnableWithMessageHistory({
  runnable: baseChain,
  getMessageHistory: (sessionId) => {
    // sessionId is "<workspaceId>:<conversationId>" — keeps the cache key
    // unique per tenant. Cross-tenant leakage is a P0 bug.
    const [workspaceId, conversationId] = sessionId.split(":");
    return new PostgresChatMessageHistory(workspaceId, conversationId);
  },
  inputMessagesKey: "input",
  historyMessagesKey: "history",
});

const ChatBody = z.object({
  workspaceId:    z.string().uuid(),
  conversationId: z.string().uuid(),
  input:          z.string().min(1).max(8000),
  language:       z.string().default("English"),
});

const app = new Hono();
app.post("/api/chat", async (c) => {
  const auth = c.get("auth");                  // from JWT middleware
  const body = ChatBody.parse(await c.req.json());

  if (body.workspaceId !== auth.workspaceId) {
    return c.json({ error: "forbidden" }, 403);
  }

  const sessionId = `${body.workspaceId}:${body.conversationId}`;
  const config = { configurable: { sessionId } };

  c.header("Content-Type", "text/event-stream");
  return stream(c, async (s) => {
    for await (const chunk of await withHistory.stream(
      { input: body.input, language: body.language },
      config,
    )) {
      await s.writeSSE({ event: "delta", data: JSON.stringify({ text: chunk }) });
    }
    await s.writeSSE({ event: "done", data: "{}" });
  });
});
```

That is 50 lines and replaces ~200 lines of the hand-rolled version in Module 46. The same code works on Anthropic, Mistral, or any local model by swapping `ChatOpenAI` for `ChatAnthropic`, `ChatMistralAI`, or `ChatOllama`. That is the deal LangChain is offering you.

### Debugging chains

Three tools, in order of effort.

```ts
// 1. Verbose mode — log every step to stdout. Cheapest, fastest.
import { setDebug } from "langchain/global";
setDebug(true);
await chain.invoke(input);
setDebug(false);

// 2. Manual logging — wrap any Runnable in a logger.
import { RunnableLambda } from "@langchain/core/runnables";
const logged = chain.pipe(
  new RunnableLambda({
    func: async (out) => {
      console.log("[chain] output:", out);
      return out;
    },
  }),
);

// 3. LangSmith — full tracing UI. Set four env vars and you are done.
process.env.LANGCHAIN_TRACING_V2 = "true";
process.env.LANGCHAIN_API_KEY    = process.env.LANGSMITH_API_KEY!;
process.env.LANGCHAIN_PROJECT    = "mission-6-chat";
// Every invoke/stream/batch now shows up in the LangSmith dashboard with
// inputs, outputs, latencies, token counts, and a parent-child tree.
```

LangSmith in 2026 is the closest thing the AI industry has to a standard observability stack. Even if you do not pay for it, the free tier covers most bootcamp projects and most early-stage startups. Set it up on day one.

## Common mistakes

- **Reaching for LangChain on a one-line call.** If your script is 20 lines, the raw SDK is shorter, faster, and easier to debug.
- **Treating `MessagesPlaceholder("history")` as magic.** It is not. If you do not pass `history` in the input, the prompt will literally contain the string "history" and the model will get confused.
- **Forgetting to scope the sessionId by workspace.** `getMessageHistory` is called with a string. If that string is just `conversationId`, you have built a cross-tenant data leak. Always encode the workspace into the sessionId and verify it matches the JWT.
- **Storing message history in a global LRU instead of the database.** The first pod restart loses every conversation. The second pod loses every conversation. The third pod loses every conversation.
- **Parsing JSON by hand when `StructuredOutputParser` exists.** Same applies to `JsonOutputParser` and the `transform`/`pipe` chain.
- **Skipping LangSmith "because we are small."** You are not too small to need observability. You are too small to debug blind.

## Exercise

Refactor the Module 46 chat endpoint to use LangChain. Acceptance criteria:

1. Replaces the hand-rolled `budgetMessages` and `callWithResilience` flow with an LCEL chain: `prompt.pipe(model).pipe(parser)`.
2. Uses `RunnableWithMessageHistory` backed by a `PostgresChatMessageHistory` class that reads/writes the `messages` table from Mission 4.
3. Adds a second chain, `triageChain`, that takes a raw ticket and returns a Zod-validated `{ product, severity, summary, escalate }` object using `StructuredOutputParser.fromZodSchema`.
4. Adds a `POST /api/chat/triage` endpoint that runs `triageChain` (not streaming — synchronous) and returns the JSON.
5. Includes a unit test that runs both chains against a fake `ChatModel` from `@langchain/core/utils/testing` and asserts the output shape.
6. Verifies that a JWT for workspace A cannot read or write the history of workspace B (cross-tenant test).

Stretch: turn on LangSmith tracing and walk through one full request in the dashboard. Annotate each node with the cost (input/output tokens × $/M).

## AI Industry Note (2026)

LangChain in 2026 is a different beast from the LangChain of 2023. Three shifts:

1. **The "wrapper tax" debate is over.** In 2023, Twitter was full of "you do not need LangChain" takes. In 2026, most production teams use it or LlamaIndex for at least the orchestration layer, because the alternative is rebuilding the same observability, retry, and streaming glue every time. The skill is not "should I use it" — it is "which pieces."
2. **LangSmith is the de facto tracing tool.** Even teams that use raw SDKs pipe their calls through LangSmith for observability. The free tier in 2026 covers most projects; the paid tier is what you buy the day your first investor asks for an eval dashboard.
3. **The `Runnable` interface won.** Competing frameworks (LlamaIndex agents, Haystack, the DSPy-style declarative prompt compilers) have all converged on the same idea: small composable units, async-first, streamable. Learning LCEL in this module is learning the industry's lingua franca for AI orchestration, not a vendor-specific dialect.
