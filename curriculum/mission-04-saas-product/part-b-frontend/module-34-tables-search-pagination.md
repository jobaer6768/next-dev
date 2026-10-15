# Module 34: Data Tables, Search, Filtering & Pagination

> Mission 4 · Part B · Frontend — A great data table is a product. A bad one is a spreadsheet the user can't quit. Learn to build tables that scale to 10,000 rows without dropping a frame.

## Why this matters

Every admin dashboard, every "users" page, every billing portal, every analytics list — it's a data table. The bootcamp version of a data table is `array.map(renderRow)`. The production version handles **sorting, filtering, pagination, selection, bulk actions, column visibility, URL state, and loading states** for tables that may have millions of rows. You're not going to ship a list of 1.2M Stripe charges rendered as DOM nodes.

By the end of this module you'll have a reusable data-table primitive that you can drop into any feature, with server-side pagination, URL-driven filters, debounced search, and shareable views. This is the single most reused component in any real SaaS codebase.

## Learning objectives

By the end of this module you will be able to:

1. Build a **headless data table** with **TanStack Table v8** and style it with shadcn/ui.
2. Wire up **server-side pagination** (offset and cursor strategies).
3. Implement **debounced search** with a custom `useDebouncedValue` hook.
4. Drive **filters and sort** from URL search params so views are shareable.
5. Add **row selection** and **bulk actions** (delete, archive, export).
6. Compose a feature-level "tasks" page that uses the primitive.

## Conceptual model

### 1. The TanStack Table mental model

TanStack Table is **headless**. It does not render DOM. It takes your data + columns, and gives you back a model of what's visible, sorted, selected, and paged. You then `flex render()` it however you like. This is the right abstraction: the table engine and the table styling are independent concerns.

The data flow:

```
   raw data
   (from API / props)
        |
        v
   useReactTable({ data, columns, state, onStateChange })
        |
        v
   table.getRowModel(), table.getHeaderGroups(), table.getState()
        |
        v
   your JSX renders rows, headers, pagination
```

A column is a definition, not a React component. You attach `accessorKey`, `header`, optional `cell` renderer, and sort/filter functions.

### 2. The reusable `<DataTable />` primitive

**`components/ui/data-table.tsx`**

```tsx
"use client";

import {
  ColumnDef,
  flexRender,
  getCoreRowModel,
  useReactTable,
  type Table as TanstackTable,
} from "@tanstack/react-table";
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";

interface DataTableProps<TData, TValue> {
  columns: ColumnDef<TData, TValue>[];
  data: TData[];
  // controlled state (optional)
  state?: any;
  onStateChange?: (state: any) => void;
  // row selection column
  enableSelection?: boolean;
  // toolbar slot
  toolbar?: (table: TanstackTable<TData>) => React.ReactNode;
}

export function DataTable<TData, TValue>({
  columns,
  data,
  state,
  onStateChange,
  enableSelection = false,
  toolbar,
}: DataTableProps<TData, TValue>) {
  const table = useReactTable({
    data,
    columns,
    state,
    onStateChange,
    getCoreRowModel: getCoreRowModel(),
    manualPagination: true,
    manualSorting: true,
    manualFiltering: true,
    enableRowSelection: enableSelection,
  });

  return (
    <div className="space-y-4">
      {toolbar?.(table)}
      <div className="rounded-md border">
        <Table>
          <TableHeader>
            {table.getHeaderGroups().map((hg) => (
              <TableRow key={hg.id}>
                {hg.headers.map((header) => (
                  <TableHead key={header.id}>
                    {header.isPlaceholder
                      ? null
                      : flexRender(
                          header.column.columnDef.header,
                          header.getContext()
                        )}
                  </TableHead>
                ))}
              </TableRow>
            ))}
          </TableHeader>
          <TableBody>
            {table.getRowModel().rows.length ? (
              table.getRowModel().rows.map((row) => (
                <TableRow key={row.id}>
                  {row.getVisibleCells().map((cell) => (
                    <TableCell key={cell.id}>
                      {flexRender(cell.column.columnDef.cell, cell.getContext())}
                    </TableCell>
                  ))}
                </TableRow>
              ))
            ) : (
              <TableRow>
                <TableCell colSpan={columns.length} className="h-24 text-center">
                  No results.
                </TableCell>
              </TableRow>
            )}
          </TableBody>
        </Table>
      </div>
    </div>
  );
}
```

