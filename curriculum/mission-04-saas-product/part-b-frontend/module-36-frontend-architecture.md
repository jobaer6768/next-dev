# Module 36: Scalable Frontend Architecture & Performance

> Mission 4 · Part B · Frontend — The architecture decisions you make in week 3 are the ones you'll either thank yourself for or curse at in month 14. Get the bones right; the rest is downhill.

## Why this matters

By Mission 4, you've built the product. Now it has to *stay* built. You'll add a feature, refactor a feature, onboard a teammate, debug a regression at 2am, and ship a hotfix without breaking the rest. The shape of your folder tree, the boundary between server and client state, your build pipeline, your error reporting — all of those decisions compound.

This module is the "settle into a real codebase" module. It's less about new features and more about the cross-cutting concerns that turn a working prototype into a product you can maintain. The frameworks, the patterns, the tooling, the deployment, the operations. The unglamorous 20% that determines whether your team ships 5 features a quarter or 50.

## Learning objectives

By the end of this module you will be able to:

1. Organize a **frontend project structure** that scales past 200 components.
2. Choose the right **state management** tool for the job (Context, Zustand, Redux Toolkit, TanStack Query).
3. Diagnose and fix the most common **performance** problems in a Next.js app.
4. Ship a **production build** to Vercel with environment, observability, and CI.
5. Add **error boundaries** and **Sentry** for production logging.
6. Set up **lint, format, type-check**, and **pre-commit hooks** that catch problems before they hit `main`.

## Conceptual model

### 1. Project structure at scale

A 200-file codebase needs a folder layout that tells you, at a glance, "where does this kind of thing live?". Here's the layout we use across our production Next.js 15 apps:

```
src/
├── app/                     # Next.js App Router
│   ├── (marketing)/         # route group — public
│   │   ├── page.tsx
│   │   └── pricing/page.tsx
│   ├── (app)/               # route group — auth required
│   │   ├── layout.tsx
│   │   ├── projects/
│   │   │   ├── page.tsx
│   │   │   ├── [id]/
│   │   │   │   ├── page.tsx
│   │   │   │   ├── settings/page.tsx
│   │   │   │   └── analytics/page.tsx
│   │   │   └── _components/     # route-private components
│   │   └── settings/page.tsx
│   ├── api/                 # route handlers
│   │   ├── projects/route.ts
│   │   └── webhooks/stripe/route.ts
│   └── layout.tsx
│
├── components/
│   ├── ui/                  # shadcn primitives — no business logic
│   │   ├── button.tsx
│   │   ├── data-table.tsx
│   │   └── file-dropzone.tsx
│   └── features/            # feature components — own business logic
│       ├── projects/
│       │   ├── project-form.tsx
│       │   ├── project-card.tsx
│       │   └── project-list.tsx
│       ├── analytics/
│       │   └── stat-card.tsx
│       └── billing/
│           └── plan-badge.tsx
│
├── lib/                     # framework-agnostic utilities
│   ├── db.ts                # Prisma client singleton
│   ├── auth.ts              # auth helpers
│   ├── utils.ts             # cn(), formatters
│   ├── fetcher.ts           # SWR-style fetcher
│   └── server/
│       ├── ratelimit.ts
│       └── email.ts
│
├── services/                # business data access (server-only)
│   ├── projects.ts
│   ├── analytics.ts
│   └── billing.ts
│
├── hooks/                   # client hooks
│   ├── use-debounced-value.ts
│   ├── use-media-query.ts
│   └── use-mounted.ts
│
├── stores/                  # zustand stores (client UI state)
│   ├── wizard-store.ts
│   └── ui-store.ts
│
├── schemas/                 # Zod schemas (shared client+server)
│   ├── project.ts
│   └── user.ts
│
├── types/                   # ambient TS types
│   ├── api.d.ts
│   └── env.d.ts
│
└── styles/
    └── globals.css
```

Three rules of thumb:

- **`components/ui`** is the design system. It has zero knowledge of your business domain. A `Button` doesn't know about `Project`.
- **`components/features`** is the product. It composes `ui` with business logic. `ProjectForm` knows about `Project`.
- **`services/`** is the only place that talks to the database. Components fetch from `services`, never from `db` directly.

Route groups `(marketing)` and `(app)` are how you say "these pages share a layout, these don't" without affecting the URL.

