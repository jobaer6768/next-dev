# Module 57: Frontend Integration — AI Features in Next.js

## Why this module matters

The backend can be perfect. If the UI is sluggish, the user thinks the AI is dumb. The frontend is where the AI features become *legible* — or where they fail. A streaming chat that doesn't render tokens as they arrive feels frozen. A document upload with no progress bar feels broken. A citation that the user can't click feels like a hallucination. A 500 from the LLM that says "Something went wrong" feels like a dead end.

This module is the bridge between the APIs you built in Modules 50–56 and the humans who pay for your product. We will use the `ai` SDK, Next.js Server Actions, and the patterns that hold up under real usage.

## Learning objectives

1. Consume streaming LLM responses in the browser with the `ai` SDK and `useChat`.
2. Build a chat UI that renders tokens, citations, and tool calls in real time.
3. Upload documents to a knowledge base with drag-and-drop, progress, and a server action.
4. Render citations as clickable footnotes with a side-panel preview.
5. Handle loading, error, and retry states gracefully.
6. Apply optimistic UI for non-streaming AI actions.

## Conceptual model

Three pillars, mapped to the components you'll build:

```
┌──────────────────────────────────────────────────────────────┐
│                       <ChatPanel />                          │
│                                                              │
│   ┌──────────────────────┐    ┌──────────────────────────┐   │
│   │ <MessageList />      │    │ <CitationDrawer />       │   │
│   │  streams tokens      │    │  side panel, PDF page    │   │
│   │  shows tool calls    │    │  preview, scroll-to      │   │
│   │  shows citations     │    │                          │   │
│   └──────────────────────┘    └──────────────────────────┘   │
│   ┌──────────────────────────────────────────────────────┐   │
│   │ <Composer />  ── input, send, stop, retry buttons   │   │
│   └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                    <KnowledgeBase />                         │
│                                                              │
│   <Dropzone />  <UploadList />  <DocStatus />                │
└──────────────────────────────────────────────────────────────┘
```

### 1. The `useChat` hook — streaming by default

Server route:

```ts
// src/app/api/chat/route.ts
import { streamText, tool } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";

export const runtime = "edge";
export const maxDuration = 60;

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: openai("gpt-4o-mini"),
    system: "You are a helpful assistant for our SaaS. Cite sources when you can.",
    messages,
    tools: {
      searchKB: tool({
        description: "Search the user's knowledge base for relevant documents.",
        parameters: z.object({ query: z.string() }),
        execute: async ({ query }) => searchKB(query),
      }),
    },
  });

  return result.toDataStreamResponse();
}
```

Client component:

```tsx
// src/components/chat/chat-panel.tsx
"use client";
import { useChat } from "@ai-sdk/react";

export function ChatPanel({ workspaceId }: { workspaceId: string }) {
  const { messages, input, handleInputChange, handleSubmit, status, stop, reload } = useChat({
    api: "/api/chat",
    body: { workspaceId },
    onError: (err) => toast.error(`Chat failed: ${err.message}`),
  });

  return (
    <div className="flex h-full flex-col">
      <MessageList messages={messages} status={status} />
      <Composer
        input={input}
        onChange={handleInputChange}
        onSubmit={handleSubmit}
        status={status}
        onStop={stop}
        onRetry={reload}
      />
    </div>
  );
}
```

That's the entire streaming integration. The `ai` SDK handles the SSE wire format, reconnection, and the `messages` array bookkeeping. You focus on the UI.

### 2. The message list with tool calls and citations

