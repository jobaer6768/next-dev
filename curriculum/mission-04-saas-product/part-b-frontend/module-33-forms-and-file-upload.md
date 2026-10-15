# Module 33: Forms, Validation & File Upload

> Mission 4 · Part B · Frontend — Forms are the connective tissue between your user and your database. Get them right, and the product feels trustworthy. Get them wrong, and nothing else matters.

## Why this matters

Every SaaS product is, at its core, a glorified form-submission engine wrapped in analytics. Onboarding flows, settings pages, project creation, file uploads, invitations — they're all forms. If your forms are slow, lossy, or hostile on errors, users will leave. If they share validation logic between client and server, you ship faster and avoid the classic "works in dev, breaks in prod" class of bugs.

In React 19 + Next.js 15, we have a new generation of primitives: `useOptimistic` for instant UI feedback, server actions for clean mutations, and the React Hook Form + Zod combo is now the de facto standard. By the end of this module you'll be able to build any form a PM throws at you — including ones that span multiple steps, upload files with progress bars, and survive flaky networks.

## Learning objectives

By the end of this module you will be able to:

1. Build performant forms with **React Hook Form** (`useForm`, `Controller`, `useFieldArray`).
2. Share a **Zod schema** between client and server so the same rules run in both places.
3. Handle **server-side validation errors** and map them back onto the correct fields.
4. Compose a **multi-step wizard** with persisted state across steps.
5. Build a **drag-and-drop file upload** component with preview and progress.
6. Use React 19's **`useOptimistic`** for instant UI updates on slow networks.
7. Build forms that feel fast even on a 3G connection.

## Conceptual model

### 1. The single source of truth: a shared Zod schema

The single biggest mistake I see on bootcamp projects is duplicating validation rules. The frontend says "email is required", the backend says "email must be a string of length 5-100" — and they drift. The fix is to make the schema a shared module.

**`lib/schemas/project.ts`** — shared by client and server:

```typescript
import { z } from "zod";

export const ACCEPTED_IMAGE_TYPES = [
  "image/jpeg",
  "image/jpg",
  "image/png",
  "image/webp",
] as const;

export const MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB

export const projectSchema = z.object({
  name: z
    .string()
    .min(3, "Project name must be at least 3 characters")
    .max(80, "Project name is too long"),
  description: z
    .string()
    .min(20, "Tell us at least a sentence about the project")
    .max(2000),
  visibility: z.enum(["private", "team", "public"]),
  tags: z
    .array(z.string().min(1).max(24))
    .min(1, "Add at least one tag")
    .max(8, "No more than 8 tags"),
  coverImage: z
    .instanceof(File)
    .refine((f) => f.size <= MAX_FILE_SIZE, "Max file size is 5MB")
    .refine(
      (f) => ACCEPTED_IMAGE_TYPES.includes(f.type as any),
      "Only .jpg, .jpeg, .png, and .webp are supported"
    )
    .optional(),
  members: z
    .array(
      z.object({
        email: z.string().email("Not a valid email"),
        role: z.enum(["viewer", "editor", "admin"]),
      })
    )
    .default([]),
});

export type ProjectInput = z.infer<typeof projectSchema>;
```

On the server, you parse `FormData` (because of the file upload) into a plain object, then run `projectSchema.parse()` — and on failure you throw a typed error. The exact same schema runs on the client to give instant field-level feedback.

### 2. A basic form with React Hook Form + Zod

**`components/features/projects/project-form.tsx`**

```tsx
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { projectSchema, type ProjectInput } from "@/lib/schemas/project";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import {
  Form,
  FormControl,
  FormDescription,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";

export function ProjectForm({
  defaultValues,
  onSubmit,
}: {
  defaultValues?: Partial<ProjectInput>;
  onSubmit: (values: ProjectInput) => Promise<void>;
}) {
  const form = useForm<ProjectInput>({
    resolver: zodResolver(projectSchema),
    defaultValues: {
      name: "",
      description: "",
      visibility: "team",
      tags: [],
      members: [],
      ...defaultValues,
    },
    mode: "onBlur", // validate when the field loses focus
  });

  const { isSubmitting } = form.formState;

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        <FormField
          control={form.control}
          name="name"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Project name</FormLabel>
              <FormControl>
                <Input placeholder="Acme Dashboard" {...field} />
              </FormControl>
              <FormDescription>
                This is how your project appears in the sidebar.
              </FormDescription>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="description"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Description</FormLabel>
              <FormControl>
                <Textarea
                  placeholder="What is this project about?"
                  rows={4}
                  {...field}
                />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="visibility"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Visibility</FormLabel>
              <Select onValueChange={field.onChange} value={field.value}>
                <FormControl>
                  <SelectTrigger>
                    <SelectValue placeholder="Choose visibility" />
                  </SelectTrigger>
                </FormControl>
                <SelectContent>
                  <SelectItem value="private">Private (just me)</SelectItem>
                  <SelectItem value="team">Team (invite only)</SelectItem>
                  <SelectItem value="public">Public (anyone with link)</SelectItem>
                </SelectContent>
              </Select>
              <FormMessage />
            </FormItem>
          )}
        />

        <Button type="submit" disabled={isSubmitting}>
          {isSubmitting ? "Saving…" : "Save project"}
        </Button>
      </form>
    </Form>
  );
}
```

