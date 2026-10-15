# Module 35: Charts, Analytics & Reporting UI

> Mission 4 · Part B · Frontend — Analytics is where a SaaS proves its value. If the dashboard is slow or the numbers are wrong, your users churn. Build it right.

## Why this matters

Every founder, PM, and ops person who logs into your app will spend a meaningful chunk of their week on your analytics page. It's the page they screenshot for the board. It's the page they open in their Monday standup. And it has to be fast — users will refresh it constantly, often right after they make a change elsewhere in the app to "see if it worked."

The performance bar is high: **interactive in under 1 second on first paint**, **incremental loads for date changes**, **server-rendered aggregates** so the first paint isn't a blank chart, and a UI that answers the question "is this number up or down from last month?" without making the user do math.

In this module you'll build a complete analytics page with Recharts, server-rendered aggregates, suspenseful streaming for slow queries, and CSV/PDF export.

## Learning objectives

By the end of this module you will be able to:

1. Build dashboards with **Recharts** (Line, Bar, Area, Pie).
2. Compute **server-side aggregates** and stream them to the client with Suspense.
3. Render **stat cards** with trend indicators and sparklines.
4. Implement a **date-range picker** with presets (7d, 30d, MTD, QTD, custom).
5. Export data to **CSV** (client-side with `papaparse`) and **PDF** (server-side).
6. Cache analytics queries with **unstable_cache** and incremental revalidation.

## Conceptual model

### 1. The shape of an analytics query

Most analytics pages are answering one of three questions:

- **How many?** (`COUNT(*)` filtered by date range)
- **How are they distributed?** (`GROUP BY status`, `GROUP BY owner`)
- **How is it changing over time?** (`GROUP BY date_trunc('day', created_at)`)

The server should precompute these. The client should never receive raw rows when an aggregate will do. The page that ships a 100k-row JSON blob "so the chart can compute its own sums" is a page I've seen in production, and it cost the team a week of performance work.

**`services/analytics.ts`** — server-side aggregation:

```typescript
import { db } from "@/lib/db";
import { unstable_cache } from "next/cache";
import { startOfDay, subDays, startOfMonth, startOfQuarter } from "date-fns";

export type DateRange = { from: Date; to: Date };

export function rangeFromPreset(preset: string, now = new Date()): DateRange {
  switch (preset) {
    case "7d":  return { from: subDays(now, 7), to: now };
    case "30d": return { from: subDays(now, 30), to: now };
    case "mtd": return { from: startOfMonth(now), to: now };
    case "qtd": return { from: startOfQuarter(now), to: now };
    case "90d": return { from: subDays(now, 90), to: now };
    default:    return { from: subDays(now, 30), to: now };
  }
}

export const getTasksOverTime = unstable_cache(
  async (projectId: string, range: DateRange) => {
    const rows = await db.$queryRaw<{ day: Date; count: bigint }[]>`
      SELECT date_trunc('day', "createdAt") AS day, COUNT(*)::bigint AS count
      FROM "Task"
      WHERE "projectId" = ${projectId}
        AND "createdAt" BETWEEN ${range.from} AND ${range.to}
      GROUP BY 1
      ORDER BY 1;
    `;
    // backfill zero-count days so the chart is contiguous
    const map = new Map(rows.map((r) => [r.day.toISOString().slice(0, 10), Number(r.count)]));
    const out: { day: string; count: number }[] = [];
    for (let d = startOfDay(range.from); d <= range.to; d = new Date(d.getTime() + 86400000)) {
      const key = d.toISOString().slice(0, 10);
      out.push({ day: key, count: map.get(key) ?? 0 });
    }
    return out;
  },
  ["tasks-over-time"],
  { revalidate: 60, tags: ["analytics"] }
);

export const getStatusDistribution = unstable_cache(
  async (projectId: string) => {
    return db.task.groupBy({
      by: ["status"],
      where: { projectId },
      _count: { _all: true },
    });
  },
  ["status-distribution"],
  { revalidate: 60, tags: ["analytics"] }
);

export const getTopContributors = unstable_cache(
  async (projectId: string, range: DateRange) => {
    return db.task.groupBy({
      by: ["assigneeId"],
      where: { projectId, createdAt: { gte: range.from, lte: range.to } },
      _count: { _all: true },
      orderBy: { _count: { assigneeId: "desc" } },
      take: 5,
    });
  },
  ["top-contributors"],
  { revalidate: 60, tags: ["analytics"] }
);
```

