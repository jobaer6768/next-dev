# Module 51: LangGraph & AI Agents

## Why this module matters

A chain is a list of steps. An agent is a *graph* — sometimes a list, sometimes a loop, sometimes a tree, sometimes a thing that talks to a human and then continues. LangChain's `LLMChain` could fake one or two of those shapes. LangGraph is the framework that takes "agent" out of the demo and puts it into production: typed state, conditional routing, durable execution, and tracing.

By the end of this module you will be able to take a workflow that a junior dev would have written as `if/else` spaghetti and express it as a graph a senior would be willing to merge on Friday.

## Learning objectives

1. Explain why a graph (not a chain) is the right mental model for an AI workflow.
2. Build a LangGraph state machine with `StateGraph`, typed channels, and reducers.
3. Implement conditional routing, cycles, and human-in-the-loop breakpoints.
4. Wire tool calling with Zod-validated schemas so the LLM cannot crash your service.
5. Trace and debug agent runs using LangSmith and `thread_id`-based inspection.
6. Ship a research agent that searches the web and uses a calculator.

## Conceptual model

A LangGraph agent is three things:

1. **State** — a typed object (`Annotation.Root({...})`) that flows through the graph. Everything the agent knows, every tool result, every retry counter: it lives here.
2. **Nodes** — pure functions `(state) => Partial<state>`. They do not know what came before or what comes next.
3. **Edges** — either fixed (`"plan" → "answer"`) or conditional (`(state) => state.needsSearch ? "search" : "answer"`).

The graph executor walks the edges, runs the nodes, updates state, and repeats until it hits an `END` node. The same mental model as React's data flow, but for AI.

### State with typed channels and reducers

```ts
// src/lib/agents/state.ts
import { Annotation } from "@langchain/langgraph";
import { BaseMessage } from "@langchain/core/messages";

const messageReducer = (left: BaseMessage[], right: BaseMessage[] | BaseMessage) => {
  return Array.isArray(right) ? [...left, ...right] : [...left, right];
};

export const AgentState = Annotation.Root({
  // The conversation so far. Reducer appends; never overwrites.
  messages: Annotation<BaseMessage[]>({ reducer: messageReducer, default: () => [] }),

  // The user's original question, frozen at the start.
  question: Annotation<string>(),

  // Working scratchpad for tool results. Overwritten each step.
  scratchpad: Annotation<Record<string, unknown>>({ default: () => ({}) }),

  // Step counter so we can refuse to loop forever.
  step: Annotation<number>({ default: () => 0, reducer: (a, b) => (b ?? 0) + 1 }),

  // Set by tools to drive the next edge.
  nextAction: Annotation<"search" | "calculate" | "answer" | "end">(),
});

export type AgentStateType = typeof AgentState.State;
```

The `reducer` is the heart of the design. Without it, every node would clobber the previous node's output. This is the LangGraph equivalent of React's `setState((s) => ({...s, ...next}))`.

### Tools with Zod schemas

```ts
// src/lib/agents/tools.ts
import { tool } from "@langchain/core/tools";
import { z } from "zod";
import { TavilySearchResults } from "@langchain/community/tools/tavily_search";

const searchSchema = z.object({
  query: z.string().min(3).describe("The web search query, in English."),
  max_results: z.number().int().min(1).max(10).default(5),
});

const calculatorSchema = z.object({
  expression: z.string().describe("A math expression, e.g. '4096 * 0.75'"),
});

export const searchTool = tool(
  async ({ query, max_results }) => {
    const tavily = new TavilySearchResults({ maxResults: max_results });
    const results = await tavily.invoke(query);
    return JSON.stringify(results);
  },
  { name: "web_search", description: "Search the public web. Use for current events, prices, docs.", schema: searchSchema }
);

export const calcTool = tool(
  async ({ expression }) => {
    // Do NOT eval() user input. Use a real math parser.
    const { evaluate } = await import("mathjs");
    const result = evaluate(expression);
    return JSON.stringify({ value: result.toString() });
  },
  { name: "calculator", description: "Evaluate a math expression. Use for arithmetic.", schema: calculatorSchema }
);

export const tools = [searchTool, calcTool];
```

**Tool schemas are contracts.** A bad schema means the LLM hallucinates parameters and your service crashes at 3am. Treat every tool schema like a public API.

### The graph itself

