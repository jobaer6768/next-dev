# Module 50: Building a Production RAG System

## Why this module matters

A model that "knows" only what it learned at training time is a museum. A model that can pull fresh, attributable facts from *your* documents is a tool. Retrieval-Augmented Generation (RAG) is the bridge: it takes a user question, finds the right paragraphs in your corpus, and hands them to the LLM as evidence. Done well, RAG turns a generic chatbot into a domain expert. Done poorly, it confidently hallucinates citations to paragraphs that do not exist.

This is the module where you stop demoing and start shipping. By the end you will have built a real PDF knowledge-base assistant for the SaaS from Mission 4, with citations, refusals, evaluation, and cost controls. Everything in Modules 51–58 will assume this pipeline exists.

## Learning objectives

By the end of this module you will be able to:

1. Describe every stage of a RAG pipeline and explain what can go wrong at each one.
2. Pick a chunking strategy for a given document type and justify it.
3. Choose between sparse, dense, and hybrid retrieval, and know when to add a re-ranker.
4. Build a prompt that cites its sources and refuses cleanly when the evidence is missing.
5. Evaluate a RAG system with RAGAS (faithfulness, answer relevance, context recall, context precision).
6. Optimize cost and latency with embedding caches, summarization, and routing models.
7. Wire a PDF upload into a Next.js + pgvector backend with a streaming chat endpoint.

## Conceptual model

A RAG system is a seven-stage pipeline. Memorize the order; it dictates the failure modes.

```
load  →  split  →  embed  →  store  →  retrieve  →  prompt  →  generate
```

Each stage is a function. Each function is something you can swap, cache, and evaluate in isolation. That is the engineering superpower RAG gives you over a fine-tune.

### 1. Load

```ts
// src/lib/rag/loaders.ts
import { PDFLoader } from "@langchain/community/document_loaders/fs/pdf";
import { DocxLoader } from "@langchain/community/document_loaders/fs/docx";
import { RecursiveCharacterTextSplitter } from "langchain/text_splitter";
import { Document } from "@langchain/core/documents";

export async function loadAndSplit(filePath: string, ext: string): Promise<Document[]> {
  const loader = ext === ".pdf" ? new PDFLoader(filePath) : new DocxLoader(filePath);
  const raw = await loader.load();

  // PDFs are noisy: 1500-char chunks with 200 overlap is a sane default
  const splitter = new RecursiveCharacterTextSplitter({
    chunkSize: 1500,
    chunkOverlap: 200,
    separators: ["\n\n", "\n", ". ", " ", ""],
  });

  const chunks = await splitter.splitDocuments(raw);

  // Every chunk must carry provenance. The downstream prompt relies on it.
  return chunks.map((d, i) => {
    d.metadata = {
      ...d.metadata,
      source: filePath,
      chunkIndex: i,
      uploadedAt: new Date().toISOString(),
    };
    return d;
  });
}
```

The mistake I see every cohort: stripping metadata. Without `source` and `chunkIndex`, your citations are fiction and your debugging is impossible.

### 2. Embed and store (pgvector)

```ts
// src/lib/rag/embeddings.ts
import { OpenAIEmbeddings } from "@langchain/openai";
import { PGVectorStore } from "@langchain/community/vectorstores/pgvector";
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

export const embeddings = new OpenAIEmbeddings({
  model: "text-embedding-3-small", // 1536 dims, $0.02 / 1M tokens
  caching: true,                    // Built-in LRU cache
  maxRetries: 3,
});

export const vectorStore = new PGVectorStore(embeddings, {
  pool,
  tableName: "kb_chunks",
  columns: {
    idColumnName: "id",
    contentColumnName: "content",
    metadataColumnName: "metadata",
    vectorColumnName: "embedding",
  },
});

await vectorStore.addDocuments(chunks);
```