`unstable_cache` is the Next.js primitive for memoizing a server function. Tag it with `analytics` and you can `revalidateTag("analytics")` from a server action when new data lands.

### 2. Streaming the page with Suspense

Don't make users wait for the slowest query. Each chart can be its own Suspense boundary, so the rest of the page paints while the heavy one is computing.

**`app/(app)/projects/[id]/analytics/page.tsx`**

```tsx
import { Suspense } from "react";
import { Skeleton } from "@/components/ui/skeleton";
import { TasksOverTimeChart } from "./_components/tasks-over-time-chart";
import { StatusPieChart } from "./_components/status-pie-chart";
import { TopContributorsList } from "./_components/top-contributors-list";
import { getTasksOverTime, getStatusDistribution, getTopContributors, rangeFromPreset } from "@/services/analytics";
import { DateRangePicker } from "./_components/date-range-picker";

export default async function AnalyticsPage({
  params,
  searchParams,
}: {
  params: Promise<{ id: string }>;
  searchParams: Promise<{ preset?: string; from?: string; to?: string }>;
}) {
  const { id } = await params;
  const sp = await searchParams;
  const range = sp.from && sp.to
    ? { from: new Date(sp.from), to: new Date(sp.to) }
    : rangeFromPreset(sp.preset ?? "30d");

  return (
    <div className="space-y-6 p-6">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-semibold">Analytics</h1>
        <DateRangePicker initial={range} preset={sp.preset ?? "30d"} />
      </div>

      <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
        <div className="lg:col-span-2">
          <Suspense fallback={<ChartSkeleton />}>
            <TasksOverTimeChartWrapper projectId={id} range={range} />
          </Suspense>
        </div>
        <Suspense fallback={<ChartSkeleton />}>
          <StatusPieChartWrapper projectId={id} />
        </Suspense>
      </div>

      <Suspense fallback={<ChartSkeleton />}>
        <TopContributorsListWrapper projectId={id} range={range} />
      </Suspense>
    </div>
  );
}

// Server components that await the queries
async function TasksOverTimeChartWrapper({ projectId, range }: any) {
  const data = await getTasksOverTime(projectId, range);
  return <TasksOverTimeChart data={data} />;
}
async function StatusPieChartWrapper({ projectId }: { projectId: string }) {
  const data = await getStatusDistribution(projectId);
  return <StatusPieChart data={data} />;
}
async function TopContributorsListWrapper({ projectId, range }: any) {
  const data = await getTopContributors(projectId, range);
  return <TopContributorsList data={data} />;
}

function ChartSkeleton() {
  return <Skeleton className="h-72 w-full" />;
}
```

Each `<Suspense>` boundary streams independently. The first chart paints first, then the pie chart streams in, then the contributors list. The user sees a fully laid-out page immediately, with placeholders filling in.

### 3. Charts with Recharts

**`app/(app)/projects/[id]/analytics/_components/tasks-over-time-chart.tsx`**

```tsx
"use client";

import {
  Area,
  AreaChart,
  CartesianGrid,
  ResponsiveContainer,
  Tooltip,
  XAxis,
  YAxis,
} from "recharts";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

export function TasksOverTimeChart({
  data,
}: {
  data: { day: string; count: number }[];
}) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>Tasks created</CardTitle>
      </CardHeader>
      <CardContent className="h-72">
        <ResponsiveContainer width="100%" height="100%">
          <AreaChart data={data} margin={{ left: 0, right: 8, top: 8, bottom: 0 }}>
            <defs>
              <linearGradient id="fill" x1="0" y1="0" x2="0" y2="1">
                <stop offset="5%" stopColor="hsl(var(--primary))" stopOpacity={0.4} />
                <stop offset="95%" stopColor="hsl(var(--primary))" stopOpacity={0} />
              </linearGradient>
            </defs>
            <CartesianGrid strokeDasharray="3 3" className="stroke-muted" />
            <XAxis
              dataKey="day"
              tickFormatter={(d) => new Date(d).toLocaleDateString(undefined, { month: "short", day: "numeric" })}
              className="text-xs"
            />
            <YAxis allowDecimals={false} className="text-xs" />
            <Tooltip
              labelFormatter={(d) => new Date(d).toLocaleDateString()}
              contentStyle={{
                background: "hsl(var(--background))",
                border: "1px solid hsl(var(--border))",
                borderRadius: 8,
              }}
            />
            <Area
              type="monotone"
              dataKey="count"
              stroke="hsl(var(--primary))"
              fill="url(#fill)"
              strokeWidth={2}
            />
          </AreaChart>
        </ResponsiveContainer>
      </CardContent>
    </Card>
  );
}
```

