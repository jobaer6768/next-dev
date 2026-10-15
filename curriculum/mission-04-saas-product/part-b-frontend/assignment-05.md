# Assignment 5 — Build a SaaS Frontend

> **Mission 4 · Part B · Capstone Frontend**
> Ship the production UI for the multi-tenant SaaS you built the backend for in Assignment 4.

---

## 1. Project Brief

You are the **frontend lead** for the same SaaS product from Assignment 4. The backend is already shipped (or at least, your teammate's is — or your own, from a few weeks ago). Your job is to build the **production frontend** that real customers will use.

In Modules 29–36 you learned the Next.js 15 App Router, React Server Components, frontend auth, dashboard layout, forms and file upload, tables and pagination, charts and analytics, and frontend architecture. **This is the assignment where you assemble that into a real, deployed, polished product.**

You will ship:

- A **Next.js 15 App Router** app with React 19 and TypeScript strict.
- **Auth** (login, register, refresh, logout) with **protected routes** via middleware.
- **Role-based UI** that hides or disables actions based on the user's role in the current workspace.
- **Team & workspace screens** (list, create, switch, invite, members).
- **File management UI** with drag-and-drop upload, previews, and signed-URL access.
- **Data tables** powered by **TanStack Table** with **search, filter, and pagination** mirrored to **URL state** (so a filtered view is shareable).
- **Analytics dashboard** with **Recharts** (line, bar, area), date-range filtering, and stat cards.
- A **responsive, production-ready UI** built on **Tailwind + shadcn/ui**, with **dark mode** via `next-themes`.
- A **deployed live link** on Vercel and a public **GitHub repo**.

The product domain is whatever you (or your backend teammate) chose in Assignment 4: project management, helpdesk, CRM, or ticketing. **Match the API; do not invent new endpoints.** If a route is missing, the right answer is to add it to the backend, not to mock it on the frontend.

> **Senior-engineer note:** A real frontend is judged on what it feels like, not on what it does. Loading states, error states, empty states, focus rings, keyboard nav, color contrast, and the difference between a button that says "Save" and a button that says "Save changes" — these are the things that separate a portfolio project from a product. The grading rubric reflects that. The 20% of marks for "polish" is not a gift; it is a recognition that you will spend 30% of your time on it.

---

## 2. Deliverables

A complete submission includes **all** of the following:

### 2.1 Code & repo

| Deliverable | Where it lives | Notes |
|---|---|---|
| App source | `/app` and `/src` | App Router, route groups for `(auth)` and `(dashboard)` |
| Components | `/components` | `ui/` (shadcn primitives), `forms/`, `tables/`, `charts/`, `workspace/` |
| API client | `/src/lib/api` | Typed fetch wrapper, Zod-validated responses, error normalization |
| Server actions / route handlers | `/app/api` (only if needed) | Prefer calling the backend directly; no business logic in the frontend |
| Auth | `/src/lib/auth` | Server-side session check (cookies or bearer from server components) |
| State | `/src/lib/queries` | TanStack Query hooks, one file per resource |
| Tables | `/components/tables` | TanStack Table + URL state via `nuqs` or `useSearchParams` |
| Charts | `/components/charts` | Recharts wrappers; data shape matches backend `timeseries` |
| Tests | `/tests` | Vitest + Testing Library for components; Playwright for one E2E |
| Env example | `/.env.example` | `NEXT_PUBLIC_API_URL`, `SENTRY_DSN`, etc. |
| README | `/README.md` | Setup, env, deploy, screenshots, demo |

### 2.2 Live deployment

**Required:** A Vercel deployment of the frontend, with `NEXT_PUBLIC_API_URL` pointed at the Assignment 4 backend (yours or a teammate's). A live URL is required for full marks; "works locally" earns at most 70% on the deployment category.

### 2.3 Submission artifact

- **GitHub repo URL** (public).
- **Vercel live URL** (a non-`*.vercel.app` custom domain is a bonus, not required).
- **10–15 screenshots** of every major screen, light + dark mode.
- A **2-minute demo video** walking through: login → create workspace → invite → switch workspace → upload a file → filter a table → open the analytics dashboard.

---

## 3. Conceptual Sessions (3 × ~90 min)

These three sessions build **one** medium frontend project — the UI for the Part A backend. They are not three separate projects; the second session continues the first, and the third ships it.

### Session 1 — "App shell: App Router layouts, auth pages, protected routes"

**Module references:** 29 (Next.js fundamentals), 30 (server components and data), 31 (frontend auth), 32 (dashboard layout).

**You will:**

1. Bootstrap: `pnpm create next-app@latest saas-frontend --typescript --tailwind --app --src-dir --import-alias '@/*'`. Add `pnpm dlx shadcn@latest init`. Add `next-themes`.
2. Define route groups:
   - `app/(auth)/login`, `app/(auth)/register`, `app/(auth)/invitations/[token]` — public, minimal chrome, centered card.
   - `app/(dashboard)/[workspaceSlug]/...` — protected, with the sidebar + topbar.
3. Build the auth pages with React Hook Form + Zod. Match the backend's `/auth/*` contract. On success, store the access token in an `httpOnly` cookie via a server action (do not put it in `localStorage`).
4. Implement `middleware.ts` that protects `app/(dashboard)/**`. Unauthenticated → redirect to `/login?next=…`. Authenticated but no `workspaceSlug` → redirect to `/workspaces` (the picker).
5. Build the dashboard shell: `Sidebar`, `Topbar` (with workspace switcher dropdown, user menu, theme toggle), `<CommandPalette/>` (bonus).
6. Render the user's workspaces in the switcher. Switching updates the URL and re-fetches.

**Exit criteria:** A logged-in user can switch workspaces, and refreshing the page keeps them logged in. A logged-out user hitting `/acme/projects` is redirected to `/login?next=/acme/projects` and, after login, lands back at the original URL.

**Outcomes:** Modules 29–32 in full. The auth contract you wrote here must match the Assignment 4 backend exactly — **use the same Zod schemas.**

### Session 2 — "Data screens: tables with search/filter/pagination, forms with validation"

**Module references:** 33 (forms & file upload), 34 (tables, search, pagination), 36 (frontend architecture).

**You will:**

1. Build the **workspace management screens**:
   - `WorkspacesListPage` — list of user's workspaces, "Create workspace" button, delete with confirmation.
   - `WorkspaceSettingsPage` — name, slug, members list, invite form, role badges.
   - `MembersTable` — TanStack Table, columns: name, email, role, joined, actions (change role, remove).
2. Build the **invitation flow**:
   - Admin types an email + role → POST to backend → success toast with a "copy invite link" action.
   - The link points to `/invitations/[token]`. Public page shows workspace name, "Accept" button (requires login; if not, redirects to `/login?next=/invitations/[token]`).
3. Build the **core domain screens** — projects and tasks (or whatever your domain uses):
   - List view: TanStack Table with column sorting, **search** (`q` param), **filters** (status, assignee, tag — each in URL state via `nuqs`), and **pagination** (page or cursor in URL).
   - Detail view: side panel with comments and attachments.
   - Create / edit forms: React Hook Form + Zod (shared schemas with the backend), server errors mapped to field errors, optimistic updates for "mark as done" via TanStack Query's `onMutate`.
4. Build the **file management UI**:
   - `FileDropzone` component: drag-and-drop, paste-to-upload, click-to-browse.
   - Shows progress, mime validation, size limit, queue.
   - File list in a card grid with mime-aware previews (image thumbnail via signed URL, pdf icon, etc.).
   - `GET /files/:id/url` → redirect to signed URL (do not proxy bytes).
5. Centralize the **API client** (`src/lib/api/client.ts`):
   - Typed fetch wrapper.
   - Zod-parses every response.
   - Normalizes errors into `{ code, message, fields? }`.
   - 401 → attempts refresh once → on second 401, redirects to `/login`.

**Exit criteria:** Every list screen supports `?q=…&status=…&page=…` and the URL fully reconstructs the view. A user can be invited, accept the invite, log in, and see the new workspace. A file can be uploaded, listed, previewed, and downloaded via signed URL.

**Outcomes:** Modules 33, 34, 36 in full, and the start of Module 35 (the data layer that powers the dashboard in Session 3).

### Session 3 — "Dashboard & ship: charts, role-based UI, deploy to Vercel"

**Module references:** 35 (charts & analytics), plus 31 (RBAC in the UI), 32 (responsive dashboard), 36 (architecture).

**You will:**

1. Build the **analytics dashboard** at `app/(dashboard)/[workspaceSlug]/dashboard`:
   - **Stat cards** at the top: total projects, open tasks, members, storage used (with progress bar against quota).
   - **Time-series chart**: line chart of `tasks_created` per day, with a date-range picker (last 7 / 30 / 90 days, or custom). Backend's `timeseries` endpoint returns `[{ bucket, value }]`. Recharts `<LineChart>` with a brush for zoom.
   - **Top contributors** bar chart (horizontal): top 10 members by `tasks_completed` in the selected range.
   - **Status breakdown** area chart: stacked area of tasks by status over time.
2. Implement **role-based UI**:
   - A `useWorkspaceRole()` hook that reads from the session.
   - `<Can role="admin">` component (or `if (role === 'admin') …`) wrapping destructive actions.
   - Server-side: protected server components also check the role, never trust the client.
   - Owner-only actions (delete workspace, transfer ownership) are gated everywhere.
3. **Responsive**:
   - Sidebar collapses to a sheet on mobile (use shadcn `Sheet`).
   - Tables become cards on `<sm`.
   - All dialogs and dropdowns are touch-friendly.
4. **Polish**:
   - Empty states with illustrations and a single CTA.
   - Skeletons on every async surface.
   - Error boundaries with a "Try again" button.
   - Toast notifications for success/failure.
   - Focus rings on every interactive element.
   - Keyboard shortcuts: `Cmd-K` palette, `g p` go to projects, `g d` go to dashboard.
5. **Observability**:
   - Wire Sentry (`@sentry/nextjs`). Capture unhandled errors and one tagged transaction per page.
   - Add a `WebVitals` reporter.
6. **Tests**:
   - One Playwright E2E: register → create workspace → invite → accept → upload file → filter table → open dashboard.
   - Component tests for the most-used forms (login, create workspace, create task).
7. **Deploy to Vercel**:
   - Connect the GitHub repo.
   - Set env vars in the Vercel dashboard (`NEXT_PUBLIC_API_URL`, `SENTRY_DSN`).
   - Configure the production domain.
   - Add a `vercel.json` if you need redirects or headers.
8. **README**: screenshots, the live URL, env table, "how to point this at a different backend" section.

**Exit criteria:** A reviewer can sign in to the live URL with the seed credentials, click around every screen, upload a file, and see the analytics chart populate — all without a single console error or a single broken layout at any viewport from 360 px to 1920 px.

**Outcomes:** Modules 35 and 36 in full, and a portfolio piece you can show in interviews.

---

## 4. Tech Stack (required)

| Layer | Choice | Pin |
|---|---|---|
| Framework | Next.js (App Router) | 15.x |
| Runtime | React | 19 |
| Language | TypeScript | 5.x, `"strict": true` |
| Styling | Tailwind CSS | 4.x |
| Primitives | shadcn/ui (Radix under the hood) | latest |
| Data fetching | TanStack Query | 5.x |
| Tables | TanStack Table | 8.x |
| URL state | `nuqs` (or `useSearchParams`) | latest |
| Forms | React Hook Form + Zod | latest |
| Charts | Recharts | 2.x |
| Theming | `next-themes` | latest |
| Errors | `@sentry/nextjs` | 8.x |
| Tests | Vitest + Testing Library + Playwright | latest |
| Package manager | `pnpm` | 9.x |
| Deploy | Vercel | — |

**Forbidden:** `localStorage` for auth tokens. Mocking the backend in production. `any` in TypeScript without a comment explaining why. Tailwind class soup with no design tokens. Chart libraries other than Recharts without an instructor exception.

---

## 5. Grading Rubric (100 points)

| Category | Pts | What "full marks" looks like |
|---|---:|---|
| **App shell & auth** | 15 | Login/register/refresh/logout work. Middleware protects routes. Unauthenticated redirects preserve `next`. Workspace switcher works. Theme toggle persists. No auth tokens in `localStorage`. |
| **Data screens (tables + forms)** | 20 | Every list page uses TanStack Table. Search, filter, sort, and pagination are all in **URL state** (shareable). Forms validate with Zod, surface server errors per-field, and have loading + error + empty states. Mutations use optimistic updates where appropriate. |
| **Dashboard & charts** | 15 | All three analytics endpoints (overview, timeseries, top-contributors) are rendered. Stat cards, time-series line, top-contributors bar, status area. Date range filtering works. Charts are accessible (aria-labels, not color-only). |
| **Responsive & polished** | 10 | Works at 360 / 768 / 1280 / 1920 px. Sidebar collapses. Tables reflow. Empty states have copy + CTA. Skeletons everywhere. No layout shift on load. Focus rings visible. Reduced-motion respected. |
| **State management** | 10 | TanStack Query used for server state. No prop-drilling beyond two levels (or use a small store). Server components for initial loads, client components for interactivity. No duplicated state. |
| **Performance** | 10 | LCP < 2.5 s on Vercel preview. No unnecessary client components. Images use `next/image`. Fonts use `next/font`. Bundle is split (Lighthouse shows route-level chunks). No N+1 fetches in lists. |
| **Deployment** | 5 | Live Vercel URL works. Env vars set in the dashboard (not hard-coded). Custom 404 page. `next/image` domains configured if needed. |
| **Tests** | 5 | One Playwright E2E that exercises the full happy path. At least three component tests. CI is green. |
| **README** | 5 | Live URL, screenshots, env table, "how to point at a different backend", architecture notes, design decisions (especially anything non-obvious about RSC vs client, or why a state choice was made). |
| **Code review rubric** | 5 | Consistent naming, no dead code, no commented-out blocks, ESLint clean, Prettier clean, no `console.log` left in, no unused imports, sensible file structure (`components/<domain>/<Component>.tsx`). |

**Late / missing:** −10 per day late, 0 after 7 days. A live URL that 500s is an automatic −10 and a 24-hour fix window.

**Bonus (up to +10):** Custom domain (+1), `cmd-K` command palette (+2), optimistic mutations on the most-used action (+1), dark-mode design is **actually designed** (not just inverted) (+2), accessibility audit (axe) clean (+2), one feature the backend doesn't have yet (you added it to both repos, with tests) (+2).

---

## 6. Submission Rules

1. **One repo per student.** Repo name: `saas-frontend-capstone`. `README.md` at the root.
2. **Backend dependency.** The README's "Backend" section must link to the Assignment 4 backend (yours or a teammate's) and document the `NEXT_PUBLIC_API_URL` env var. The grader will run the frontend against their own backend using your env table.
3. **No secrets in the repo.** `.env.example` only. The grader supplies `.env.local` for grading.
4. **CI must be green.** GitHub Actions runs `lint`, `typecheck`, `test` (Vitest) and the Playwright E2E against a deployed preview. A red main branch is an automatic −5.
5. **Tag your final submission.** `git tag submission-v1 && git push --tags`. The grader checks out the tag, runs `pnpm i && pnpm build`, and visits your Vercel URL.
6. **Demo video or screenshots.** A 2-minute walkthrough (or 10–15 screenshots) covering: login, workspace switcher, members, invite flow, file upload, table with URL state, analytics dashboard, dark mode.
7. **Plagiarism policy.** You may discuss concepts, but your code is your own. The grader runs every submission through a code-similarity check. Two submissions with > 40% similarity both get zero.
8. **AI assistance.** You may use AI tools, but you must (a) be able to explain every line in a 10-minute oral, and (b) include an `AI_LOG.md` listing which tools you used for what.

**Submission form:** [link to your cohort's form]. Paste: repo URL, tag, Vercel live URL, demo video URL.

---

## 7. AI Industry Note — "The Frontend Is the Product"

There is a saying in product engineering: **"the backend is the product, but the frontend is the experience."** Your users will never see your carefully normalized schema, your cursor pagination, your signed URLs. They will see a button. They will see a loading spinner. They will see an error message they cannot parse. **All of that is the frontend's job.**

Three things to internalize before you ship.

**First: state has a home.** Server state (data from the API) lives in TanStack Query — never in `useState`, never in `useReducer`, never in a global store. URL state (filters, pagination, sort, tab) lives in the URL — never in `useState`, because a filtered view must be shareable and back-button-restorable. UI state (open/closed, hover, focus) lives in the component — and if it has to be shared, it goes in a `zustand` store or a context, not in a query. The mistake every junior makes is putting everything in `useState` and ending up with three sources of truth that drift apart. Pick the right home, every time.

**Second: the loading state is a feature, not a bug.** When a user clicks "Save," three things happen: the button disables, a spinner appears, and a toast confirms. Skeletons on initial load are non-negotiable. Optimistic updates on common actions ("mark as done", "star", "delete") are the difference between an app that feels native and one that feels like a 2010 webform. None of this is hard; all of it is discipline.

**Third: design is not decoration. Design is the contract.** When the same `Button` is reused on 20 screens, you are communicating "this is the same kind of action." When `Card` is reused, you are communicating "this is the same kind of content." When the spacing scale is `4 / 8 / 12 / 16 / 24 / 32` and not `5 / 7 / 13`, you are communicating "we have a system." shadcn/ui gives you the system for free — but you must commit to it. No "this one button needs to be 13 px." It doesn't. Use `size="sm"`. The senior engineer's job is to push back on one-off design choices; yours is to use the system until you have a reason not to.

The next module (37) and beyond extend this frontend with internationalization, accessibility audits, and a small design system you publish as a package. For now: ship a real product, not a wireframe. The graders — and your future hiring managers — can tell the difference.