```tsx
// src/components/chat/message-list.tsx
import { Message } from "ai/react";
import { CitationChip } from "./citation-chip";

export function MessageList({ messages, status }: { messages: Message[]; status: string }) {
  return (
    <div className="flex-1 overflow-y-auto p-4 space-y-4">
      {messages.map((m) => (
        <div key={m.id} className={`flex ${m.role === "user" ? "justify-end" : "justify-start"}`}>
          <div className="max-w-prose rounded-lg p-3 bg-muted">
            {m.content}
            {m.toolInvocations?.map((t) => (
              <details key={t.toolCallId} className="mt-2 text-xs">
                <summary>🔧 {t.toolName} {t.state === "result" ? "✓" : "…"}</summary>
                {t.state === "result" && <pre>{JSON.stringify(t.result, null, 2)}</pre>}
              </details>
            ))}
            {m.experimental_attachments?.map((a, i) => (
              <CitationChip key={i} source={a.name} index={i + 1} />
            ))}
          </div>
        </div>
      ))}
      {status === "submitted" && <TypingIndicator />}
    </div>
  );
}
```

The "submitted" state is when the user has hit send but no token has arrived yet. Show a typing indicator. The "streaming" state is when tokens are flowing. Distinguish them in CSS.

### 3. The composer — input, send, stop, retry

```tsx
// src/components/chat/composer.tsx
import { useRef } from "react";

type Props = {
  input: string;
  onChange: (e: React.ChangeEvent<HTMLTextAreaElement>) => void;
  onSubmit: (e: React.FormEvent) => void;
  status: string;
  onStop: () => void;
  onRetry: () => void;
};

export function Composer({ input, onChange, onSubmit, status, onStop, onRetry }: Props) {
  const busy = status === "submitted" || status === "streaming";
  return (
    <form onSubmit={onSubmit} className="border-t p-3 flex gap-2">
      <textarea
        value={input}
        onChange={onChange}
        onKeyDown={(e) => { if (e.key === "Enter" && !e.shiftKey) { e.preventDefault(); (e.currentTarget.form as HTMLFormElement).requestSubmit(); } }}
        placeholder="Ask anything…"
        rows={1}
        className="flex-1 resize-none rounded border p-2"
        disabled={busy}
      />
      {busy ? (
        <Button type="button" onClick={onStop} variant="destructive">Stop</Button>
      ) : messages.at(-1)?.role === "assistant" ? (
        <Button type="button" onClick={onRetry}>Retry</Button>
      ) : null}
      <Button type="submit" disabled={busy || !input.trim()}>Send</Button>
    </form>
  );
}
```

The three buttons — **Stop**, **Retry**, **Send** — are non-negotiable. A chat without a stop button is a chat where a runaway agent runs until it hits the token cap.

### 4. Document upload with progress

```tsx
// src/components/kb/dropzone.tsx
"use client";
import { useDropzone } from "react-dropzone";
import { uploadDocumentAction } from "@/app/actions/upload";

export function Dropzone({ workspaceId }: { workspaceId: string }) {
  const [progress, setProgress] = useState(0);
  const [uploading, setUploading] = useState(false);

  const { getRootProps, getInputProps } = useDropzone({
    accept: { "application/pdf": [".pdf"] },
    maxSize: 50 * 1024 * 1024, // 50 MB
    onDrop: async (files) => {
      for (const file of files) {
        setUploading(true);
        setProgress(0);
        const form = new FormData();
        form.set("file", file);
        form.set("workspaceId", workspaceId);
        const xhr = new XMLHttpRequest();
        xhr.upload.onprogress = (e) => e.lengthComputable && setProgress((e.loaded / e.total) * 100);
        xhr.onload = () => {
          setUploading(false);
          if (xhr.status >= 200 && xhr.status < 300) toast.success(`${file.name} uploaded`);
          else toast.error(`Upload failed: ${xhr.statusText}`);
        };
        xhr.open("POST", "/api/kb/upload");
        xhr.send(form);
      }
    },
  });

  return (
    <div {...getRootProps()} className="border-2 border-dashed rounded-lg p-8 text-center cursor-pointer">
      <input {...getInputProps()} />
      {uploading ? (
        <div>
          <div>Uploading… {progress.toFixed(0)}%</div>
          <Progress value={progress} />
        </div>
      ) : (
        <p>Drop PDFs here, or click to select</p>
      )}
    </div>
  );
}
```