**`status-pie-chart.tsx`**

```tsx
"use client";

import { Cell, Pie, PieChart, ResponsiveContainer, Tooltip } from "recharts";

const COLORS = ["#22c55e", "#3b82f6", "#f59e0b", "#ef4444", "#8b5cf6"];

export function StatusPieChart({
  data,
}: {
  data: { status: string; _count: { _all: number } }[];
}) {
  const chart = data.map((d) => ({ name: d.status, value: d._count._all }));
  return (
    <Card>
      <CardHeader><CardTitle>Status</CardTitle></CardHeader>
      <CardContent className="h-72">
        <ResponsiveContainer width="100%" height="100%">
          <PieChart>
            <Pie data={chart} dataKey="value" nameKey="name" innerRadius={50} outerRadius={80}>
              {chart.map((_, i) => <Cell key={i} fill={COLORS[i % COLORS.length]} />)}
            </Pie>
            <Tooltip />
          </PieChart>
        </ResponsiveContainer>
      </CardContent>
    </Card>
  );
}
```

Recharts takes a little getting used to: it's `<ResponsiveContainer>`-wrapped, expects `data` and `dataKey`, and most props (stroke, fill) accept CSS variables. The result is pretty, but it isn't a *fast* library at the extreme end. For >10k points, consider `visx` or `apache-echarts`. For 99% of dashboards, Recharts is fine.

### 4. Stat cards with trend indicators and sparklines

A "stat card" is the unit of an analytics page: a big number, a label, a small comparison, and a sparkline.

**`components/ui/stat-card.tsx`**

```tsx
import { ArrowDown, ArrowUp } from "lucide-react";
import { cn } from "@/lib/utils";
import { ResponsiveContainer, LineChart, Line } from "recharts";

export function StatCard({
  label,
  value,
  delta,        // { current, previous, pct }
  sparkline,    // number[]
}: {
  label: string;
  value: string | number;
  delta?: { current: number; previous: number; pct: number };
  sparkline?: number[];
}) {
  const positive = (delta?.pct ?? 0) >= 0;
  return (
    <div className="rounded-lg border bg-card p-5">
      <div className="text-sm text-muted-foreground">{label}</div>
      <div className="mt-1 flex items-baseline justify-between">
        <div className="text-2xl font-semibold tabular-nums">{value}</div>
        {delta && (
          <div className={cn(
            "flex items-center gap-1 text-xs",
            positive ? "text-emerald-600" : "text-rose-600"
          )}>
            {positive ? <ArrowUp className="h-3 w-3" /> : <ArrowDown className="h-3 w-3" />}
            {Math.abs(delta.pct).toFixed(1)}%
          </div>
        )}
      </div>
      {sparkline && (
        <div className="mt-3 h-10">
          <ResponsiveContainer>
            <LineChart data={sparkline.map((v, i) => ({ i, v }))}>
              <Line
                type="monotone"
                dataKey="v"
                stroke="hsl(var(--primary))"
                strokeWidth={1.5}
                dot={false}
              />
            </LineChart>
          </ResponsiveContainer>
        </div>
      )}
      {delta && (
        <div className="mt-1 text-xs text-muted-foreground">
          vs {delta.previous.toLocaleString()} last period
        </div>
      )}
    </div>
  );
}
```

Compute the delta server-side:

```typescript
export async function getStatWithDelta(projectId: string, range: DateRange) {
  const ms = range.to.getTime() - range.from.getTime();
  const prev = { from: new Date(range.from.getTime() - ms), to: range.from };

  const [current, previous] = await Promise.all([
    db.task.count({ where: { projectId, createdAt: range } }),
    db.task.count({ where: { projectId, createdAt: prev } }),
  ]);

  const pct = previous === 0 ? 100 : ((current - previous) / previous) * 100;
  return { current, previous, pct };
}
```

