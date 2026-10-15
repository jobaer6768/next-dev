# Module 32 — Building the Product Dashboard: Layout & UI Architecture

> The shell that holds the product together.

---

## Why this module matters

Most of the SaaS product is *inside* the dashboard. The dashboard is the sidebar you navigate by, the topbar that shows the user, the empty states that teach the user what to do, the toasts that confirm their actions, the modals that ask for confirmation. Get the shell right and every feature you build later is faster. Get it wrong and every feature fights the UI.

This module gives you the patterns and the components you'll reuse across the entire frontend.

---

## Learning objectives

- Architect a dashboard with sidebar, topbar, and nested layouts.
- Make the UI responsive (mobile nav with a sheet).
- Compose server and client components.
- Build loading skeletons and empty states.
- Wire up toasts, modals, and confirmation dialogs.
- Implement dark mode.

---

## 1. The dashboard architecture

```
┌────────────────────────────────────────────────────────────┐
│ Topbar (search, workspace switcher, user menu, theme)      │
├──────────┬─────────────────────────────────────────────────┤
│ Sidebar  │ Main content                                     │
│ ──────── │   <Suspense> page content </Suspense>            │
│ Projects │                                                 │
│ Tasks    │                                                 │
│ Members  │                                                 │
│ Settings │                                                 │
└──────────┴─────────────────────────────────────────────────┘
```

The dashboard is a *nested* layout in the App Router. It's a single layout component that wraps every `(dashboard)/*` route. The page itself is a server component that fetches its data and renders inside the layout.

---

## 2. The Sidebar (client component, collapsible)

```tsx
// src/components/layout/sidebar.tsx
"use client";
import Link from "next/link";
import { usePathname } from "next/navigation";
import { cn } from "@/lib/utils";
import { Home, FolderKanban, Users, Settings, ChevronsLeft } from "lucide-react";

const items = [
  { href: "/dashboard",     label: "Overview",  icon: Home },
  { href: "/projects",      label: "Projects",  icon: FolderKanban },
  { href: "/members",       label: "Members",   icon: Users },
  { href: "/settings",      label: "Settings",  icon: Settings },
];

export function Sidebar({ collapsed, onToggle }: { collapsed: boolean; onToggle: () => void }) {
  const pathname = usePathname();
  return (
    <aside className={cn("border-r bg-card transition-all", collapsed ? "w-16" : "w-60")}>
      <div className="flex h-14 items-center justify-between px-4">
        {!collapsed && <span className="font-semibold">NextLevel</span>}
        <button onClick={onToggle} aria-label="Toggle sidebar">
          <ChevronsLeft className={cn("h-4 w-4 transition-transform", collapsed && "rotate-180")} />
        </button>
      </div>
      <nav className="space-y-1 p-2">
        {items.map((item) => {
          const Icon = item.icon;
          const active = pathname === item.href || pathname.startsWith(item.href + "/");
          return (
            <Link key={item.href} href={item.href}
              className={cn("flex items-center gap-2 rounded-md px-3 py-2 text-sm hover:bg-muted",
                active && "bg-muted text-foreground font-medium")}>
              <Icon className="h-4 w-4" />
              {!collapsed && <span>{item.label}</span>}
            </Link>
          );
        })}
      </nav>
    </aside>
  );
}
```

`usePathname` makes the active link work without manual state. `cn()` is a Tailwind class-name combiner (shadcn convention).

---

## 3. Mobile nav with a Sheet

```tsx
// src/components/layout/mobile-nav.tsx
"use client";
import { useState } from "react";
import { Menu } from "lucide-react";
import { Sheet, SheetContent, SheetTrigger } from "@/components/ui/sheet";
import { Sidebar } from "./sidebar";

export function MobileNav() {
  const [open, setOpen] = useState(false);
  return (
    <Sheet open={open} onOpenChange={setOpen}>
      <SheetTrigger asChild>
        <button className="md:hidden p-2" aria-label="Open menu"><Menu className="h-5 w-5" /></button>
      </SheetTrigger>
      <SheetContent side="left" className="p-0 w-60">
        <Sidebar collapsed={false} onToggle={() => setOpen(false)} />
      </SheetContent>
    </Sheet>
  );
}
```

The same `Sidebar` component renders in both the desktop column and the mobile sheet. **One component, two layouts.**

---

## 4. The Topbar

```tsx
// src/components/layout/topbar.tsx
import { UserMenu } from "./user-menu";
import { WorkspaceSwitcher } from "./workspace-switcher";
import { GlobalSearch } from "./global-search";
import { ThemeToggle } from "./theme-toggle";
import { MobileNav } from "./mobile-nav";
import { getSession } from "@/lib/auth";

export async function Topbar() {
  const session = await getSession();
  return (
    <header className="flex h-14 items-center gap-4 border-b bg-card px-4">
      <MobileNav />
      <WorkspaceSwitcher
        workspaces={session!.memberships.map((m) => m.workspace)}
        active={session!.memberships.find((m) => m.isActive)?.workspace.id ?? ""}
      />
      <div className="flex-1" />
      <GlobalSearch />
      <ThemeToggle />
      <UserMenu user={session!.user} />
    </header>
  );
}
```