Notice the `manual*` flags — they tell TanStack "don't try to sort/paginate/filter client-side, the server will do it". This is the production setting once your data grows past a few hundred rows.

### 3. Column definitions and the row-actions pattern

**`app/(app)/projects/_components/columns.tsx`**

```tsx
"use client";

import { ColumnDef } from "@tanstack/react-table";
import { MoreHorizontal, ArrowUpDown } from "lucide-react";
import { Button } from "@/components/ui/button";
import { Checkbox } from "@/components/ui/checkbox";
import { Badge } from "@/components/ui/badge";
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu";
import Link from "next/link";
import type { Project } from "@/types";

export const columns: ColumnDef<Project>[] = [
  // selection checkbox
  {
    id: "select",
    header: ({ table }) => (
      <Checkbox
        checked={table.getIsAllPageRowsSelected()}
        onCheckedChange={(v) => table.toggleAllPageRowsSelected(!!v)}
        aria-label="Select all"
      />
    ),
    cell: ({ row }) => (
      <Checkbox
        checked={row.getIsSelected()}
        onCheckedChange={(v) => row.toggleSelected(!!v)}
        aria-label="Select row"
      />
    ),
    enableSorting: false,
    enableHiding: false,
  },
  {
    accessorKey: "name",
    header: ({ column }) => (
      <Button
        variant="ghost"
        onClick={() => column.toggleSorting(column.getIsSorted() === "asc")}
      >
        Name <ArrowUpDown className="ml-2 h-4 w-4" />
      </Button>
    ),
    cell: ({ row }) => (
      <Link
        href={`/projects/${row.original.id}`}
        className="font-medium hover:underline"
      >
        {row.original.name}
      </Link>
    ),
  },
  {
    accessorKey: "status",
    header: "Status",
    cell: ({ row }) => {
      const status = row.original.status;
      return (
        <Badge variant={status === "active" ? "default" : "secondary"}>
          {status}
        </Badge>
      );
    },
  },
  {
    accessorKey: "owner",
    header: "Owner",
  },
  {
    accessorKey: "createdAt",
    header: "Created",
    cell: ({ row }) => new Date(row.original.createdAt).toLocaleDateString(),
  },
  {
    id: "actions",
    cell: ({ row }) => (
      <DropdownMenu>
        <DropdownMenuTrigger asChild>
          <Button variant="ghost" size="icon">
            <MoreHorizontal className="h-4 w-4" />
          </Button>
        </DropdownMenuTrigger>
        <DropdownMenuContent align="end">
          <DropdownMenuItem asChild>
            <Link href={`/projects/${row.original.id}`}>Open</Link>
          </DropdownMenuItem>
          <DropdownMenuItem>Duplicate</DropdownMenuItem>
          <DropdownMenuItem className="text-destructive">
            Delete
          </DropdownMenuItem>
        </DropdownMenuContent>
      </DropdownMenu>
    ),
  },
];
```

The pattern is: **one object per column, render-function in `header` and `cell` for any interactivity**. Sortable columns get a button header. Selectable tables get a checkbox column with `id: "select"`.

### 4. URL-driven filters and sort

Make the table state live in the URL. Why?

- The user can share `/projects?status=active&sort=-createdAt` and a colleague sees the same view.
- The browser back button "just works".
- Server components can read the same params to render the initial page.

**`lib/tables/parse-search-params.ts`**