### 2. State management — the matrix

There are four jobs. Pick the right tool for each one.

| Job | Tool | Why |
|-----|------|-----|
| **Server state** (data from the API) | **TanStack Query** | Caching, revalidation, dedup, optimistic updates — for free |
| **Client UI state** (modal open, theme, wizard step) | **Zustand** | Tiny, no provider, no boilerplate, persists in 5 lines |
| **Form state** | **React Hook Form** | Already in module 33; non-negotiable |
| **Cross-cutting global state** (rare) | **Context** | For things that change once per session: auth, theme |

The most common mistake: **storing server data in Zustand**. Don't. TanStack Query is built for it. It handles cache invalidation, refetch-on-focus, and stale data correctly — all of which you will reimplement badly if you try to do it in Zustand.

**A typical query layer:**

```typescript
// services/projects.ts
export async function listProjects(params: ListParams) {
  // server-side call
}

// hooks/queries/use-projects.ts
"use client";
import { useQuery, keepPreviousData } from "@tanstack/react-query";
import { listProjects } from "@/services/projects";

export function useProjects(params: ListParams) {
  return useQuery({
    queryKey: ["projects", params],
    queryFn: () => listProjects(params),
    placeholderData: keepPreviousData, // smooth pagination
    staleTime: 30_000,
  });
}
```

**A typical UI store:**

```typescript
// stores/ui-store.ts
import { create } from "zustand";

type UIState = {
  commandPaletteOpen: boolean;
  sidebarCollapsed: boolean;
  setCommandPalette: (open: boolean) => void;
  toggleSidebar: () => void;
};

export const useUI = create<UIState>((set) => ({
  commandPaletteOpen: false,
  sidebarCollapsed: false,
  setCommandPalette: (commandPaletteOpen) => set({ commandPaletteOpen }),
  toggleSidebar: () => set((s) => ({ sidebarCollapsed: !s.sidebarCollapsed })),
}));
```

Use it:

```tsx
"use client";
const open = useUI((s) => s.commandPaletteOpen);
const setOpen = useUI((s) => s.setCommandPalette);
```

**When you might actually need Redux Toolkit:** when you have a *very* large, deeply nested client state with many actions, time-travel debugging, or strict immutability requirements. For 95% of SaaS, Zustand is enough.

### 3. Performance

**a) Images**

`<Image>` from `next/image` is non-negotiable. It does:

- Automatic format conversion (AVIF/WebP) based on `Accept`.
- Responsive `srcset` for the user's device.
- Lazy loading by default.
- Blur-up placeholders.

```tsx
import Image from "next/image";

<Image
  src="/hero.png"
  alt="Hero"
  width={1200}
  height={600}
  priority        // for above-the-fold
  placeholder="blur"
  blurDataURL={shimmer(1200, 600)}
/>
```

`shimmer` is a tiny base64 PNG generator that gives a smooth loading effect — search the web for "next image blur placeholder".

**b) Fonts**

`next/font` self-hosts your fonts and eliminates a render-blocking request.

```tsx
// app/layout.tsx
import { Inter } from "next/font/google";

const inter = Inter({ subsets: ["latin"], variable: "--font-inter" });

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={inter.variable}>
      <body className="font-sans">{children}</body>
    </html>
  );
}
```

Then in Tailwind: `font-family: var(--font-inter)`. Self-hosted. Zero CLS. Zero external requests.

**c) Bundle size**

`@next/bundle-analyzer` is your microscope.

```bash
pnpm add -D @next/bundle-analyzer
```

```js
// next.config.js
const withBundleAnalyzer = require("@next/bundle-analyzer")({ enabled: process.env.ANALYZE === "true" });
module.exports = withBundleAnalyzer({});
```

```bash
ANALYZE=true pnpm build
```

Three rules:

- **Don't import the world.** A single import of `lodash` adds 70KB. Use `lodash-es` or import individual functions.
- **Dynamic-import heavy widgets** that aren't on the first screen.
- **Don't put a charting library on the home page** if it isn't used there.

**d) Dynamic imports**

```tsx
"use client";

import dynamic from "next/dynamic";
import { Skeleton } from "@/components/ui/skeleton";

const AnalyticsCharts = dynamic(
  () => import("@/components/features/analytics/charts").then((m) => m.Charts),
  {
    loading: () => <Skeleton className="h-72 w-full" />,
    ssr: false,
  }
);
```

