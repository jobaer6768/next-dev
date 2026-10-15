# Module 30 — Server Components, Data Fetching & Server Actions

> The data layer that makes Next.js feel like a backend-with-UI.

---

## Why this module matters

In Module 29 you learned *where* routes live. In this module you learn *how data flows through them*. Server Components flip the default from "fetch in `useEffect`" to "fetch at the server, ship HTML to the browser." The result is faster, simpler, and harder to get wrong.

By the end of this module, the Part A backend will be talking to the Part B frontend with typed data, server-side auth, and zero client-side fetching boilerplate.

---

## Learning objectives

- Decide server vs client component correctly.
- Implement static, dynamic, and streaming data fetching.
- Use caching, revalidation, and `revalidateTag` correctly.
- Build API routes in Next.js (route handlers).
- Write server actions for forms.
- Connect the Next.js app to the Part A backend securely.

---

## 1. Server vs client — the decision rule (deep dive)

| Need | Server component | Client component |
|------|------------------|------------------|
| Async data access | ✅ | ❌ (must use `useEffect` or TanStack Query) |
| Sensitive tokens | ✅ (only on server) | ❌ (visible in JS bundle) |
| `useState`, `useEffect` | ❌ | ✅ |
| Event handlers (`onClick`) | ❌ | ✅ |
| Browser-only APIs (`window`, `localStorage`) | ❌ | ✅ |
| Big data lists | ✅ (HTML only) | ❌ (must serialize JSON) |
| Animations, focus management | ❌ | ✅ |
| SEO content | ✅ | ⚠️ (delays render) |

**Default to server components. Add `"use client"` only when you need interactivity.**

A common pattern: a server component fetches data, then passes it as props to a client component for interactive rendering (forms, modals, etc.).

```tsx
// page.tsx (server)
export default async function Page() {
  const data = await fetchData();
  return <InteractiveTable rows={data} />;   // "use client" inside
}
```

---

## 2. Data fetching strategies

### Static (build time)

```tsx
async function getPlans() {
  const res = await fetch("https://api.example.com/plans", { cache: "force-cache" });
  return res.json();
}
```

`cache: "force-cache"` (default) caches indefinitely. Good for content that rarely changes (marketing pages, pricing).

### Dynamic (per request)

```tsx
async function getMyProjects() {
  const res = await fetch("https://api.example.com/projects", { cache: "no-store" });
  return res.json();
}
```

`cache: "no-store"` always fetches. Use for user-specific data that must be fresh.

### Revalidated (ISR-style)

```tsx
async function getPosts() {
  const res = await fetch("https://api.example.com/posts", { next: { revalidate: 60 } });
  return res.json();
}
```

Refetches every 60 seconds. Good for content that changes often but can tolerate a minute of staleness.

### Tagged (manual invalidation)

```tsx
async function getProject(id: string) {
  const res = await fetch(`https://api.example.com/projects/${id}`, {
    next: { tags: [`project-${id}`] },
  });
  return res.json();
}
```

Then from a server action:

```tsx
import { revalidateTag } from "next/cache";
revalidateTag(`project-${id}`);
```

This is the most flexible strategy. We will use it heavily in Mission 4.

---

## 3. Streaming with Suspense

```tsx
// app/(dashboard)/analytics/page.tsx
import { Suspense } from "react";
import { RevenueChart } from "@/components/analytics/revenue-chart";
import { SignupsTable } from "@/components/analytics/signups-table";
import { Skeleton } from "@/components/ui/skeleton";

export default function AnalyticsPage() {
  return (
    <div className="grid grid-cols-2 gap-6">
      <Suspense fallback={<Skeleton className="h-80" />}>
        <RevenueChart />
      </Suspense>
      <Suspense fallback={<Skeleton className="h-80" />}>
        <SignupsTable />
      </Suspense>
    </div>
  );
}
```

Each `<Suspense>` boundary streams independently. The page becomes interactive the moment the first chunk lands; slow queries don't block the page.

```tsx
// RevenueChart.tsx (server component)
async function RevenueChart() {
  const data = await fetchRevenue();    // slow query
  return <Chart data={data} />;
}
```

---

## 4. Route handlers (API routes in App Router)

```ts
// src/app/api/projects/route.ts
import { NextResponse } from "next/server";
import { listProjects } from "@/services/projects";

export async function GET() {
  try {
    const projects = await listProjects();
    return NextResponse.json({ data: projects });
  } catch (err) {
    return NextResponse.json(
      { error: { code: "INTERNAL", message: "internal error" } },
      { status: 500 },
    );
  }
}