### 5. Date-range picker with presets

shadcn has a calendar primitive; pair it with a small preset selector. The picker updates the URL — `?preset=30d` or `?from=2026-01-01&to=2026-01-31`.

**`date-range-picker.tsx`** (sketch — full code in the shadcn examples):

```tsx
"use client";

import { useRouter } from "next/navigation";
import { Button } from "@/components/ui/button";
import { Popover, PopoverContent, PopoverTrigger } from "@/components/ui/popover";
import { Calendar } from "@/components/ui/calendar";
import { addDays, format, startOfMonth, startOfQuarter } from "date-fns";

const PRESETS: { label: string; getValue: () => { from: Date; to: Date } }[] = [
  { label: "Last 7 days",  getValue: () => ({ from: addDays(new Date(), -7),  to: new Date() }) },
  { label: "Last 30 days", getValue: () => ({ from: addDays(new Date(), -30), to: new Date() }) },
  { label: "Month to date",getValue: () => ({ from: startOfMonth(new Date()), to: new Date() }) },
  { label: "Quarter to date", getValue: () => ({ from: startOfQuarter(new Date()), to: new Date() }) },
];

export function DateRangePicker({ initial, preset }: { initial: { from: Date; to: Date }; preset: string }) {
  const router = useRouter();

  function apply(presetName: string, range?: { from: Date; to: Date }) {
    const sp = new URLSearchParams();
    if (range) {
      sp.set("from", format(range.from, "yyyy-MM-dd"));
      sp.set("to", format(range.to, "yyyy-MM-dd"));
    } else {
      sp.set("preset", presetName);
    }
    router.push(`?${sp.toString()}`);
  }

  return (
    <Popover>
      <PopoverTrigger asChild>
        <Button variant="outline">
          {preset !== "custom"
            ? PRESETS.find((p) => p.label.toLowerCase().startsWith(preset))?.label ?? "Custom"
            : `${format(initial.from, "MMM d")} – ${format(initial.to, "MMM d")}`}
        </Button>
      </PopoverTrigger>
      <PopoverContent className="w-auto p-0" align="end">
        <div className="grid grid-cols-[180px_1fr]">
          <div className="border-r p-2 space-y-1">
            {PRESETS.map((p) => (
              <Button
                key={p.label}
                variant="ghost"
                size="sm"
                className="w-full justify-start"
                onClick={() => apply(p.label.toLowerCase().split(" ")[0])}
              >
                {p.label}
              </Button>
            ))}
          </div>
          <Calendar
            mode="range"
            selected={{ from: initial.from, to: initial.to }}
            onSelect={(r) => r?.from && r?.to && apply("custom", r)}
            numberOfMonths={2}
          />
        </div>
      </PopoverContent>
    </Popover>
  );
}
```

### 6. CSV export (client-side, papaparse)

```tsx
import Papa from "papaparse";

export function downloadCSV<T extends Record<string, unknown>>(rows: T[], filename: string) {
  const csv = Papa.unparse(rows);
  const blob = new Blob([csv], { type: "text/csv;charset=utf-8;" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = filename;
  a.click();
  URL.revokeObjectURL(url);
}
```

For an export button that does its own fetch:

```tsx
async function exportTasksCSV(projectId: string, range: DateRange) {
  const res = await fetch(`/api/projects/${projectId}/export?from=${range.from.toISOString()}&to=${range.to.toISOString()}`);
  const rows = await res.json();
  downloadCSV(rows, `tasks-${new Date().toISOString().slice(0, 10)}.csv`);
}
```

Server route that streams the CSV without holding it all in memory:

```typescript
// app/api/projects/[id]/export/route.ts
export async function GET(req: Request, { params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const url = new URL(req.url);
  const from = new Date(url.searchParams.get("from")!);
  const to = new Date(url.searchParams.get("to")!);

  const stream = new ReadableStream({
    async start(controller) {
      const enc = new TextEncoder();
      controller.enqueue(enc.encode("id,title,status,createdAt\n"));
      for await (const t of db.$queryRawStream<any>`
        SELECT id, title, status, "createdAt"
        FROM "Task"
        WHERE "projectId" = ${id} AND "createdAt" BETWEEN ${from} AND ${to}
      `) {
        controller.enqueue(enc.encode(`${t.id},"${t.title}",${t.status},${t.createdAt}\n`));
      }
      controller.close();
    },
  });

  return new Response(stream, {
    headers: {
      "Content-Type": "text/csv",
      "Content-Disposition": `attachment; filename="tasks.csv"`,
    },
  });
}
```