**e) Suspense for slow UI**

Module 35 covered streaming. The principle: any expensive client component goes inside a `<Suspense>` boundary with a meaningful skeleton. The user sees the page; the slow parts stream in.

### 4. Error boundaries and Sentry

**An error boundary catches render-phase errors in its subtree** and shows a fallback. Server errors are different — they produce a 500 unless you handle them. Both are common.

**`app/error.tsx`** (catches errors in the route segment):

```tsx
"use client";
import { useEffect } from "react";
import { Button } from "@/components/ui/button";

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    // forward to Sentry, LogRocket, Datadog, etc.
    console.error(error);
  }, [error]);

  return (
    <div className="p-6 text-center space-y-3">
      <h2 className="text-xl font-semibold">Something went wrong</h2>
      <p className="text-muted-foreground">
        {error.message || "An unexpected error occurred."}
      </p>
      <Button onClick={reset}>Try again</Button>
    </div>
  );
}
```

**Sentry setup** in 2026:

```bash
pnpm add @sentry/nextjs
npx @sentry/wizard@latest -i nextjs
```

The wizard wires up:

- Server-side error capture (in `sentry.server.config.ts`)
- Client-side error capture (in `sentry.client.config.ts`)
- Source-map upload on build
- A tunnel route to avoid ad-blockers

In your `instrumentation.ts`:

```typescript
export async function register() {
  if (process.env.NEXT_RUNTIME === "nodejs") {
    await import("./sentry.server.config");
  }
  if (process.env.NEXT_RUNTIME === "edge") {
    await import("./sentry.edge.config");
  }
}
```

Now every uncaught error — render, server action, route handler, edge function — lands in Sentry with a full stack trace, breadcrumbs, and the user's session replay.

### 5. Environment configuration

**`.env.local`** for development (git-ignored).

```
DATABASE_URL=postgres://...
NEXTAUTH_SECRET=...
STRIPE_SECRET_KEY=sk_test_...
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

Two classes:

- **`NEXT_PUBLIC_*`** — exposed to the browser. Anything in `process.env.NEXT_PUBLIC_*` is inlined at build time.
- **Everything else** — server only. Don't prefix with `NEXT_PUBLIC_` if it must stay server-side.

**Runtime vs build-time.** In Vercel:

- Build-time env vars are baked into the bundle. If you change one, you must redeploy.
- Runtime env vars are read on each request. Set them in the Vercel dashboard and you can change behavior without redeploying.

Use runtime for anything that differs per environment (per-tenant keys, etc). Use build-time for "this is the URL we use during SSG".

**`next.config.js`** for build-time configuration that can't be env vars:

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  experimental: { typedRoutes: true },
  images: {
    remotePatterns: [
      { protocol: "https", hostname: "cdn.acme.com" },
    ],
  },
  async headers() {
    return [
      {
        source: "/(.*)",
        headers: [
          { key: "X-Content-Type-Options", value: "nosniff" },
          { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
          { key: "X-Frame-Options", value: "DENY" },
        ],
      },
    ];
  },
};
module.exports = nextConfig;
```

### 6. Deploying to Vercel

Vercel is what Next.js was built for. The setup is:

1. Push to GitHub.
2. Import the repo at [vercel.com/new](https://vercel.com/new).
3. Set env vars.
4. Push. Vercel builds, deploys, and gives you a URL.

What Vercel does for free:

- **CDN**: every static asset is on the edge.
- **Image optimization**: `next/image` is served from Vercel's image CDN with on-the-fly resizing.
- **Edge cache**: server-rendered pages are cached at the edge with stale-while-revalidate.
- **Preview deploys**: every PR gets a unique URL — share with your PM.
- **Zero-config TLS, custom domains, redirects, analytics** — all built in.

The mental model: you ship a Next.js app and *forget about infrastructure*. That tradeoff is worth real money for a small team.

### 7. Production hygiene

The pre-commit gauntlet: **lint, format, type-check, test**. Hooks make it impossible to commit broken code.

**`package.json`:**

```json
{
  "scripts": {
    "lint": "next lint",
    "lint:fix": "next lint --fix",
    "format": "prettier --write .",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "build": "next build"
  }
}
```

**Prettier config** (`.prettierrc`):

```json
{
  "semi": true,
  "singleQuote": false,
  "trailingComma": "es5",
  "printWidth": 100,
  "plugins": ["prettier-plugin-tailwindcss"]
}
```

**ESLint** extends `next/core-web-vitals` and a couple of opinionated plugins:

```js
// .eslintrc.json
{
  "extends": ["next/core-web-vitals", "prettier"],
  "rules": {
    "no-console": ["warn", { "allow": ["warn", "error"] }],
    "@next/next/no-img-element": "error"
  }
}
```

**Pre-commit hook with `lefthook`** (`.lefthook.yml`):

```yaml
pre-commit:
  parallel: true
  commands:
    lint:
      glob: "*.{ts,tsx}"
      run: pnpm lint:fix {staged_files}
    typecheck:
      run: pnpm typecheck
    format:
      glob: "*.{ts,tsx,md,json}"
      run: pnpm prettier --write {staged_files}
```

**CI** (`.github/workflows/ci.yml`):

```yaml
name: CI
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck
      - run: pnpm lint
      - run: pnpm test
      - run: pnpm build
        env:
          DATABASE_URL: postgres://test
          NEXTAUTH_SECRET: ci-secret
```

A PR that fails any of these is not mergeable. That's the whole point of CI.

## Common mistakes

- **A `services/` folder that doesn't actually centralize data access.** If `db.project.findMany()` shows up in components, your abstraction has leaked.
- **Storing fetched data in Zustand.** Use TanStack Query. It's literally built for this.
- **Using `<img>` instead of `<Image>`.** Each one is a missed optimization opportunity.
- **Adding `useEffect` to "load data" inside a component that could be a server component.** Server components exist to remove this exact pattern.
- **No error boundary.** The default behavior is a white screen. Always have an `error.tsx` and a `not-found.tsx` per route segment.
- **Skipping the pre-commit hooks** because "we'll remember". You won't.
- **Treating Vercel as "just hosting".** It's your CDN, your image optimizer, your edge cache, your preview environments. Use it.

## Exercise

Refactor your mission-4 codebase with the following checklist:

1. **Folder structure** matches the layout in this module. Move components to `ui/` or `features/` based on whether they have business logic.
2. **No `db.*` calls outside `services/`**. Use grep to confirm: `grep -r "from \"@/lib/db\"" src/components` should return nothing.
3. **No server state in Zustand.** All API data is fetched via TanStack Query.
4. **All images** use `next/image`. No `<img>` tags.
5. **An `error.tsx` and `not-found.tsx` in every route segment** that has a meaningful risk of failure.
6. **Sentry** wired up with a `vercel.json` deploy hook. Trigger a test error in dev, confirm it shows up in the Sentry dashboard.
7. **`.env.example`** in the repo with all required vars (no values).
8. **CI workflow** that runs lint, typecheck, test, build on every PR. Block merge on failure.
9. **Pre-commit hooks** via `lefthook` or `husky` + `lint-staged`.
10. **Bundle analyzer** runs and you can identify your top-3 largest deps. Dynamic-import any that aren't needed on the home page.

Bonus: add a **`/healthz`** route that returns `{ ok: true, version: process.env.VERCEL_GIT_COMMIT_SHA }` — UptimeRobot or your load balancer can ping it.

## AI Industry Note (2026)

Frontend architecture in 2026 has one new axis: **AI-assisted code generation** is now a first-class teammate. Cursor, Claude Code, and Copilot Workspace are writing 30–60% of new code in many startups. The implication for your architecture: **boundaries matter more than ever**, because you want to be able to tell the AI "create a new analytics page that uses the same primitives" and have it succeed on the first try.

The folders and patterns in this module are exactly what those tools are trained on. `components/ui`, `components/features`, `services/`, `hooks/`, `schemas/`, `stores/` — these are the names an LLM will guess, and the right answer. If you have a custom folder layout with names like `widgets/`, `stuff/`, or `misc/`, you're making the AI's job harder and your own job harder.

There's also a new role emerging: **prompt-driven component generation**. Tools like **v0.dev**, **Anthropic Artifacts**, and **Replit Agent** can take a Figma file or a prompt and produce a working shadcn-style component. The bottleneck is no longer "can we make a Button?" — it's "is our design system expressive enough that the AI can compose it?". **Tokens, variants, slots** — these become the API your AI writes against. Build them well.