```typescript
export type ListParams = {
  page: number;
  pageSize: number;
  sort?: string; // "field" or "-field"
  search?: string;
  status?: "active" | "archived" | "all";
  ownerId?: string;
};

export function parseListParams(sp: URLSearchParams): ListParams {
  const sort = sp.get("sort") ?? undefined;
  return {
    page: Number(sp.get("page") ?? 1),
    pageSize: Number(sp.get("pageSize") ?? 25),
    sort,
    search: sp.get("q") ?? undefined,
    status: (sp.get("status") as ListParams["status"]) ?? "all",
    ownerId: sp.get("ownerId") ?? undefined,
  };
}

export function buildSearchString(p: Partial<ListParams>): string {
  const usp = new URLSearchParams();
  for (const [k, v] of Object.entries(p)) {
    if (v != null && v !== "") usp.set(k, String(v));
  }
  return usp.toString();
}
```

The page that renders the table:

**`app/(app)/projects/page.tsx`**

```tsx
import { columns } from "./_components/columns";
import { ProjectsTableClient } from "./_components/projects-table-client";
import { listProjects } from "@/services/projects";
import { parseListParams } from "@/lib/tables/parse-search-params";

export default async function ProjectsPage({
  searchParams,
}: {
  searchParams: Promise<Record<string, string | string[] | undefined>>;
}) {
  const sp = new URLSearchParams(
    Object.entries(await searchParams).flatMap(([k, v]) =>
      Array.isArray(v) ? v.map((x) => [k, x]) : v ? [[k, v]] : []
    )
  );
  const params = parseListParams(sp);
  const { rows, total } = await listProjects(params);

  return (
    <ProjectsTableClient
      rows={rows}
      total={total}
      page={params.page}
      pageSize={params.pageSize}
      sort={params.sort}
      search={params.search}
      status={params.status}
    />
  );
}
```

The client component reads these props and **re-renders the table** when the user changes sort, filter, or search — pushing the new state back into the URL via `router.push`.

### 5. Debounced search

Never send a request on every keystroke. The classic 300ms debounce gives users time to finish typing.

**`hooks/use-debounced-value.ts`**

```typescript
import { useEffect, useState } from "react";

export function useDebouncedValue<T>(value: T, delay = 300): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  return debounced;
}
```

Use it in a search input that pushes to the URL:

```tsx
"use client";

import { useRouter, useSearchParams } from "next/navigation";
import { useEffect, useState } from "react";
import { Search } from "lucide-react";
import { useDebouncedValue } from "@/hooks/use-debounced-value";
import { Input } from "@/components/ui/input";

export function SearchInput() {
  const router = useRouter();
  const sp = useSearchParams();
  const [value, setValue] = useState(sp.get("q") ?? "");
  const debounced = useDebouncedValue(value, 300);

  useEffect(() => {
    const next = new URLSearchParams(sp);
    if (debounced) next.set("q", debounced);
    else next.delete("q");
    next.delete("page"); // reset paging on search
    router.push(`?${next.toString()}`);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [debounced]);

  return (
    <div className="relative">
      <Search className="absolute left-3 top-2.5 h-4 w-4 text-muted-foreground" />
      <Input
        value={value}
        onChange={(e) => setValue(e.target.value)}
        placeholder="Search projects…"
        className="pl-9 w-72"
      />
    </div>
  );
}
```

### 6. Server-side pagination — offset and cursor

**Offset (the simple one):**

```typescript
// services/projects.ts
import { db } from "@/lib/db";

export async function listProjects(p: ListParams) {
  const where = {
    ...(p.status && p.status !== "all" ? { status: p.status } : {}),
    ...(p.ownerId ? { ownerId: p.ownerId } : {}),
    ...(p.search
      ? { name: { contains: p.search, mode: "insensitive" as const } }
      : {}),
  };

  const [rows, total] = await Promise.all([
    db.project.findMany({
      where,
      orderBy: p.sort
        ? { [p.sort.replace("-", "")]: p.sort.startsWith("-") ? "desc" : "asc" }
        : { createdAt: "desc" },
      skip: (p.page - 1) * p.pageSize,
      take: p.pageSize,
    }),
    db.project.count({ where }),
  ]);

  return { rows, total };
}
```

**Cursor (the scalable one)** — for tables that may be filtered, sorted, and updated concurrently:

