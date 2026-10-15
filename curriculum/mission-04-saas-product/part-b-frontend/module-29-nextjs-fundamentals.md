# Module 29 — Next.js Fundamentals & The App Router

> Why Next.js is the default React framework in 2026, and the conventions every senior frontend engineer uses.

---

## Why this module matters

You could build the SaaS frontend with plain React + Vite. You would re-invent routing, code splitting, server rendering, image optimization, and SEO meta tags badly. Next.js gives you all of those for free, with a single, well-documented mental model: the **App Router**.

By the end of this module you will be able to spin up a Next.js app, lay out routes with nested layouts, add loading and error states, and connect it to the Part A backend.

---

## Learning objectives

- Explain why Next.js exists and what the App Router gives you.
- Create a Next.js project with TypeScript, Tailwind, and ESLint.
- Use the file-system router: pages, layouts, nested layouts, route groups.
- Use dynamic routes, catch-all routes, loading, error, and not-found files.
- Add metadata for SEO.
- Use `Link`, `useRouter`, `usePathname`, and `useSearchParams`.
- Style with Tailwind and set up shadcn/ui.

---

## 1. Why Next.js

Plain React is a *library* for rendering UI. It does not give you:

- Routing (you reach for React Router).
- Server-side rendering (you set up Vite SSR or two apps).
- Data fetching conventions (every team invents their own).
- Code splitting (it happens, but not always where you want).
- SEO meta tags (you hand-roll them).
- Image optimization (you wire up `sharp`).

Next.js gives you all of these, opinionated and battle-tested. The App Router (introduced in 13.4, now the default) is built on **React Server Components** — your UI is server-rendered by default, with JavaScript only shipping for the parts that need it.

---

## 2. Project setup

```bash
pnpm create next-app@latest nl-saas-frontend --ts --tailwind --eslint --app --src-dir --import-alias "@/*"
cd nl-saas-frontend
pnpm dev
```

What you got:

- `app/` directory with the App Router.
- `src/` mirror so you can colocate non-routed code.
- `@/*` path alias.
- Tailwind CSS pre-wired.
- ESLint config.

For shadcn/ui:

```bash
pnpm dlx shadcn@latest init
pnpm dlx shadcn@latest add button card input form dialog dropdown-menu table tabs toast sheet avatar skeleton separator
```

This pulls in Radix UI primitives + Tailwind-styled components. You now have a production-ready design system.

---

## 3. The App Router file conventions

```
src/app/
├── layout.tsx           # root layout — wraps every route
├── page.tsx             # / — homepage
├── loading.tsx          # auto-shown while the route is loading
├── error.tsx            # auto-shown on unhandled errors
├── not-found.tsx        # 404 UI
├── globals.css
│
├── (marketing)/         # route group — no URL segment
│   ├── layout.tsx       # marketing-only layout
│   ├── page.tsx         # /
│   └── pricing/page.tsx # /pricing
│
├── (auth)/
│   ├── layout.tsx
│   ├── login/page.tsx
│   └── register/page.tsx
│
└── (dashboard)/
    ├── layout.tsx       # dashboard shell — sidebar + topbar
    ├── projects/
    │   ├── page.tsx     # /projects
    │   └── [id]/
    │       ├── page.tsx # /projects/:id
    │       └── settings/page.tsx
    └── settings/page.tsx
```

A **route group** `(name)/` lets you organize files without changing the URL. `(marketing)/pricing/page.tsx` → `/pricing`.

---

## 4. Layouts — composition

```tsx
// src/app/layout.tsx
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: { default: "NextLevel", template: "%s · NextLevel" },
  description: "Be the software developer AI can't easily replace",
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body className="min-h-screen bg-background text-foreground antialiased">
        {children}
      </body>
    </html>
  );
}
```

```tsx
// src/app/(dashboard)/layout.tsx
import { Sidebar } from "@/components/sidebar";
import { Topbar } from "@/components/topbar";
import { requireUser } from "@/lib/auth";

export default async function DashboardLayout({ children }: { children: React.ReactNode }) {
  const user = await requireUser();   // server-side auth check
  return (
    <div className="grid min-h-screen grid-cols-[260px_1fr]">
      <Sidebar user={user} />
      <div className="flex flex-col">
        <Topbar user={user} />
        <main className="flex-1 p-6">{children}</main>
      </div>
    </div>
  );
}
```