The pattern above is mostly boilerplate. Once you have `<Form />` and its sub-components set up, the only "real" code per field is the `<FormField>` and the inputs. shadcn/ui gives you all of those pieces.

### 3. Field arrays: dynamic lists of inputs

`useFieldArray` is the React Hook Form API for an array of fields — perfect for "add another tag", "invite another member", "add another line item".

```tsx
import { useFieldArray } from "react-hook-form";
import { Trash2, Plus } from "lucide-react";

export function MemberInviteList({ form }: { form: UseFormReturn<ProjectInput> }) {
  const { fields, append, remove } = useFieldArray({
    control: form.control,
    name: "members",
  });

  return (
    <div className="space-y-3">
      {fields.map((field, index) => (
        <div key={field.id} className="flex items-start gap-2">
          <FormField
            control={form.control}
            name={`members.${index}.email`}
            render={({ field: f }) => (
              <FormItem className="flex-1">
                <FormControl>
                  <Input type="email" placeholder="teammate@acme.com" {...f} />
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />
          <FormField
            control={form.control}
            name={`members.${index}.role`}
            render={({ field: f }) => (
              <FormItem>
                <Select onValueChange={f.onChange} value={f.value}>
                  <FormControl>
                    <SelectTrigger className="w-32">
                      <SelectValue />
                    </SelectTrigger>
                  </FormControl>
                  <SelectContent>
                    <SelectItem value="viewer">Viewer</SelectItem>
                    <SelectItem value="editor">Editor</SelectItem>
                    <SelectItem value="admin">Admin</SelectItem>
                  </SelectContent>
                </Select>
                <FormMessage />
              </FormItem>
            )}
          />
          <Button
            type="button"
            variant="ghost"
            size="icon"
            onClick={() => remove(index)}
            aria-label="Remove invite"
          >
            <Trash2 className="h-4 w-4" />
          </Button>
        </div>
      ))}
      <Button
        type="button"
        variant="outline"
        onClick={() => append({ email: "", role: "viewer" })}
      >
        <Plus className="h-4 w-4 mr-2" /> Invite another teammate
      </Button>
    </div>
  );
}
```

The `field.id` from `useFieldArray` is critical — it's the stable key React needs to animate the list correctly.

### 4. Mapping server validation errors back onto the form

This is the bit every bootcamp skips, and the bit that separates a working product from a polished one. When the server rejects a payload, it returns Zod's `flatten()` shape:

```json
{
  "formErrors": ["members: at most 8 invites"],
  "fieldErrors": {
    "name": ["Name is reserved"],
    "members.0.email": ["Already invited"]
  }
}
```

Write a small helper that converts this into `setError` calls on your RHF form:

**`lib/forms/apply-server-errors.ts`**

```typescript
import type { UseFormReturn, FieldValues, Path } from "react-hook-form";
import { z } from "zod";

type Flat = z.inferFlattenedErrors<any>["fieldErrors"];

export function applyServerErrors<T extends FieldValues>(
  form: UseFormReturn<T>,
  fieldErrors: Flat,
  formErrors?: string[]
) {
  for (const [path, messages] of Object.entries(fieldErrors)) {
    if (!messages?.length) continue;
    form.setError(path as Path<T>, { message: messages[0] });
  }
  if (formErrors?.length) {
    form.setError("root.serverError", { message: formErrors.join(", ") });
  }
}
```

On the server, return the flattened errors:

```typescript
// app/api/projects/route.ts
import { NextResponse } from "next/server";
import { projectSchema } from "@/lib/schemas/project";

export async function POST(req: Request) {
  const body = await req.json();
  const parsed = projectSchema.safeParse(body);
  if (!parsed.success) {
    return NextResponse.json(
      { ok: false, errors: parsed.error.flatten() },
      { status: 422 }
    );
  }
  // ...create project
  return NextResponse.json({ ok: true, project });
}
```