Server action (Module 50/56's worker takes over from here):

```ts
// src/app/actions/upload.ts
"use server";
import { summarizeQueue } from "@/lib/queue/summarize";
import { saveToBlob } from "@/lib/storage";

export async function uploadDocumentAction(form: FormData) {
  const file = form.get("file") as File;
  const workspaceId = form.get("workspaceId") as string;
  if (!file || file.size === 0) throw new Error("empty_file");
  const path = await saveToBlob(file);
  const job = await summarizeQueue.add("summarize", { workspaceId, documentId: path, pdfPath: path, userId: "self" });
  return { jobId: job.id };
}
```

### 5. Citations as clickable footnotes

```tsx
// src/components/chat/citation-chip.tsx
"use client";
import { useState } from "react";

export function CitationChip({ source, index, onOpen }: { source: string; index: number; onOpen: (s: string) => void }) {
  return (
    <button
      onClick={() => onOpen(source)}
      className="ml-1 inline-flex h-5 min-w-5 items-center justify-center rounded bg-primary/10 text-xs text-primary hover:bg-primary/20"
    >
      {index}
    </button>
  );
}

export function CitationDrawer({ source, onClose }: { source: string | null; onClose: () => void }) {
  return (
    <AnimatePresence>
      {source && (
        <motion.aside
          className="fixed right-0 top-0 h-full w-96 bg-background shadow-xl"
          initial={{ x: "100%" }} animate={{ x: 0 }} exit={{ x: "100%" }}
        >
          <header className="flex items-center justify-between p-3 border-b">
            <div className="font-medium truncate">{source}</div>
            <Button variant="ghost" size="sm" onClick={onClose}>✕</Button>
          </header>
          <iframe src={`/api/kb/preview?source=${encodeURIComponent(source)}`} className="w-full h-[calc(100%-3rem)]" />
        </motion.aside>
      )}
    </AnimatePresence>
  );
}
```

Citations are *the* user-trust feature of a RAG product. A clickable chip that opens the actual source PDF page is the difference between "trustworthy AI" and "fancy autocomplete."

### 6. Loading, error, and retry states

Three states, three components:

```tsx
// src/components/states/loading.tsx
export function LoadingState({ hint }: { hint?: string }) {
  return (
    <div className="flex items-center gap-2 text-muted-foreground">
      <Spinner /> <span>{hint ?? "Loading…"}</span>
    </div>
  );
}

// src/components/states/error.tsx
export function ErrorState({ error, onRetry }: { error: Error; onRetry?: () => void }) {
  return (
    <div className="rounded border border-destructive/30 bg-destructive/5 p-3">
      <div className="font-medium">Something went wrong</div>
      <div className="text-sm text-muted-foreground">{error.message}</div>
      {onRetry && <Button size="sm" onClick={onRetry} className="mt-2">Retry</Button>}
    </div>
  );
}

// src/components/states/empty.tsx
export function EmptyState({ title, hint, action }: { title: string; hint?: string; action?: React.ReactNode }) {
  return (
    <div className="text-center p-8">
      <h3 className="text-lg font-medium">{title}</h3>
      {hint && <p className="text-sm text-muted-foreground mt-1">{hint}</p>}
      {action && <div className="mt-3">{action}</div>}
    </div>
  );
}
```

Use them everywhere. Every async resource has these three states; the user perceives "polished" or "broken" based on whether you handle them.

### 7. Optimistic UI

For non-streaming AI actions (e.g., "generate a project name"), show the result placeholder immediately and swap in the real answer when it arrives.

```tsx
// src/components/ai/optimistic-name.tsx
"use client";
import { useOptimistic } from "react";

type State = { name: string; pending: boolean };

export function ProjectNamer({ initial }: { initial: string }) {
  const [state, addOptimistic] = useOptimistic<State, string>(
    { name: initial, pending: false },
    (cur, _optimistic) => ({ name: "Generating…", pending: true })
  );

  async function regen() {
    addOptimistic("Generating…");
    const res = await fetch("/api/ai/project-name", { method: "POST" });
    const { name } = await res.json();
    // Server returns the new value; React reconciles.
  }

  return (
    <div className="flex items-center gap-2">
      <span className="font-medium">{state.name}</span>
      {state.pending && <Spinner size="sm" />}
      <Button size="sm" onClick={regen}>Regenerate</Button>
    </div>
  );
}
```

The `useOptimistic` hook (React 19) makes this declarative. The "Generating…" placeholder renders synchronously; the real value lands when the server replies.

## Common mistakes

1. **No stop button.** A user who can't cancel a runaway agent will close the tab. Provide a stop, always.
2. **Streaming without reconnection.** If the SSE connection drops, the message disappears. Buffer the streamed text in state and replay on reconnect.
3. **Uploads with no progress bar.** A 50 MB PDF that "loads" with no feedback feels frozen. Always show progress.
4. **Citations that don't link to anything.** A `[1]` chip with no preview is a lie. Make it open the source.
5. **Error messages that say "Something went wrong."** Tell the user *what* went wrong and *what to do* (retry, contact support, etc.).
6. **Optimistic UI without rollback.** If the server call fails, the placeholder must disappear and an error must show. `useOptimistic` handles this if you wire the error path.
7. **No keyboard shortcuts.** Power users want `Cmd+K` to focus the composer, `Cmd+/` to stop, `↑` for last message. Add them early.
8. **Forgetting accessibility.** Every chat input needs a label. Every citation chip needs `aria-label="Open source 1"`. Every spinner needs `role="status"`.

## Exercise

Build the user-facing AI surfaces for your SaaS.

**Acceptance criteria**

1. A chat panel at `/app/[workspace]/chat` that uses `useChat` and streams tokens. Stop, retry, and send buttons are present and work.
2. A knowledge base page at `/app/[workspace]/kb` with a drag-and-drop PDF uploader that shows upload progress and submits to a server action.
3. Citation chips in chat messages are clickable; clicking opens a side panel that previews the source PDF page.
4. Loading, error, and empty states are used on every async surface (chat, upload list, job list).
5. A "regenerate" action on the project name uses `useOptimistic` and rolls back on error.
6. The whole UI is keyboard-navigable; tabbing through messages and citations works; screen readers announce the chat state.

**Stretch**

- Add a `/app/[workspace]/jobs` page that shows a live-updating list of background jobs (Module 56's `QueueEvents`) with progress bars and cancel buttons.
- Add a `Cmd+K` command palette that opens an AI command bar.

## AI Industry Note (2026)

The frontend is where 2026 AI products differentiate. The model is the same for everyone; the UX is not.

- **Streaming is the default.** Not streaming is now a bug report. Linear's and Notion's AI features are the gold standard; users expect tokens to appear within 200ms of "Send."
- **The `ai` SDK by Vercel is the de-facto client.** It is what most Next.js apps use, and what most tutorials teach. The competitors (`assistant-ui`, `copilotkit`) have feature parity for 80% of cases.
- **Server Actions took off.** After a year of "fetch to /api/*", Next.js Server Actions are now the standard for non-streaming AI calls. They are simpler, type-safe, and integrate with `useOptimistic` for free.
- **Accessibility is finally getting attention.** EU AI Act and US state-level laws now require AI-generated content to be labeled. Screen-reader announcements of streaming tokens and citations are not nice-to-haves.
- **Citation UX is a competitive advantage.** A 2026 user who tries your product and the citations are *clickable* will not go back to the competitor whose citations are plain text. It is a small feature with a large trust impact.
- **Optimistic UI is the new dark mode.** The React 19 `useOptimistic` is the trick that makes AI features feel "fast" even when the server is slow. Use it.
- **Voice input is everywhere.** Whisper.cpp in the browser is good enough for "type by talking." A mic button next to the composer is now expected.