```ts
// src/lib/agents/graph.ts
import { StateGraph, END } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { ToolMessage, AIMessage } from "@langchain/core/messages";
import { AgentState } from "./state";
import { tools } from "./tools";

const model = new ChatOpenAI({ model: "gpt-4o-mini", temperature: 0 }).bindTools(tools);

async function planner(state: typeof AgentState.State) {
  // Decide what to do next. The LLM picks a tool or decides it has enough to answer.
  const response = await model.invoke([
    { role: "system", content: "You are a research agent. Use tools when needed. Be concise." },
    ...state.messages,
  ]);
  return { messages: [response] };
}

async function toolRunner(state: typeof AgentState.State) {
  const last = state.messages[state.messages.length - 1] as AIMessage;
  const results: ToolMessage[] = [];
  for (const call of last.tool_calls ?? []) {
    const t = tools.find((x) => x.name === call.name);
    if (!t) continue;
    const out = await t.invoke(call.args);
    results.push(new ToolMessage({ tool_call_id: call.id!, content: String(out), name: call.name }));
  }
  return { messages: results };
}

function shouldContinue(state: typeof AgentState.State) {
  const last = state.messages[state.messages.length - 1] as AIMessage;
  if (state.step >= 8) return "end";               // Safety cap
  if (last.tool_calls && last.tool_calls.length) return "tools";
  return "end";
}

const graph = new StateGraph(AgentState)
  .addNode("planner", planner)
  .addNode("tools", toolRunner)
  .addEdge("__start__", "planner")
  .addConditionalEdges("planner", shouldContinue, { tools: "tools", end: END })
  .addEdge("tools", "planner");

export const agent = graph.compile();
```

Notice three things:

1. **`step` is in state.** The agent cannot loop forever — a hard cap is part of the graph, not an afterthought.
2. **`shouldContinue` is pure.** Given the same state, it returns the same edge. That makes it testable and debuggable.
3. **The graph is data.** `agent.toJSON()` gives you a serializable spec you can ship to a UI as a flow visualizer.

### Running and tracing

```ts
// src/lib/agents/run.ts
import { agent } from "./graph";

export async function research(question: string) {
  return agent.invoke(
    { question, messages: [{ role: "user", content: question }] },
    {
      configurable: { thread_id: crypto.randomUUID() }, // Critical for checkpointing
      callbacks: [{ handleLLMEnd: (out) => console.log("token cost", out.llmOutput?.tokenUsage) }],
    }
  );
}
```

The `thread_id` is what lets LangSmith replay a run later. Treat it as the request ID — log it, return it to the client, store it on the user's row.

## Common mistakes

1. **No step cap.** Agents *will* loop. If your state has a `step` counter, use it. If it does not, add one today.
2. **Tools that throw.** A 500 from Tavily should not crash the whole run. Wrap every tool in a try/catch and return `error: "..."` as the tool output. The LLM can recover from "search failed"; it cannot recover from an unhandled promise rejection.
3. **Schemas without `description`.** The LLM reads the schema as the tool's API docs. A `description` field on every parameter is the difference between a tool that works and one the model guesses wrong about.
4. **Putting logic in the LLM that belongs in code. "If the calculator returns NaN, retry" is a graph edge, not a prompt instruction. Prompts are unreliable; edges are not.**
5. **Skipping tracing.** Without LangSmith (or your own structured logger), debugging an agent is reading a transcript in reverse. Instrument day one.
6. **Mutating state outside reducers.** If a node mutates `state.messages` directly, you will get subtle ordering bugs. Always `return` a partial; let the framework merge.

## Exercise

Build a research agent that answers questions like "What is the population of Tokyo divided by the population of São Paulo?"

**Acceptance criteria**

1. The agent uses `web_search` for the facts and `calculator` for the division.
2. If a tool fails, the agent retries at most once and then answers with what it has.
3. The step counter is hard-capped at 8. After that, the agent returns the best partial answer it has.
4. Every run has a `thread_id` that is logged to stdout and returned to the caller.
5. Wire it into a Next.js `/api/research` route that streams intermediate tool calls to the client as Server-Sent Events.

**Stretch**

- Add a `human_approval` node: if the calculator is called with a number > 1e9, the graph pauses and asks the client "Allow this calculation?" via an interrupt.
- Persist state to a Postgres checkpointer so a user can resume a session 30 minutes later.

## AI Industry Note (2026)

LangGraph crossed a line in 2025–2026: it is no longer "LangChain's graph thing." It is the de-facto orchestration layer for production agents, with serious adoption in fintech, legal, and dev-tools. What changed:

- **Durable execution is now expected.** A `PostgresSaver` or `RedisSaver` checkpointer is a default, not an upgrade. Users close the tab and reopen it expecting the conversation to be exactly where they left it.
- **LangGraph Platform (launched 2025) is the Anthropic-recommended deploy target for agents.** Competing stacks (CrewAI, AutoGen, raw `instructor`) still have niche advantages, but LangGraph wins on observability and tooling integration.
- **Tool calling reliability is solved by the model, not the framework.** `gpt-4o`, `claude-3.5-sonnet`, and `claude-3.7-sonnet` all hit 95%+ tool-call schema correctness on real workloads. The framework's job is now *recovery from the 5%*, not coaxing the model.
- **The "agent loop" pattern is being standardized.** Planner → executor → critic → revise is the LLM equivalent of a CI pipeline. Newer patterns (Reflexion, Tree-of-Thought) live as library plugins rather than competing frameworks.
- **Multi-agent is the new frontier.** Module 52 territory: instead of one agent with 20 tools, you orchestrate 5 agents with 4 tools each. The bottleneck is now communication, not reasoning.