On the client, the `onSubmit` handler:

```typescript
async function onSubmit(values: ProjectInput) {
  const res = await fetch("/api/projects", {
    method: "POST",
    body: JSON.stringify(values),
    headers: { "Content-Type": "application/json" },
  });
  if (!res.ok) {
    const data = await res.json();
    if (data?.errors) {
      applyServerErrors(form, data.errors.fieldErrors, data.errors.formErrors);
    }
    return;
  }
  router.push(`/projects/${(await res.json()).project.id}`);
}
```

### 5. File upload with drag-and-drop, preview, and progress

**`components/ui/file-dropzone.tsx`**

```tsx
"use client";

import { useCallback, useState } from "react";
import { useDropzone } from "react-dropzone";
import Image from "next/image";
import { UploadCloud, X } from "lucide-react";
import { cn } from "@/lib/utils";

export function FileDropzone({
  value,
  onChange,
  accept = { "image/*": [".jpg", ".jpeg", ".png", ".webp"] },
  maxSize = 5 * 1024 * 1024,
}: {
  value?: File | null;
  onChange: (file: File | null) => void;
  accept?: Record<string, string[]>;
  maxSize?: number;
}) {
  const [preview, setPreview] = useState<string | null>(null);

  const onDrop = useCallback(
    (files: File[]) => {
      const file = files[0] ?? null;
      onChange(file);
      if (file) {
        setPreview(URL.createObjectURL(file));
      }
    },
    [onChange]
  );

  const { getRootProps, getInputProps, isDragActive, fileRejections } =
    useDropzone({
      onDrop,
      accept,
      maxSize,
      multiple: false,
    });

  const rejection = fileRejections[0]?.errors[0]?.message;

  return (
    <div className="space-y-2">
      <div
        {...getRootProps()}
        className={cn(
          "border-2 border-dashed rounded-lg p-6 text-center cursor-pointer transition-colors",
          isDragActive
            ? "border-primary bg-primary/5"
            : "border-muted-foreground/25 hover:border-primary"
        )}
      >
        <input {...getInputProps()} />
        {preview && value ? (
          <div className="relative inline-block">
            <Image
              src={preview}
              alt="Preview"
              width={160}
              height={160}
              className="rounded-md object-cover"
              unoptimized
            />
            <button
              type="button"
              onClick={(e) => {
                e.stopPropagation();
                onChange(null);
                setPreview(null);
              }}
              className="absolute -top-2 -right-2 bg-background rounded-full p-1 shadow"
            >
              <X className="h-4 w-4" />
            </button>
          </div>
        ) : (
          <div className="flex flex-col items-center gap-2 text-muted-foreground">
            <UploadCloud className="h-8 w-8" />
            <p className="text-sm">
              Drag a file here, or <span className="text-primary">click to select</span>
            </p>
            <p className="text-xs">Up to {Math.round(maxSize / 1024 / 1024)}MB</p>
          </div>
        )}
      </div>
      {rejection && <p className="text-sm text-destructive">{rejection}</p>}
    </div>
  );
}
```

For upload progress, use XHR (not `fetch`, which has no progress event):

```typescript
async function uploadWithProgress(
  url: string,
  file: File,
  onProgress: (pct: number) => void
): Promise<{ url: string }> {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    const form = new FormData();
    form.append("file", file);
    xhr.open("POST", url);
    xhr.upload.onprogress = (e) => {
      if (e.lengthComputable) onProgress((e.loaded / e.total) * 100);
    };
    xhr.onload = () => {
      if (xhr.status >= 200 && xhr.status < 300) {
        resolve(JSON.parse(xhr.responseText));
      } else reject(new Error(xhr.responseText));
    };
    xhr.onerror = () => reject(new Error("Network error"));
    xhr.send(form);
  });
}
```

### 6. Optimistic UI with React 19's `useOptimistic`

`useOptimistic` lets you update the UI as if the request already succeeded, then reconcile when the server responds. It's the difference between a Slack that feels instant and one that feels laggy.

**Server action first — `app/projects/actions.ts`:**

```typescript
"use server";
import { revalidatePath } from "next/cache";

export async function addTodo(formData: FormData) {
  const title = formData.get("title") as string;
  // ...persist to DB
  revalidatePath("/projects/[id]", "page");
  return { id: crypto.randomUUID(), title };
}
```

**Optimistic client — `components/features/todos/todo-list.tsx`:**