Layouts nest. The root layout wraps everything. The dashboard layout wraps the dashboard. Each route can override only what it needs.

---

## 5. Pages — server components by default

```tsx
// src/app/(dashboard)/projects/page.tsx
import { listProjects } from "@/services/projects";
import { ProjectList } from "@/components/projects/project-list";

export default async function ProjectsPage() {
  const projects = await listProjects();
  return (
    <div className="space-y-4">
      <h1 className="text-2xl font-semibold">Projects</h1>
      <ProjectList initial={projects} />
    </div>
  );
}
```

This is a **server component**. It runs on the server, calls `listProjects()` (which fetches the Part A backend), and ships the rendered HTML to the browser. Zero JS for this page by default.

---

## 6. Dynamic routes

```tsx
// src/app/(dashboard)/projects/[id]/page.tsx
import { notFound } from "next/navigation";
import { getProject } from "@/services/projects";

export default async function ProjectPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const project = await getProject(id);
  if (!project) notFound();
  return <ProjectDetail project={project} />;
}
```

### Catch-all

```tsx
// src/app/docs/[...slug]/page.tsx
export default async function DocsPage({ params }: { params: Promise<{ slug: string[] }> }) {
  const { slug } = await params;
  return <DocRenderer path={slug.join("/")} />;
}
```

### Optional catch-all

```tsx
// src/app/shop/[[...slug]]/page.tsx  → /shop, /shop/a, /shop/a/b
```

---

## 7. Loading, error, not-found

```tsx
// src/app/(dashboard)/projects/loading.tsx
import { Skeleton } from "@/components/ui/skeleton";

export default function Loading() {
  return (
    <div className="space-y-3">
      <Skeleton className="h-8 w-48" />
      <Skeleton className="h-32 w-full" />
    </div>
  );
}
```

```tsx
// src/app/(dashboard)/projects/error.tsx
"use client";
import { useEffect } from "react";
import { Button } from "@/components/ui/button";

export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  useEffect(() => { console.error(error); }, [error]);
  return (
    <div className="p-6">
      <h2 className="text-lg font-semibold">Something went wrong</h2>
      <Button onClick={reset} className="mt-3">Try again</Button>
    </div>
  );
}
```

```tsx
// src/app/(dashboard)/projects/[id]/not-found.tsx
import Link from "next/link";
export default function NotFound() {
  return <div>Project not found. <Link href="/projects">Back to projects</Link></div>;
}
```

---

## 8. Metadata & SEO

```tsx
// Static
export const metadata: Metadata = { title: "Pricing" };

// Dynamic
export async function generateMetadata({ params }: { params: { id: string } }): Promise<Metadata> {
  const project = await getProject(params.id);
  return { title: project.name, description: project.description };
}
```

```ts
// src/app/sitemap.ts
import type { MetadataRoute } from "next";
export default function sitemap(): MetadataRoute.Sitemap {
  return [{ url: "https://nextlevel.app", lastModified: new Date() }];
}

// src/app/robots.ts
export default function robots(): MetadataRoute.Robots {
  return { rules: { userAgent: "*", allow: "/" } };
}
```

---

## 9. Navigation

```tsx
import Link from "next/link";
<Link href="/projects/123">View project</Link>      // client-side navigation
<Link href="/projects/123" prefetch={false}>...</Link>
```

```tsx
"use client";
import { useRouter, usePathname, useSearchParams } from "next/navigation";

export function FilterBar() {
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();

  const setFilter = (key: string, value: string) => {
    const params = new URLSearchParams(searchParams.toString());
    params.set(key, value);
    router.push(`${pathname}?${params.toString()}`);
  };
  // ...
}
```

---

## 10. Server vs client components — the decision rule