The schema is one SQL migration:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE TABLE kb_chunks (
  id        BIGSERIAL PRIMARY KEY,
  content   TEXT NOT NULL,
  metadata  JSONB NOT NULL,
  embedding vector(1536) NOT NULL,
  workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX kb_chunks_vec_idx ON kb_chunks USING hnsw (embedding vector_cosine_ops);
CREATE INDEX kb_chunks_workspace_idx ON kb_chunks (workspace_id);
```

**Always scope by `workspace_id`.** A RAG system that leaks customer data into another tenant's answers is a lawsuit.

### 3. Retrieve (top-k, then re-rank)

```ts
// src/lib/rag/retrieve.ts
import { vectorStore } from "./embeddings";

export type RetrievedChunk = {
  content: string;
  source: string;
  chunkIndex: number;
  score: number;
};

export async function retrieve(
  query: string,
  workspaceId: string,
  k = 20
): Promise<RetrievedChunk[]> {
  // Stage 1: cheap dense recall — get 20 candidates
  const candidates = await vectorStore.similaritySearchWithScore(query, k, {
    filter: { workspace_id: workspaceId },
  });

  // Stage 2 (optional): re-rank with a cross-encoder
  // Cohere Rerank v3 lifts answer relevance by ~15% on most benchmarks
  if (process.env.COHERE_API_KEY) {
    const { CohereRerank } = await import("@langchain/cohere");
    const reranker = new CohereRerank({ model: "rerank-english-v3.0", topN: 5 });
    const docs = candidates.map(([d]) => d);
    const reranked = await reranker.compressDocuments(docs, query);
    return reranked.map((d) => ({
      content: d.pageContent,
      source: d.metadata.source as string,
      chunkIndex: d.metadata.chunkIndex as number,
      score: d.metadata.relevanceScore as number,
    }));
  }

  return candidates.map(([d, score]) => ({
    content: d.pageContent,
    source: d.metadata.source as string,
    chunkIndex: d.metadata.chunkIndex as number,
    score,
  }));
}
```

Rule of thumb: top-k=20 → rerank → top-n=5. The retriever finds plausible paragraphs; the re-ranker finds the *right* ones. They solve different problems.

### 4. Prompt assembly with citations

```ts
// src/lib/rag/prompt.ts
import { ChatPromptTemplate } from "@langchain/core/prompts";

export const ragPrompt = ChatPromptTemplate.fromMessages([
  ["system", `You are a support assistant for the user's workspace.
Answer the question using ONLY the numbered sources below.
If the answer is not in the sources, reply: "I don't have that information."
After every claim, cite the source number in brackets, e.g. [1][3].
Never invent a citation.

SOURCES:
{context}`],
  ["human", "{question}"],
]);

export function buildContext(chunks: RetrievedChunk[]): string {
  return chunks
    .map((c, i) => `[${i + 1}] (${c.source}#${c.chunkIndex})\n${c.content}`)
    .join("\n\n---\n\n");
}
```

The "I don't have that information" refusal is the single most important string in the file. It is what separates a RAG system from a hallucination machine.

### 5. Generate, streaming

```ts
// src/app/api/chat/route.ts
import { ChatOpenAI } from "@langchain/openai";
import { StringOutputParser } from "@langchain/core/output_parsers";
import { ragPrompt, buildContext } from "@/lib/rag/prompt";
import { retrieve } from "@/lib/rag/retrieve";

export async function POST(req: Request) {
  const { question, workspaceId } = await req.json();
  const chunks = await retrieve(question, workspaceId);
  const context = buildContext(chunks);

  const model = new ChatOpenAI({
    model: "gpt-4o-mini",
    temperature: 0,
    streaming: true,
  });

  const stream = await ragPrompt
    .pipe(model)
    .pipe(new StringOutputParser())
    .stream({ question, context });

  return new Response(stream, { headers: { "content-type": "text/event-stream" } });
}
```

## Common mistakes

1. **Chunking by fixed character count, ignoring structure.** A 1500-char chunk that lands mid-sentence destroys retrieval. Use `RecursiveCharacterTextSplitter` and put `separators` ahead of `\n`.
2. **Skipping the re-ranker.** Bi-encoders (the thing that makes embeddings) are fast but lossy. A cross-encoder reads query + document together and is much more accurate. It is also slow — that is why you only re-rank the top-k.
3. **No tenant scoping.** Every retrieval query must filter by `workspace_id`. Test this with a cross-tenant test suite, not vibes.
4. **Refusing too eagerly.** "I don't have that information" should fire when retrieval confidence is low, not when retrieval returned zero (which can mean the embedding model is bad, not that the answer is missing).
5. **Forgetting to evaluate.** A RAG system that *feels* good is not a good RAG system. Wire RAGAS into your CI.
6. **Re-embedding on every upload without a cache.** Embedding a 200-page PDF twice is a 2¢ mistake. Multiply by 10k docs and it is a small car.

## Exercise

Build a PDF knowledge-base assistant for your Mission 4 SaaS.

**Acceptance criteria**

1. A user in a workspace can upload a PDF, see it chunked and embedded, and chat with it.
2. Each assistant answer cites at least one source by file name and chunk index.
3. If no chunk scores above 0.65 (cosine), the system refuses with the exact phrase "I don't have that information in the provided documents."
4. A RAGAS evaluation script reports faithfulness, answer-relevance, and context-recall on a 20-question test set you author.
5. Embedding calls are cached (the second upload of the same file should not re-embed).
6. The chat endpoint streams; the UI shows tokens as they arrive.

**Stretch**

- Add a hybrid retriever (BM25 + dense) and a Cohere re-ranker; compare RAGAS scores before/after.
- Add a `/api/sources/[id]` endpoint that returns the original PDF page as an image so the user can verify the citation.

## AI Industry Note (2026)

In 2026, "naive RAG" is table stakes. The frontier has moved to:

- **Late chunking** (Jina, 2024): embed the *full document* first, then take pooled embeddings of the chunks. Improves retrieval on long, context-heavy docs by 8–14% on most benchmarks.
- **Agentic RAG** (Module 51 territory): the LLM decides *when* to retrieve and *which* queries to issue, instead of a single top-k call. A multi-hop question like "Compare our Q1 and Q2 refund policy" needs at least two retrievals.
- **RAG evaluation as code.** RAGAS, DeepEval, and Phoenix are CI gates. If you cannot tell your PM that you moved answer-relevance from 0.71 to 0.83, you do not have a RAG system; you have a vibe.
- **Cost has collapsed but not vanished.** `text-embedding-3-small` at $0.02/1M tokens and `gpt-4o-mini` at $0.15/1M mean a 1M-question/month RAG product is comfortably under $500/mo. The new cost worry is *retrieval compute at scale* (HNSW indexes get expensive above ~10M vectors), not LLM tokens.
- **pgvector vs. dedicated vector DBs.** For under 10M vectors and under 1k QPS, pgvector + HNSW wins on operational simplicity. Above that, Qdrant or Turbopuffer start to pay for themselves.

**Key takeaway for 2026:** A production RAG system is a *measured* system. If you are not running an eval suite on every prompt and embedding-model change, you are shipping vibes.
