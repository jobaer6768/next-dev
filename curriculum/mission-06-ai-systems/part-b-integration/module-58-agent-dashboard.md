# Module 58: Frontend Integration — Agent Dashboard & Polish

## Why this module matters

A user has uploaded 200 PDFs, run 1,400 agent jobs, spent $312 of a $500 AI budget, and is about to upgrade. They open one page and decide whether they trust your product. That page is the dashboard. If it cannot tell them *what is in their knowledge base*, *what their agents have been doing*, *how much they have spent*, and *what is happening right now*, it has failed — even if the chat UI is gorgeous.

This module closes the loop on Mission 6. The chat is the headline; the dashboard is the renewals team. We will build the four surfaces every AI SaaS dashboard must have, plus the polish that distinguishes "shipped" from "feels good."

## Learning objectives

1. Build a knowledge-base management UI: list, upload, delete, status, vector counts.
2. Surface agent run history and traces as a timeline with intermediate tool calls.
3. Display per-workspace AI usage analytics: tokens, cost, requests, latency.
4. Implement feature gating and quota UI ("X / Y AI credits used this month").
5. Polish for accessibility (keyboard shortcuts, ARIA labels) and mobile responsiveness.

## Conceptual model

The dashboard has four tabs, each answering one question:

| Tab | Question | Data source |
|---|---|---|
| Knowledge Base | "What's in my data?" | `kb_documents`, `kb_chunks` |
| Agent Runs | "What have my agents done?" | LangGraph `thread_id` checkpointer + log table |
| Usage & Costs | "How much have I spent?" | `usage` table + Redis counters |
| Settings | "How is this thing configured?" | Workspace settings, integrations |

```
┌────────────────────────────────────────────────────────────────┐
│  Workspace: Acme Co.                       [Plan: Pro ▾]      │
│  Credits: ████████░░ 32,400 / 50,000                          │
├────────────────────────────────────────────────────────────────┤
│  [Knowledge Base] [Agent Runs] [Usage] [Settings]             │
│                                                                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Active tab content                                     │  │
│  └────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

### 1. Knowledge-base management UI

Server side — a paginated query:

```ts
// src/app/api/kb/documents/route.ts
import { db } from "@/lib/db";
import { requireWorkspace } from "@/lib/auth";

export async function GET(req: Request) {
  const ws = await requireWorkspace(req);
  const url = new URL(req.url);
  const cursor = url.searchParams.get("cursor");

  const docs = await db.query(
    `SELECT d.id, d.filename, d.status, d.created_at,
            COUNT(c.id)::int AS chunks
       FROM kb_documents d
       LEFT JOIN kb_chunks c ON c.document_id = d.id
      WHERE d.workspace_id = $1 AND ($2::uuid IS NULL OR d.id < $2)
      GROUP BY d.id
      ORDER BY d.id DESC
      LIMIT 25`,
    [ws.id, cursor]
  );

  return Response.json({ docs: docs.rows, nextCursor: docs.rows.at(-1)?.id ?? null });
}
```

Client — the list:

```tsx
// src/components/kb/document-list.tsx
"use client";
import { useInfiniteQuery, useMutation, useQueryClient } from "@tanstack/react-query";

export function DocumentList({ workspaceId }: { workspaceId: string }) {
  const qc = useQueryClient();
  const { data, fetchNextPage, hasNextPage } = useInfiniteQuery({
    queryKey: ["kb-docs", workspaceId],
    queryFn: async ({ pageParam }) => {
      const res = await fetch(`/api/kb/documents?cursor=${pageParam ?? ""}`);
      return res.json();
    },
    initialPageParam: null as string | null,
    getNextPageParam: (l) => l.nextCursor,
  });

  const del = useMutation({
    mutationFn: (id: string) => fetch(`/api/kb/documents/${id}`, { method: "DELETE" }),
    onSuccess: () => qc.invalidateQueries({ queryKey: ["kb-docs"] }),
  });

  return (
    <div>
      <table className="w-full">
        <thead><tr><th>File</th><th>Status</th><th>Chunks</th><th>Uploaded</th><th /></tr></thead>
        <tbody>
          {data?.pages.flatMap((p) => p.docs).map((d) => (
            <tr key={d.id} className="border-t">
              <td className="font-medium">{d.filename}</td>
              <td><StatusBadge status={d.status} /></td>
              <td>{d.chunks}</td>
              <td>{new Date(d.created_at).toLocaleDateString()}</td>
              <td><Button variant="ghost" size="sm" onClick={() => del.mutate(d.id)}>Delete</Button></td>
            </tr>
          ))}
        </tbody>
      </table>
      {hasNextPage && <Button onClick={() => fetchNextPage()}>Load more</Button>}
    </div>
  );
}