`Topbar` is a server component (it calls `getSession()`). It composes client components for the interactive parts.

---

## 5. Skeletons and empty states

shadcn ships a `<Skeleton/>` (just an animated gray rectangle). Wrap your loading UI in it.

```tsx
// src/app/(dashboard)/projects/loading.tsx
import { Skeleton } from "@/components/ui/skeleton";

export default function Loading() {
  return (
    <div className="space-y-3">
      <Skeleton className="h-8 w-48" />
      {Array.from({ length: 6 }).map((_, i) => (
        <Skeleton key={i} className="h-14 w-full" />
      ))}
    </div>
  );
}
```

For empty states, build a small `<EmptyState/>` component:

```tsx
// src/components/empty-state.tsx
import { Inbox } from "lucide-react";
import Link from "next/link";
import { Button } from "@/components/ui/button";

export function EmptyState({
  icon: Icon = Inbox,
  title,
  description,
  action,
}: {
  icon?: React.ComponentType<{ className?: string }>;
  title: string;
  description?: string;
  action?: { label: string; href: string };
}) {
  return (
    <div className="flex flex-col items-center justify-center text-center py-16">
      <Icon className="h-10 w-10 text-muted-foreground" />
      <h3 className="mt-3 text-lg font-semibold">{title}</h3>
      {description && <p className="mt-1 text-sm text-muted-foreground max-w-md">{description}</p>}
      {action && (
        <Button asChild className="mt-4"><Link href={action.href}>{action.label}</Link></Button>
      )}
    </div>
  );
}
```

Use it in every list:

```tsx
if (projects.length === 0) {
  return <EmptyState title="No projects yet" description="Create your first project to get started." action={{ label: "New project", href: "/projects/new" }} />;
}
```

---

## 6. Toasts (sonner)

```tsx
// src/app/layout.tsx
import { Toaster } from "@/components/ui/sonner";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {children}
        <Toaster richColors closeButton />
      </body>
    </html>
  );
}
```

```tsx
// anywhere in a client component
import { toast } from "sonner";
toast.success("Project created");
toast.error("Failed to delete project");
```

For server actions, use `sonner` from a client component that wraps the form.

---

## 7. Modals and confirmation dialogs

```tsx
// src/components/projects/delete-project-dialog.tsx
"use client";
import { useState, useTransition } from "react";
import { Button } from "@/components/ui/button";
import { Dialog, DialogContent, DialogDescription, DialogFooter, DialogHeader, DialogTitle, DialogTrigger } from "@/components/ui/dialog";
import { deleteProjectAction } from "./actions";

export function DeleteProjectDialog({ id, name }: { id: string; name: string }) {
  const [open, setOpen] = useState(false);
  const [pending, startTransition] = useTransition();

  return (
    <Dialog open={open} onOpenChange={setOpen}>
      <DialogTrigger asChild>
        <Button variant="destructive">Delete</Button>
      </DialogTrigger>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>Delete {name}?</DialogTitle>
          <DialogDescription>This permanently deletes the project and all its tasks. This action cannot be undone.</DialogDescription>
        </DialogHeader>
        <DialogFooter>
          <Button variant="ghost" onClick={() => setOpen(false)}>Cancel</Button>
          <Button
            variant="destructive"
            disabled={pending}
            onClick={() => startTransition(async () => {
              await deleteProjectAction(id);
              setOpen(false);
            })}
          >
            {pending ? "Deleting..." : "Delete"}
          </Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
}
```

`useTransition` gives you the `pending` flag for the button without manual state.

---

## 8. Dark mode

```bash
pnpm add next-themes
```

```tsx
// src/components/theme-provider.tsx
"use client";
import { ThemeProvider as NextThemesProvider } from "next-themes";
export function ThemeProvider({ children, ...props }: React.ComponentProps<typeof NextThemesProvider>) {
  return <NextThemesProvider {...props}>{children}</NextThemesProvider>;
}
```

```tsx
// src/app/layout.tsx
<ThemeProvider attribute="class" defaultTheme="system" enableSystem>
  {children}
</ThemeProvider>
```

```tsx
// src/components/theme-toggle.tsx
"use client";
import { useTheme } from "next-themes";
import { Moon, Sun } from "lucide-react";
import { Button } from "@/components/ui/button";

export function ThemeToggle() {
  const { theme, setTheme } = useTheme();
  return (
    <Button variant="ghost" size="icon" onClick={() => setTheme(theme === "dark" ? "light" : "dark")}>
      <Sun className="h-4 w-4 dark:hidden" />
      <Moon className="h-4 w-4 hidden dark:block" />
    </Button>
  );
}
```