For exports up to ~50k rows, client-side papaparse is fine. Beyond that, server stream is the only way to keep memory bounded.

### 7. PDF export (server-side, pdfkit or puppeteer)

Two strategies:

- **Generate from HTML** (puppeteer/playwright): pixel-perfect, but heavyweight. Spin up a headless browser, navigate to `/print/projects/[id]`, print to PDF. Use for invoices, contracts, anything pretty.
- **Programmatic** (pdfkit or `@react-pdf/renderer`): lighter, but you rebuild the layout in code. Use for receipts, reports.

For a "shareable report" feature, **headless Chrome on a route marked for print** is the path of least resistance:

```typescript
// app/api/reports/[id]/pdf/route.ts
import { launchBrowser } from "@/lib/browser";

export async function GET(_req: Request, { params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const browser = await launchBrowser();
  const page = await browser.newPage();
  await page.goto(`${process.env.APP_URL}/projects/${id}/report?print=1`, {
    waitUntil: "networkidle0",
  });
  const pdf = await page.pdf({ format: "A4", printBackground: true });
  await browser.close();
  return new Response(pdf, {
    headers: {
      "Content-Type": "application/pdf",
      "Content-Disposition": `attachment; filename="report-${id}.pdf"`,
    },
  });
}
```

The route `/projects/[id]/report?print=1` is just your dashboard with print stylesheets and no nav. The browser renders it, puppeteer prints it.

## Common mistakes

- **Shipping raw rows.** A dashboard that aggregates on the client will be slow, period. Aggregate on the server.
- **No streaming.** A page that awaits every query serially will be 4× slower than the slowest query. Use `<Suspense>` per chart.
- **Recharts inside a `useEffect` data fetch.** Recharts is client-only. If your data is small enough, pass it as props from a server component — the chart becomes a "view" with no fetch of its own.
- **Forgetting the empty state.** "Tasks created" with zero data is just an empty chart — confusing. Add a meaningful empty state per chart.
- **Number formatting with `toFixed` in the wrong place.** Currency and percentages need `Intl.NumberFormat`, not `toFixed`. `new Intl.NumberFormat("en-US", { style: "currency", currency: "USD" }).format(1234.5)` → "$1,234.50".
- **Unbounded CSV exports.** Without `queryRawStream` or pagination, exporting "all tasks" can OOM your server.
- **Charts that re-render on every parent update.** Wrap them in `React.memo` if they're expensive.

## Exercise

Build a **/projects/[id]/analytics** page with:

1. **Stat cards** at the top: Tasks created, Tasks completed, Avg time-to-complete, Active members. Each with delta vs previous period and a sparkline.
2. **Tasks-over-time** area chart (server-aggregated, streamed via Suspense).
3. **Status distribution** pie chart.
4. **Top 5 contributors** list with their counts.
5. A **date-range picker** that supports 7d, 30d, MTD, QTD, and a custom calendar range. State lives in the URL.
6. An **"Export CSV"** button that streams the underlying rows.
7. An **"Export PDF"** button that renders the dashboard in headless Chrome and downloads the PDF.

Bonus: a **comparison view** that overlays the previous period as a dashed line on the time-series chart. Bonus 2: a **forecast** that uses a simple linear regression to project next 7 days.

## AI Industry Note (2026)

In 2026, analytics pages are increasingly **AI-explained**. The pattern: you ship the charts, and an LLM sits in a sidebar that can answer "why did signups drop last Tuesday?" by reading the underlying data and producing a written summary. **Vercel AI SDK + tool calling** is the stack: you expose `getTasksOverTime`, `getStatusDistribution`, etc. as Zod-typed tools, and the LLM picks the right ones, calls them, and synthesizes a paragraph. **Hex.tech**, **Suma**, and **Fabric** are the commercial templates — the patterns they use are the ones you should study.

There's also a fast-growing category of **chart-from-text** features: the user types "show me churn by plan tier for Q1", your server validates it against a Zod grammar of supported queries, and renders the result. Treat your analytics primitives as **tools**, not pages — that's the design principle that makes the AI features cheap to add later.
