# Module 31 — Authentication on the Frontend

> Login, sessions, protected routes, and the role-based UI that turns a static site into a product.

---

## Why this module matters

In Module 22 you built the auth API. In Module 23 you protected the routes on the server. Now you need the UI: login pages, registration forms, a session-aware header, and a way to redirect unauthorized users *before* they see protected content.

Done wrong, auth UI leaks tokens, has flash-of-unauthenticated-content, or trusts the client to enforce roles. Done right, the user barely notices it — they sign in once and the rest of the app Just Works.

---

## Learning objectives

- Build login and registration UIs with React Hook Form + Zod.
- Store and refresh tokens using httpOnly cookies (set by the backend).
- Use Next.js middleware for route protection.
- Implement protected layouts with redirect logic.
- Render UI conditionally based on user role.
- Build a workspace switcher.

---

## 1. Login & Registration UIs

The forms use React Hook Form + Zod (full details in Module 33; the snippet below is the minimum needed for auth).

```tsx
"use client";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import { useRouter, useSearchParams } from "next/navigation";
import { useState } from "react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from "@/components/ui/form";

const Schema = z.object({
  email: z.string().email(),
  password: z.string().min(8).max(200),
});
type Values = z.infer<typeof Schema>;

export function LoginForm() {
  const router = useRouter();
  const search = useSearchParams();
  const [serverError, setServerError] = useState<string | null>(null);

  const form = useForm<Values>({ resolver: zodResolver(Schema), defaultValues: { email: "", password: "" } });

  async function onSubmit(values: Values) {
    setServerError(null);
    const res = await fetch("/api/auth/login", {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify(values),
    });
    if (!res.ok) {
      setServerError("Invalid email or password");
      return;
    }
    router.push(search.get("next") ?? "/dashboard");
    router.refresh();
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        <FormField control={form.control} name="email" render={({ field }) => (
          <FormItem>
            <FormLabel>Email</FormLabel>
            <FormControl><Input type="email" {...field} /></FormControl>
            <FormMessage />
          </FormItem>
        )} />
        <FormField control={form.control} name="password" render={({ field }) => (
          <FormItem>
            <FormLabel>Password</FormLabel>
            <FormControl><Input type="password" {...field} /></FormControl>
            <FormMessage />
          </FormItem>
        )} />
        {serverError && <p className="text-sm text-destructive">{serverError}</p>}
        <Button type="submit" className="w-full">Sign in</Button>
      </form>
    </Form>
  );
}
```

The form posts to a Next.js route handler that proxies to the Part A backend. The backend sets the httpOnly cookie and returns 204; the frontend navigates.

### Why a route handler instead of calling the backend directly?

The backend is on a different origin (or it will be, in production). The browser can't set httpOnly cookies cross-origin. So:

```
Browser → POST /api/auth/login (Next.js route handler)
        → forwards to Part A backend
        ← Set-Cookie: accessToken=...; HttpOnly; Secure; SameSite=Lax
        ← 204
        → 204 to browser
        → browser stores the cookie on the *Next.js* origin
        → subsequent requests to Next.js include the cookie
        → Next.js server forwards the cookie to the backend
```

Both apps effectively share the cookie via the proxy. **The browser never touches the token.**

---

## 2. The auth route handlers (proxy layer)

```ts
// src/app/api/auth/login/route.ts
import { NextResponse } from "next/server";

const API = process.env.API_URL!;

export async function POST(req: Request) {
  const body = await req.json();
  const res = await fetch(`${API}/api/v1/auth/login`, {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify(body),
  });
  if (!res.ok) {
    const data = await res.json().catch(() => ({}));
    return NextResponse.json(data, { status: res.status });
  }
  // Forward Set-Cookie headers from the backend
  const out = new NextResponse(null, { status: 204 });
  const setCookie = res.headers.get("set-cookie");
  if (setCookie) out.headers.append("set-cookie", setCookie);
  return out;
}
```