function StatusBadge({ status }: { status: string }) {
  const styles: Record<string, string> = {
    uploaded: "bg-blue-100 text-blue-800",
    embedding: "bg-yellow-100 text-yellow-800",
    ready: "bg-green-100 text-green-800",
    failed: "bg-red-100 text-red-800",
  };
  return <span className={`rounded px-2 py-0.5 text-xs ${styles[status] ?? ""}`}>{status}</span>;
}
```

The three states a row can be in (`uploaded`, `embedding`, `ready`, `failed`) map to the worker's progress events. Use Server-Sent Events or polling to refresh the status when a job is running.

### 2. Agent run history and traces

Each agent run has a `thread_id`. We persist intermediate state into a JSONB column so the dashboard can render the timeline without replaying.

```sql
CREATE TABLE agent_runs (
  id          UUID PRIMARY KEY,
  workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  user_id     UUID NOT NULL,
  thread_id   TEXT NOT NULL,
  question    TEXT NOT NULL,
  answer      TEXT,
  status      TEXT NOT NULL,        -- 'running' | 'complete' | 'failed' | 'awaiting_approval'
  trace       JSONB NOT NULL DEFAULT '[]'::jsonb,
  tokens      INT NOT NULL DEFAULT 0,
  created_at  TIMESTAMPTZ DEFAULT now(),
  completed_at TIMESTAMPTZ
);
CREATE INDEX agent_runs_ws_created_idx ON agent_runs (workspace_id, created_at DESC);
```

The trace is appended by the worker:

```ts
// src/lib/agents/tracer.ts
export async function recordStep(threadId: string, step: { type: string; payload: unknown }) {
  await db.query(
    "UPDATE agent_runs SET trace = trace || $1::jsonb WHERE thread_id = $2",
    [JSON.stringify([{ ...step, ts: new Date().toISOString() }]), threadId]
  );
}
```

The dashboard renders it as a vertical timeline:

```tsx
// src/components/agents/run-timeline.tsx
export function RunTimeline({ trace }: { trace: TraceStep[] }) {
  return (
    <ol className="border-l pl-4 space-y-3">
      {trace.map((s, i) => (
        <li key={i} className="relative">
          <span className="absolute -left-[1.45rem] top-1.5 h-2 w-2 rounded-full bg-primary" />
          <div className="font-medium">{s.type}</div>
          <pre className="text-xs bg-muted rounded p-2 overflow-x-auto">{JSON.stringify(s.payload, null, 2)}</pre>
          <div className="text-xs text-muted-foreground">{new Date(s.ts).toLocaleTimeString()}</div>
        </li>
      ))}
    </ol>
  );
}
```

The detail page:

```tsx
// src/app/app/[workspace]/agents/[runId]/page.tsx
export default async function RunPage({ params }: { params: { workspace: string; runId: string } }) {
  const run = await db.query("SELECT * FROM agent_runs WHERE id = $1", [params.runId]);
  if (!run.rowCount) return notFound();
  const r = run.rows[0];
  return (
    <div className="grid lg:grid-cols-3 gap-4 p-4">
      <Card className="lg:col-span-2">
        <h2 className="font-medium">Question</h2>
        <p>{r.question}</p>
        <h2 className="font-medium mt-4">Answer</h2>
        <p>{r.answer}</p>
      </Card>
      <Card>
        <h2 className="font-medium">Trace</h2>
        <RunTimeline trace={r.trace} />
      </Card>
    </div>
  );
}
```

### 3. Usage & cost analytics

The data:

```ts
// src/lib/billing/usage.ts
export async function usageByDay(workspaceId: string, days = 30) {
  const rows = await db.query(
    `SELECT date_trunc('day', created_at) AS day,
            SUM(tokens)::int AS tokens,
            COUNT(*)::int AS requests,
            AVG(latency_ms)::int AS avg_latency_ms
       FROM llm_call_log
      WHERE workspace_id = $1 AND created_at > now() - ($2 || ' days')::interval
      GROUP BY 1
      ORDER BY 1`,
    [workspaceId, days]
  );
  return rows.rows;
}
```

The visualization (Recharts):

```tsx
// src/components/billing/usage-chart.tsx
"use client";
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip } from "recharts";