export async function POST(req: Request) {
  const body = await req.json();
  // validate with Zod, call service, etc.
  return NextResponse.json({ data: created }, { status: 201 });
}
```

You can also use **dynamic segments**:

```ts
// src/app/api/projects/[id]/route.ts
export async function GET(_: Request, { params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const p = await getProject(id);
  if (!p) return NextResponse.json({ error: { code: "NOT_FOUND" } }, { status: 404 });
  return NextResponse.json({ data: p });
}
```

For our SaaS, **most API calls go directly to the Part A backend**; route handlers are useful for things that need to live in the same Node process (webhooks, file proxies, AI streaming endpoints).

---

## 5. Server Actions — forms without an API layer

Server Actions are async functions that run on the server and can be called directly from a form.

```tsx
// src/app/(dashboard)/projects/new/actions.ts
"use server";
import { revalidatePath } from "next/cache";
import { redirect } from "next/navigation";
import { z } from "zod";
import { requireUser } from "@/lib/auth";
import { createProject } from "@/services/projects";

const Schema = z.object({ name: z.string().min(1).max(100) });

export async function createProjectAction(formData: FormData) {
  const user = await requireUser();
  const parsed = Schema.safeParse({ name: formData.get("name") });
  if (!parsed.success) {
    return { ok: false, error: "name is required" } as const;
  }
  const project = await createProject({ name: parsed.data.name, ownerId: user.id });
  revalidatePath("/projects");
  redirect(`/projects/${project.id}`);
}
```

```tsx
// src/app/(dashboard)/projects/new/page.tsx
import { createProjectAction } from "./actions";

export default function NewProjectPage() {
  return (
    <form action={createProjectAction} className="space-y-4">
      <input name="name" placeholder="Project name" className="border p-2" />
      <button type="submit" className="bg-primary text-white px-4 py-2 rounded">Create</button>
    </form>
  );
}
```

The form submits, the server runs the action, the project is created, the page is revalidated, and the user is redirected. **No client-side fetch, no `useState`, no JSON parsing.**

For client-side validation + server action, use React Hook Form + Zod in Module 33.

---

## 6. Connecting to the Part A backend

### A typed `api` client

```ts
// src/lib/api.ts
import "server-only";

const BASE = process.env.API_URL ?? "http://localhost:4000";

export class ApiError extends Error {
  constructor(public status: number, public code: string, message: string) { super(message); }
}

export async function api<T>(path: string, init: RequestInit = {}): Promise<T> {
  const res = await fetch(`${BASE}${path}`, {
    ...init,
    headers: {
      "content-type": "application/json",
      cookie: init.headers ? String(init.headers["cookie"] ?? "") : "",
      ...(init.headers ?? {}),
    },
    cache: init.cache ?? "no-store",
    next: init.next,
  });

  if (!res.ok) {
    const body = await res.json().catch(() => ({}));
    throw new ApiError(res.status, body?.error?.code ?? "UNKNOWN", body?.error?.message ?? res.statusText);
  }
  return res.json() as Promise<T>;
}
```

### Forwarding auth cookies

```ts
// src/services/projects.ts
import "server-only";
import { cookies } from "next/headers";
import { api } from "@/lib/api";

export async function listProjects() {
  const cookieStore = await cookies();
  return api<{ data: Project[] }>("/api/v1/projects", {
    headers: { cookie: cookieStore.toString() },
  }).then((r) => r.data);
}
```

The Next.js server forwards the user's cookies to the Part A backend, which validates the JWT and returns the user's data. **No tokens in the browser.**

---

## 7. Where the data lives

```
Browser (no tokens)
   ↓ HTTPS + httpOnly cookies
Next.js server (route handlers, server actions, RSC fetch)
   ↓ HTTPS + cookies forwarded
Part A backend (Express + Drizzle + Postgres)
```

The browser **never** sees your API token, your database URL, or your LLM API key. They all live on the server.

---

## 8. Common mistakes

1. **Forgetting `"use client"` on a component that uses hooks.** Compile error, but easy to miss.
2. **Fetching on the client when server fetching would do.** Use TanStack Query only when you need *client-side* caching (offline, optimistic updates, polling).
3. **Mixing `cache: "force-cache"` with user-specific data.** Always use `cache: "no-store"` or tagged caching for authenticated endpoints.
4. **Not calling `revalidatePath` after a mutation.** The user sees stale data.
5. **Putting secrets in a client component.** Build will warn you; production will leak them.
6. **Forgetting to `await cookies()` / `await params` in Next 15.** Both are now async.

---

## Exercise

1. Build a `lib/api.ts` typed client that forwards cookies and parses `ApiError`.
2. Build a `services/projects.ts` that wraps `/api/v1/projects/*` and exposes typed functions.
3. Build a `/(dashboard)/projects/page.tsx` server component that lists projects.
4. Add a `/(dashboard)/projects/new/page.tsx` with a server action that creates a project and redirects.
5. Add `/(dashboard)/projects/[id]/page.tsx` that uses a Suspense boundary around a "recent activity" component, with a `<Skeleton/>` fallback.
6. Add `revalidatePath` after every mutation.
7. Verify the auth cookie is forwarded correctly (inspect the network tab in DevTools).

---

## AI Industry Note (2026)

- **Server Actions are the new "API for forms".** Most CRUD in a Next.js app in 2026 is a server action + a form. **Plain REST endpoints still exist for cross-app consumption** (mobile, partners), but for first-party forms, server actions are the senior default.
- **The "client fetch with TanStack Query" pattern is now the exception, not the rule.** Use TanStack Query when you need *client-side* features (optimistic updates, infinite scroll, polling). For everything else, server components are faster and simpler.
- **What to read next:** the [Next.js docs on Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations) and the [docs on caching](https://nextjs.org/docs/app/building-your-application/caching). The caching page is short, dense, and the difference between a fast app and a slow one.