shadcn's CSS variables already define a dark palette in `globals.css`, so dark mode Just Works.

---

## 9. The complete dashboard layout

```tsx
// src/app/(dashboard)/layout.tsx
import { requireUser } from "@/lib/auth";
import { Sidebar } from "@/components/layout/sidebar";
import { Topbar } from "@/components/layout/topbar";
import { ShellState } from "@/components/layout/shell-state";

export default async function DashboardLayout({ children }: { children: React.ReactNode }) {
  const user = await requireUser();
  return (
    <ShellState>
      <div className="grid min-h-screen md:grid-cols-[auto_1fr]">
        <Sidebar />
        <div className="flex flex-col">
          <Topbar />
          <main className="flex-1 p-6">{children}</main>
        </div>
      </div>
    </ShellState>
  );
}
```

`ShellState` is a client component that owns the sidebar-collapsed state and exposes it via React context to both the desktop sidebar and the mobile sheet.

---

## 10. Component composition pattern

A page typically combines:

- **Server component** — fetches data, decides what to render.
- **Client wrapper** — owns local UI state (filters, modals, sort).
- **shadcn primitives** — buttons, inputs, dialogs.

```tsx
// page.tsx (server)
export default async function ProjectsPage() {
  const projects = await listProjects();
  return <ProjectsClient initial={projects} />;
}
```

```tsx
// projects-client.tsx ("use client")
export function ProjectsClient({ initial }: { initial: Project[] }) {
  const [filter, setFilter] = useState("");
  const filtered = initial.filter((p) => p.name.includes(filter));
  return (
    <>
      <Input value={filter} onChange={(e) => setFilter(e.target.value)} placeholder="Filter projects" />
      <ProjectList projects={filtered} />
    </>
  );
}
```

The data lives on the server; the interactivity lives on the client. You get fast first paints and snappy interactions.

---

## 11. Accessibility — the free wins

shadcn/ui is built on Radix, which handles:

- **Keyboard navigation** — Tab, Arrow keys, Enter, Esc.
- **ARIA attributes** — `aria-expanded`, `aria-controls`, `aria-selected`, etc.
- **Focus management** — focus is trapped in modals, restored on close.

You get most of this for free. **Do not override Radix's behavior with custom CSS unless you know what you're doing.**

The places to pay attention:

- All `<img>` need an `alt`.
- All interactive elements need a label (use `aria-label` for icon-only buttons).
- Color contrast — verify with the shadcn theme variables.
- Form errors — announce them with `aria-live="polite"`.

---

## 12. Common mistakes

1. **Building the shell as a single client component.** Keep the shell as server components, push client logic to small leaves.
2. **No empty states.** Every list needs an empty state.
3. **No loading states.** Every async UI needs a skeleton.
4. **Toasts for errors that need acknowledgement.** Errors belong in the UI; toasts are for confirmations.
5. **Modal-on-modal.** Don't stack dialogs.
6. **Sidebar active-link logic that ignores nested routes.** Use `pathname.startsWith(item.href + "/")`.

---

## Exercise

1. Set up shadcn/ui: install `button`, `card`, `input`, `dialog`, `dropdown-menu`, `sheet`, `skeleton`, `sonner`, `tooltip`, `avatar`, `tabs`.
2. Build the `ShellState` context, the desktop `Sidebar`, and the mobile `Sheet` nav.
3. Build the `Topbar` with workspace switcher, search input (placeholder), theme toggle, and user menu.
4. Add `EmptyState` and use it in `/projects` when there are no projects.
5. Add `loading.tsx` for `/projects` and `/members` with skeletons.
6. Wire up dark mode with `next-themes` and a `ThemeToggle`.
7. Build a `DeleteProjectDialog` using `Dialog` + `useTransition`.

---

## AI Industry Note (2026)

- **shadcn/ui + Tailwind + Radix is the most productive UI stack in 2026.** The components are copy-paste, accessible by default, and theme-able. AI assistants are excellent at generating variants because the patterns are so consistent. "Build a settings page with a form, a danger zone, and a save button" produces a coherent result on the first try.
- **The server-component + client-leaf pattern is the modern equivalent of "MVC for React."** AI assistants will often default to client components because the older training data favors them. The senior move is to always ask the AI to *default to server components* and to *only* mark `"use client"` at the leaves. The result is faster, smaller, and easier to maintain.
- **What to read next:** the [shadcn/ui docs](https://ui.shadcn.com/) (every component is a code snippet you can copy) and the [Radix UI primitives docs](https://www.radix-ui.com/primitives) (the underlying accessible primitives that power shadcn).