```typescript
export async function listProjectsCursor(p: { cursor?: string; take: number }) {
  const items = await db.project.findMany({
    take: p.take + 1,
    ...(p.cursor ? { cursor: { id: p.cursor }, skip: 1 } : {}),
    orderBy: { createdAt: "desc" },
  });
  const nextCursor = items.length > p.take ? items.pop()!.id : null;
  return { items, nextCursor };
}
```

Offset is fine up to ~100k rows. Cursor is the right call once you have write contention (rows being inserted/deleted between pages) or you're scrolling infinitely.

### 7. Bulk actions and the toolbar

The toolbar gets `table` as a prop and can do anything with selection:

```tsx
function ProjectsToolbar({ table }: { table: TanstackTable<Project> }) {
  const selected = table.getFilteredSelectedRowModel().rows;
  const router = useRouter();
  const utils = api.useUtils();

  async function bulkDelete() {
    const ids = selected.map((r) => r.original.id);
    await fetch("/api/projects/bulk-delete", {
      method: "POST",
      body: JSON.stringify({ ids }),
    });
    table.resetRowSelection();
    router.refresh();
  }

  return (
    <div className="flex items-center justify-between">
      <SearchInput />
      <StatusFilter />
      {selected.length > 0 && (
        <div className="flex items-center gap-2">
          <span className="text-sm text-muted-foreground">
            {selected.length} selected
          </span>
          <Button variant="destructive" onClick={bulkDelete}>
            <Trash2 className="h-4 w-4 mr-2" /> Delete
          </Button>
          <Button variant="outline">Archive</Button>
        </div>
      )}
    </div>
  );
}
```

Pass it to the table via the `toolbar` slot in the `DataTable`:

```tsx
<DataTable
  data={rows}
  columns={columns}
  state={tableState}
  onStateChange={setTableState}
  enableSelection
  toolbar={(t) => <ProjectsToolbar table={t} />}
/>
```

## Common mistakes

- **Rendering all rows at once.** "We only have 5,000 rows" is a sentence you say in week 1 and regret in month 6. Always paginate server-side.
- **Putting sort/filter state in component state instead of the URL.** The first time a user shares a link and it doesn't work, you'll fix this.
- **Not resetting `page` when the user changes the search query.** They search, end up on page 4, get 0 results, and complain.
- **Debouncing on the wrong side.** If you debounce the *input value* in component state (good) you don't make the API slow; if you debounce the *fetch* (also fine) you may end up with stale state. The custom hook above does the former.
- **Forgetting `enableRowSelection` and wondering why checkboxes do nothing.**
- **Using `manualPagination: false` on a 10k-row table.** The browser will start to choke at 5k visible cells.

## Exercise

Build a **`/projects`** page that lists projects with:

1. Server-side pagination (25 per page) with a shadcn `<DataTablePagination />`.
2. A debounced search box that resets to page 1.
3. A status filter (`all | active | archived`) and an owner filter.
4. Sort by name, createdAt, or updatedAt (clicking the header toggles asc/desc).
5. Row selection with a "Delete selected" bulk action that requires a confirmation dialog.
6. The **full filter+sort+page+search state lives in the URL**. Try copying the URL into an incognito tab — the same view loads.
7. Add a "Copy share link" button next to the search box that copies the current URL to clipboard.

Bonus: switch to **cursor-based pagination** and add "Load more" — confirm that new rows inserted at the top don't cause duplicates.

## AI Industry Note (2026)

In 2026, "AI-native" tables mean tables that *know* what you're looking for. We're seeing three patterns ship to production:

1. **Natural-language filters**: a search box that accepts "active projects owned by Maria from last month" and translates it to a query object before sending to the server. **Vercel AI SDK + structured output** with your Zod filter schema is the de facto stack.
2. **Auto-summaries**: an LLM-generated one-line summary of each row in a `summary` column, produced by a background job and cached.
3. **Predictive sort**: the table learns from your previous clicks which column you usually care about and reorders columns on a per-user basis. Plausible, **Statsig** and **GrowthBook** ship the experimentation primitive for this.

The primitive you build in this module — clean URL state, server-side pagination, headless columns — is the same primitive those features plug into. Get the foundation right and AI features become a 50-line addition rather than a rewrite.