```tsx
"use client";

import { useOptimistic, useState } from "react";
import { addTodo } from "@/app/projects/actions";

type Todo = { id: string; title: string };

export function TodoList({ initial }: { initial: Todo[] }) {
  const [todos, setTodos] = useState<Todo[]>(initial);
  const [optimistic, addOptimistic] = useOptimistic(
    todos,
    (state, newTitle: string) => [
      ...state,
      { id: `optimistic-${Date.now()}`, title: newTitle },
    ]
  );

  async function action(formData: FormData) {
    const title = formData.get("title") as string;
    addOptimistic(title); // immediate
    const saved = await addTodo(formData);
    setTodos((prev) => [...prev, saved]); // reconcile
  }

  return (
    <form action={action} className="space-y-3">
      <input name="title" placeholder="What needs doing?" required />
      <button>Add</button>
      <ul>
        {optimistic.map((t) => (
          <li key={t.id} className={t.id.startsWith("optimistic-") ? "opacity-60" : ""}>
            {t.title}
          </li>
        ))}
      </ul>
    </form>
  );
}
```

The new todo appears instantly, with reduced opacity to signal it's pending. When the server action returns, the real id replaces the optimistic one and opacity goes to 1.

### 7. Multi-step wizard state with Zustand

For a 3- or 4-step wizard, RHF is fine for each step's fields but you'll want a separate store to track "which step am I on" and "what's the cumulative data".

**`stores/wizard-store.ts`:**

```typescript
import { create } from "zustand";
import { persist } from "zustand/middleware";
import type { ProjectInput } from "@/lib/schemas/project";

type WizardState = {
  step: number;
  data: Partial<ProjectInput>;
  setStep: (s: number) => void;
  patch: (d: Partial<ProjectInput>) => void;
  reset: () => void;
};

export const useWizard = create<WizardState>()(
  persist(
    (set) => ({
      step: 0,
      data: {},
      setStep: (step) => set({ step }),
      patch: (data) => set((s) => ({ data: { ...s.data, ...data } })),
      reset: () => set({ step: 0, data: {} }),
    }),
    { name: "project-wizard" }
  )
);
```

In each step, use `form.reset(wizard.data)` to hydrate, then on `onSubmit` call `wizard.patch(values)` + `wizard.setStep(step + 1)`. Persist means a user can close the tab and resume.

## Common mistakes

- **Validating only on the client.** Anything user-controlled must be re-validated on the server. The shared schema makes this cheap.
- **Calling `form.setError("name", ...)` from a server response without flattening.** Server errors nest. You must map `fieldErrors["name"]` to `setError("name", ...)` per the helper above.
- **Forgetting `key={field.id}` in `useFieldArray`.** RHF's stable ids exist for a reason — without them, removals will re-render the wrong rows.
- **Uploading files via `fetch` for progress.** `fetch` doesn't expose upload progress. Use XHR.
- **Not un-revoking object URLs.** `URL.createObjectURL` leaks memory. Call `URL.revokeObjectURL` on cleanup, or use `useEffect(() => () => URL.revokeObjectURL(preview), [preview])`.
- **Using `useOptimistic` outside a Transition or form action.** It throws if called in an effect. The whole point is that you call it before the async work.

## Exercise

Build a **"Create Project"** page at `/projects/new` with three steps:

1. **Basics**: name, description, visibility (uses RHF + Zod).
2. **Team**: invite up to 8 teammates using `useFieldArray`. Validate unique emails (custom Zod refine).
3. **Cover**: drag-and-drop a cover image. Show preview, reject >2MB, then upload to `/api/upload` with progress bar and use `useOptimistic` to show a placeholder thumbnail in a "Recent projects" list while the upload runs.

Requirements:
- The wizard state must persist across page reloads.
- The server must reject duplicates with a friendly field error.
- A11y: every input has a `<label>`, errors use `aria-describedby`, and the progress bar has `aria-valuenow`.
- Test it by throttling your network in DevTools to "Slow 3G" and confirm the UI feels instant.

## AI Industry Note (2026)

In 2026, AI-assisted form-filling is now table-stakes. Modern SaaS UIs offer inline "suggested values" powered by an LLM that reads prior projects in the workspace and pre-fills the description. Several production libraries — **Inferable**, **Reflag**, **Vercel AI SDK with structured output** — provide form-completion UIs that you wire into a Zod schema and the model returns schema-conformant JSON. The skill you're learning here — schema-first design — is exactly the contract those models use. If your schema is messy, the model can't fill it; if your schema is clean, you get a 10x UX win for free. Treat the Zod schema as your public API for *both* the human and the machine.