Similarly for `/api/auth/logout`, `/api/auth/register`, `/api/auth/refresh`.

---

## 3. Session on the server

```ts
// src/lib/auth.ts
import "server-only";
import { cookies } from "next/headers";
import { redirect } from "next/navigation";
import { cache } from "react";
import { api, ApiError } from "./api";

export type Session = { user: User; memberships: Membership[] };

export const getSession = cache(async (): Promise<Session | null> => {
  const cookieStore = await cookies();
  if (!cookieStore.get("accessToken")) return null;
  try {
    return await api<Session>("/api/v1/auth/me", {
      headers: { cookie: cookieStore.toString() },
      cache: "no-store",
    });
  } catch (e) {
    if (e instanceof ApiError && e.status === 401) return null;
    throw e;
  }
});

export async function requireUser() {
  const session = await getSession();
  if (!session) redirect("/login");
  return session.user;
}
```

`cache()` dedupes `getSession()` per request. Many components can call it; the backend is hit once.

---

## 4. Next.js middleware for route protection

```ts
// src/middleware.ts
import { NextResponse, type NextRequest } from "next/server";

const PROTECTED = ["/dashboard", "/projects", "/settings", "/billing"];
const AUTH_ONLY = ["/login", "/register"];

export function middleware(req: NextRequest) {
  const { pathname } = req.nextUrl;
  const hasAccessToken = req.cookies.has("accessToken");

  if (PROTECTED.some((p) => pathname.startsWith(p)) && !hasAccessToken) {
    const url = req.nextUrl.clone();
    url.pathname = "/login";
    url.searchParams.set("next", pathname);
    return NextResponse.redirect(url);
  }
  if (AUTH_ONLY.includes(pathname) && hasAccessToken) {
    const url = req.nextUrl.clone();
    url.pathname = "/dashboard";
    return NextResponse.redirect(url);
  }
  return NextResponse.next();
}

export const config = {
  matcher: ["/((?!_next|api|.*\\..*).*)"],
};
```

The middleware is a **first line of defense** — it short-circuits unauthenticated requests before they hit the page. The page-level `requireUser()` is the **second line** (in case the cookie was tampered or expired between middleware and render).

---

## 5. Protected layouts

```tsx
// src/app/(dashboard)/layout.tsx
import { requireUser } from "@/lib/auth";
import { Sidebar } from "@/components/layout/sidebar";
import { Topbar } from "@/components/layout/topbar";

export default async function DashboardLayout({ children }: { children: React.ReactNode }) {
  const user = await requireUser();   // redirects to /login if no session
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

Any route under `(dashboard)/` is automatically protected.

---

## 6. Role-based UI

```tsx
// src/components/layout/sidebar.tsx
import Link from "next/link";
import type { User } from "@/lib/auth";

export function Sidebar({ user }: { user: User }) {
  const isAdmin = user.role === "admin";   // or check via session memberships
  return (
    <nav className="border-r p-4 space-y-2">
      <Link href="/projects">Projects</Link>
      {isAdmin && <Link href="/admin">Admin</Link>}
      <Link href="/settings">Settings</Link>
    </nav>
  );
}
```

**Server-side rendering means the user *never* sees admin UI they shouldn't.** No flash of forbidden content, no client-side hacks.

For more granular per-route checks inside a server component:

```tsx
import { requireWorkspaceRole } from "@/lib/auth";
export default async function AdminPage({ params }: { params: { id: string } }) {
  await requireWorkspaceRole(params.id, "admin");
  // ...
}
```

---

## 7. Workspace switching

Workspaces are the tenant boundary. A user can be in many workspaces. The UI must let them switch.

### The pattern: store the active workspace in a cookie

```tsx
// Server action
"use server";
import { cookies } from "next/headers";
import { revalidatePath } from "next/cache";