| Need | Use |
|------|-----|
| Fetch data | Server |
| Access backend secrets / cookies | Server |
| Render large lists from DB | Server |
| Use `useState`, `useEffect`, browser APIs | Client |
| Handle click / form events | Client |
| Use shadcn/ui (Radix) | Client (they have interactivity) |
| Use third-party libraries that need browser | Client |

Add `"use client"` **at the top of the file** when needed. The convention:

- Keep pages and layouts as server components.
- Push client components to `components/features/*` and leaf nodes.

---

## 11. Tailwind + shadcn/ui

shadcn/ui is *not* a component library. It's a CLI that copies components into your `components/ui/` folder. You own the code, you can edit it.

```tsx
import { Button } from "@/components/ui/button";
import { Card, CardHeader, CardTitle, CardContent } from "@/components/ui/card";

export function Hero() {
  return (
    <Card>
      <CardHeader>
        <CardTitle>Welcome</CardTitle>
      </CardHeader>
      <CardContent>
        <Button>Get started</Button>
      </CardContent>
    </Card>
  );
}
```

The shadcn theme lives in `src/app/globals.css` and reads CSS variables for colors, so dark mode is a one-line config (Module 32).

---

## 12. The project structure we'll use in Missions 4 and 6

```
src/
├── app/                  # routes only
│   ├── (auth)/login/page.tsx
│   ├── (dashboard)/...
│   └── api/.../route.ts
├── components/
│   ├── ui/               # shadcn components
│   ├── features/         # feature-specific (project-list, task-form, ...)
│   └── layout/           # shell (sidebar, topbar)
├── lib/                  # utility code
│   ├── api.ts            # typed fetch wrapper
│   ├── auth.ts           # requireUser, getSession
│   └── utils.ts          # cn(), date helpers
├── services/             # server-only data accessors
│   ├── projects.ts
│   └── tasks.ts
├── hooks/                # client hooks
│   ├── use-debounce.ts
│   └── use-toast.ts
├── types/                # shared types
└── styles/globals.css
```

---

## Exercise

1. Create a Next.js app with `create-next-app` and TypeScript, Tailwind, ESLint, App Router.
2. Initialize shadcn/ui and add `button`, `card`, `input`, `form`, `dialog`, `dropdown-menu`.
3. Build a root layout with the global font and a `<Toaster/>` from shadcn.
4. Add a `(marketing)/` route group with `page.tsx` (a small landing page) and `pricing/page.tsx`.
5. Add a `(dashboard)/` route group with a layout containing a Sidebar + Topbar.
6. Add a `loading.tsx` and `error.tsx` to the dashboard group.
7. Add a dynamic route `dashboard/projects/[id]/page.tsx` that renders a fake project.
8. Add metadata for SEO at the root, the marketing group, and the dynamic route.

When done, `pnpm dev` and click around. You should feel the difference from a Vite + React Router setup: navigation is instant, no full page reloads, layouts persist.

---

## Common mistakes

1. **Adding `"use client"` to every file.** Most files should be server components.
2. **Forgetting to use `<Link>`.** Plain `<a href>` triggers a full page reload.
3. **Mutating `searchParams` directly.** Always create a new `URLSearchParams`.
4. **Huge root layout.** Keep the root layout minimal; nest heavier UI in route groups.
5. **Not handling `params` as a Promise in Next 15.** `params` is now async — `await` it.

---

## AI Industry Note (2026)

- **The App Router is the new default and the AI knows it.** When you ask Claude or GPT-4o to "build a Next.js app with auth," it will reach for App Router conventions: `app/(auth)/login/page.tsx`, server actions, route handlers. If you still write Pages Router code, the AI will produce outdated patterns. Migrate now.
- **Server Components are the most important React shift since hooks.** Most of your data fetching moves server-side. The client only handles interactivity. This is the biggest leverage point for performance in 2026: less JS, faster TTI, better SEO.
- **What to read next:** the [Next.js App Router docs](https://nextjs.org/docs/app) (in particular the "Data Fetching", "Caching", and "Server Actions" pages) and the [shadcn/ui docs](https://ui.shadcn.com/). Both are short and you can return to them as a reference for years.