export function UsageChart({ data }: { data: { day: string; tokens: number; requests: number }[] }) {
  return (
    <LineChart width={600} height={240} data={data}>
      <CartesianGrid strokeDasharray="3 3" />
      <XAxis dataKey="day" />
      <YAxis />
      <Tooltip />
      <Line type="monotone" dataKey="tokens" stroke="#8884d8" />
      <Line type="monotone" dataKey="requests" stroke="#82ca9d" />
    </LineChart>
  );
}
```

The header summary:

```tsx
// src/components/billing/usage-summary.tsx
export function UsageSummary({ workspace, usage }: { workspace: Workspace; usage: UsageStats }) {
  const pct = (usage.tokensThisMonth / workspace.planLimit) * 100;
  return (
    <Card className="p-4">
      <div className="flex items-center justify-between">
        <div>
          <div className="text-sm text-muted-foreground">AI credits this month</div>
          <div className="text-2xl font-semibold">
            {usage.tokensThisMonth.toLocaleString()} / {workspace.planLimit.toLocaleString()}
          </div>
        </div>
        <div className="text-right">
          <div className="text-sm text-muted-foreground">Cost (est.)</div>
          <div className="text-2xl font-semibold">${usage.estimatedCost.toFixed(2)}</div>
        </div>
      </div>
      <Progress value={pct} className="mt-2" />
      {pct > 80 && <p className="text-sm text-amber-600 mt-2">You've used {pct.toFixed(0)}% of your credits. Consider upgrading.</p>}
    </Card>
  );
}
```

### 4. Feature gating and quota UI

Two surfaces: the API gate and the UI gate.

The API gate (re-usable helper):

```ts
// src/lib/billing/quota.ts
export async function requireQuota(workspaceId: string) {
  const { tokensThisMonth, planLimit } = await usageTracker.assertBudget(workspaceId, planLimitFor(workspaceId));
  return { remaining: planLimit - tokensThisMonth, limit: planLimit };
}
```

The UI gate — show, don't hide:

```tsx
// src/components/chat/upgrade-prompt.tsx
export function UpgradePrompt({ remaining, limit }: { remaining: number; limit: number }) {
  if (remaining > limit * 0.2) return null;
  return (
    <div className="rounded border border-amber-300 bg-amber-50 p-3 text-amber-900">
      You're running low on AI credits ({remaining.toLocaleString()} left).{" "}
      <Link href="/settings/billing" className="underline font-medium">Upgrade</Link>
    </div>
  );
}
```

The principle: **show a soft warning at 80%, a hard block at 100%.** Don't surprise the user with a silent refusal at the moment they hit send.

### 5. Accessibility and mobile polish

#### Keyboard shortcuts

```tsx
// src/components/layout/keyboard-shortcuts.tsx
"use client";
import { useHotkeys } from "react-hotkeys-hook";

export function useAppShortcuts() {
  useHotkeys("mod+k", (e) => { e.preventDefault(); document.getElementById("cmd-k")?.focus(); });
  useHotkeys("mod+/", (e) => { e.preventDefault(); (document.querySelector("textarea") as HTMLTextAreaElement)?.focus(); });
  useHotkeys("escape", () => (document.activeElement as HTMLElement)?.blur());
}
```

#### ARIA labels

```tsx
<button aria-label="Stop generating" onClick={onStop}>Stop</button>
<div role="status" aria-live="polite">{loading && "Loading knowledge base…"}</div>
<table role="grid" aria-label="Knowledge base documents">…</table>
```

The four ARIA attributes that do 80% of the work: `aria-label`, `aria-live`, `aria-busy`, `aria-describedby`. Sprinkle them as you build, not after.

#### Mobile responsiveness

The dashboard tables do not work on a phone. Use a `<details>`-based card layout below the `sm` breakpoint:

```tsx
<div className="hidden sm:block">
  <table>…</table>
</div>
<div className="sm:hidden space-y-2">
  {docs.map((d) => (
    <details key={d.id} className="rounded border p-3">
      <summary className="font-medium">{d.filename}</summary>
      <div className="mt-2 text-sm">Status: {d.status}</div>
      <div className="text-sm">Chunks: {d.chunks}</div>
    </details>
  ))}
</div>
```

The pattern: same data, two layouts. Don't hide functionality on mobile; reshape it.

#### Performance polish

A few small things that add up:

- **Prefetch on hover.** `useQuery({ enabled: false })` + `queryClient.prefetchQuery` on the link's `onMouseEnter`.
- **Suspense boundaries.** Wrap each tab in `<Suspense fallback={<LoadingState />}>` so the user sees the page shell immediately.
- **`loading.tsx` per route.** Next.js renders the file `loading.tsx` in the same segment while the page loads. One file, two seconds of perceived performance.

## Common mistakes

1. **Hiding features when quota is low.** Soft warnings, not silent failures. Tell the user *why* and *what to do*.
2. **No empty states.** A dashboard with zero documents is the moment the user is most likely to leave. An empty state with "Upload your first PDF" beats a blank table.
3. **Tables with 30 columns.** They don't fit a laptop. Either paginate or move secondary fields behind a `<details>`.
4. **No loading state.** A dashboard that flickers "no data" → "data" is broken. Use skeletons.
5. **Forgetting mobile.** Half your users will see this on a phone. Build responsive from day one.
6. **Cost dashboard that doesn't match the bill.** If the dashboard says $312 and the invoice says $400, the user notices. Reconcile monthly.
7. **Inaccessible modals.** A modal without a focus trap and `Esc`-to-close is a usability bug, not a polish item.
8. **Keyboard shortcuts without help text.** A `Cmd+K` users can't discover is a `Cmd+K` users won't use. Always include a "?" overlay listing them.

## Exercise

Add the AI dashboard to your SaaS.

**Acceptance criteria**

1. A `/app/[workspace]/kb` page lists documents with status, chunk count, upload date, and a Delete action that confirms then removes.
2. A `/app/[workspace]/agents` page lists past agent runs; clicking one opens a detail view with the question, the answer, and the full trace timeline.
3. A `/app/[workspace]/usage` page shows a 30-day token/requests chart and a summary card with current usage vs. limit.
4. When usage > 80%, an amber banner appears on the chat page with a link to upgrade.
5. Every async surface (lists, charts, detail views) has a skeleton loading state and an error retry.
6. The dashboard is keyboard-navigable; `Cmd+K` opens a command palette; `Tab` cycles through actionable controls.
7. The layout reflows to a card view below 640px width without losing functionality.

**Stretch**

- Add a CSV export of usage data (Module 4's pattern).
- Add a "Schedule recurring run" form that enqueues an agent job every Monday at 9am using BullMQ's repeatable jobs.

## AI Industry Note (2026)

In 2026, the dashboard is a *product surface*, not an afterthought. The teams shipping the best AI products are the teams that treat observability and quota UX as features in their own right.

- **Trace UIs are now table stakes.** LangSmith, Helicone, Langfuse, and Arize Phoenix are the four most-installed internal tools in AI teams. If your dashboard does not show the trace, you will be asked "why does LangSmith show X and you show Y?"
- **Cost dashboards are a sales tool.** The fastest way to expand an enterprise account is to put a dashboard in front of the IT buyer that they did not have to ask for. "You spent $4,200 on AI this quarter, here is what you got" closes renewals.
- **Usage-based pricing is the new seat-based pricing.** The number of AI products that bill purely per seat in 2026 is small. Most bill on tokens, requests, or compute-minutes. The dashboard is where the customer sees what they're buying.
- **Self-serve quota upgrades.** The "Upgrade" button on the quota warning banner is, on average, the highest-converting button in the product. Make it obvious.
- **Accessibility laws are catching up.** EU AI Act, California's SB-1001, and the US Section 508 updates now require AI features to disclose AI-generated content and be accessible. A non-accessible dashboard is a legal liability, not a polish item.
- **Mobile dashboards are the new battleground.** A surprising amount of AI-feature usage happens on the phone (sales reps, executives, on-call engineers). The 2026 dashboard is mobile-first; the desktop view is a bonus.