export async function switchWorkspace(workspaceId: string) {
  cookies().set("activeWorkspace", workspaceId, { httpOnly: false, sameSite: "lax", path: "/" });
  revalidatePath("/", "layout");
}
```

```tsx
// Client component
"use client";
import { switchWorkspace } from "./actions";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";

export function WorkspaceSwitcher({ workspaces, active }: { workspaces: Workspace[]; active: string }) {
  return (
    <Select
      defaultValue={active}
      onValueChange={async (v) => {
        await switchWorkspace(v);
        window.location.reload();   // or router.refresh()
      }}
    >
      <SelectTrigger><SelectValue /></SelectTrigger>
      <SelectContent>
        {workspaces.map((w) => <SelectItem key={w.id} value={w.id}>{w.name}</SelectItem>)}
      </SelectContent>
    </Select>
  );
}
```

### Why a cookie and not localStorage?

- The cookie is sent automatically with every request, so the server can read the active workspace for every API call.
- The cookie can be marked `httpOnly: true` (then it's only readable server-side) and `sameSite: "lax"` (CSRF protection).
- The server is the source of truth for what the user is currently looking at.

---

## 8. Logout

```tsx
// Server action
"use server";
import { cookies } from "next/headers";
import { redirect } from "next/navigation";

export async function logout() {
  await fetch(`${process.env.API_URL}/api/v1/auth/logout`, {
    method: "POST",
    headers: { cookie: (await cookies()).toString() },
  });
  (await cookies()).delete("accessToken");
  (await cookies()).delete("refreshToken");
  redirect("/login");
}
```

The backend revokes the refresh token (Module 23); the frontend clears its cookies; the user is redirected.

---

## 9. Common mistakes

1. **Storing tokens in `localStorage`.** XSS-vulnerable. Use httpOnly cookies.
2. **Checking auth only in the client.** Use middleware + server components. The client can lie.
3. **Hardcoding the redirect URL after login.** Read `?next=` from the URL (and validate it).
4. **Forgetting to `router.refresh()` after login.** The server-rendered page may still show the old "logged out" state.
5. **Setting the cookie on the wrong domain.** When the backend is on a different origin, the cookie must be set by the *Next.js* origin (via the proxy route handler) or by the backend with `domain=.yourapp.com`.
6. **Rendering admin UI for non-admins on the client.** Always gate on the server.

---

## Exercise

1. Build `LoginForm` and `RegisterForm` with React Hook Form + Zod.
2. Build the proxy route handlers: `/api/auth/login`, `/api/auth/logout`, `/api/auth/register`, `/api/auth/refresh`.
3. Build `getSession` and `requireUser` in `lib/auth.ts` using `cache()`.
4. Build `middleware.ts` with the protected-path matcher.
5. Build the dashboard layout that calls `requireUser()`.
6. Build a `<WorkspaceSwitcher/>` that uses a server action to update a cookie.
7. Test: sign in, navigate to a protected page, try to navigate to `/login` while signed in (should redirect), sign out, see the protected page redirect to `/login`.

---

## AI Industry Note (2026)

- **The "tokens in localStorage" anti-pattern is the single most common security mistake in AI-generated React code.** When you ask an AI to "build login for Next.js," it often reaches for `localStorage.setItem("token", ...)` because that pattern is older and more prevalent in its training data. **Always specify: "Use httpOnly cookies set by the server. Never put the JWT in localStorage or sessionStorage."** That one sentence in your prompt prevents a security incident.
- **The Next.js middleware + server component pattern is the new "auth guard"** in 2026. Older code used `useEffect` + `useRouter().replace("/login")` on the client. That flash of unauthenticated content is gone in App Router. The AI needs to be told this — otherwise it'll produce the old pattern.
- **What to read next:** the [Next.js docs on middleware](https://nextjs.org/docs/app/building-your-application/routing/middleware) (short) and the OWASP [Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) (essential, regardless of framework).
