# Module 48: Document Processing & Embeddings

## Why this module matters

An LLM that has not read your documents is a tourist. An LLM that *has* read your documents — and can find the right paragraph in 50ms out of 50,000 of them — is a teammate. That capability is the entire point of RAG, and it has two halves: turning documents into **embeddings** (vectors of numbers that capture meaning) and **searching** those embeddings fast (a vector store). This module teaches both halves, end to end, on the SaaS from Mission 4.

You will also learn the part no one warns you about: chunking is the single biggest determinant of RAG quality, and the default settings will silently wreck your results. A 4,000-token chunk with no overlap will glue half-sentences together and your retrieval will hallucinate. A 200-token chunk with no semantic awareness will split a definition from the term it defines and your retrieval will miss. This module teaches you to think about chunks the way a database engineer thinks about indexes: deliberately, with measurements, and with the right tool for the workload.

## Learning objectives

By the end of this module you will be able to:

1. Load documents from PDF, web, CSV, and plain text into a uniform `Document` type.
2. Explain the tradeoffs between recursive, token-based, semantic, and sentence-based chunking.
3. Describe what an embedding is, what its dimensionality means, and why cosine similarity is the standard distance.
4. Pick an embedding model for a given cost / quality / latency budget.
5. Compare the five major vector stores (Pinecone, Weaviate, Qdrant, Chroma, pgvector) on cost, scale, and ops burden.
6. Build a retrievers pipeline with similarity, MMR, and hybrid search.
7. Wire a full `load → split → embed → store → query` pipeline for the Mission 4 SaaS.

## Conceptual model

The pipeline is the same shape as a database ETL. The units change; the discipline does not.

```
load  →  split  →  embed  →  store  →  retrieve  →  use
```

### 1. Loading: one interface, many formats

LangChain's `DocumentLoader` abstraction gives every format the same shape: `load()` returns `Document[]` where each `Document` has `pageContent: string` and `metadata: Record<string, any>`. The metadata is your audit trail — keep it rich.

```ts
// src/lib/rag/loaders.ts
import { PDFLoader } from "@langchain/community/document_loaders/fs/pdf";
import { DocxLoader } from "@langchain/community/document_loaders/fs/docx";
import { CSVLoader } from "@langchain/community/document_loaders/fs/csv";
import { CheerioWebBaseLoader } from "@langchain/community/document_loaders/web/cheerio";
import { Document } from "@langchain/core/documents";
import { readFile } from "node:fs/promises";

export type LoadedDoc = {
  source: string;
  docs: Document[];
};

export async function loadFromFile(path: string, mime: string): Promise<LoadedDoc> {
  let docs: Document[];
  if (mime === "application/pdf") {
    docs = await new PDFLoader(path, { splitPages: true }).load();
  } else if (mime === "application/vnd.openxmlformats-officedocument.wordprocessingml.document") {
    docs = await new DocxLoader(path).load();
  } else if (mime === "text/csv") {
    docs = await new CSVLoader(path).load();          // each row → one Document
  } else if (mime === "text/plain") {
    const raw = await readFile(path, "utf8");
    docs = [new Document({ pageContent: raw, metadata: { source: path } })];
  } else {
    throw new Error(`unsupported mime: ${mime}`);
  }

  // Stamp every chunk with provenance and tenant info up front. You cannot
  // add it later without re-embedding — pay the cost now.
  for (const d of docs) {
    d.metadata = {
      ...d.metadata,
      source: path,
      loadedAt: new Date().toISOString(),
    };
  }
  return { source: path, docs };
}

export async function loadFromUrl(url: string): Promise<LoadedDoc> {
  const loader = new CheerioWebBaseLoader(url, {
    selector: "main, article, .content",     // skip nav/footer/ads
  });
  const docs = await loader.load();
  for (const d of docs) {
    d.metadata = { ...d.metadata, source: url, loadedAt: new Date().toISOString() };
  }
  return { source: url, docs };
}
```

The provenance is non-negotiable. When the model cites its sources, the citation comes from `metadata.source`. When you debug "why is this chunk in the wrong workspace," the answer is in `metadata.workspaceId`. When the legal team asks "what user data was used to answer this," the audit log has the row.

### 2. Splitting: the most important decision you will make

Chunking is a tradeoff. Bigger chunks give the model more context per retrieval but reduce the precision of the match (a 4,000-token chunk about invoices will match "billing question" queries that want a specific clause). Smaller chunks are precise but lose the surrounding context. The art is finding a size that keeps semantic units together.

```ts
// src/lib/rag/splitters.ts
import { RecursiveCharacterTextSplitter } from "@langchain/textsplitters";
import { TokenTextSplitter } from "@langchain/textsplitters";
import { Document } from "@langchain/core/documents";

// The workhorse. Splits on a hierarchy of separators, preferring
// the most "natural" boundary (paragraph > line > sentence > word > char).
export const defaultSplitter = new RecursiveCharacterTextSplitter({
  chunkSize: 1_000,             // chars
  chunkOverlap: 200,            // 20% overlap keeps sentences from being cut in half
  separators: ["\n\n", "\n", ". ", "? ", "! ", " ", ""],
});

// Use when you care about token budget, not character budget.
// Especially important for embedding models with hard token limits.
export const tokenSplitter = new TokenTextSplitter({
  encodingName: "cl100k_base",  // GPT-4 / GPT-4.1 tokenizer
  chunkSize: 256,
  chunkOverlap: 32,
});

// Domain-specific splitter for code: split on function/class boundaries
// instead of character counts. (You will write your own.)
export class CodeSplitter {
  async splitDocuments(docs: Document[]): Promise<Document[]> { /* ... */ }
}
```

Why the recursive splitter? It tries to keep paragraphs together, falls back to sentences, then to words, then to characters. The result is that 90% of your chunks are "whole thought" units, which is exactly what you want for retrieval.

A common gotcha: **chunk size should match embedding model limits, not just LLM context windows.** `text-embedding-3-small` accepts up to 8,191 tokens per call. If your chunk is 8,000 tokens, you have 191 tokens of safety margin and one weird Unicode character will fail. Stay under 2,000 tokens per chunk and you never have to think about it.

A second gotcha: **overlap is insurance, not a bug.** Without overlap, a sentence that straddles two chunks will be split in half and neither half will match a query that contains the full sentence. 10–20% overlap is the standard.

A third gotcha: **chunk size depends on the query.** For a "find me the section on error handling" query, you want bigger chunks. For a "find me the exact function signature" query, you want smaller chunks. The principled answer is: pick the chunk size that maximizes your eval score, and measure it. Module 50 teaches the eval.

### 3. Embeddings: vectors of numbers that capture meaning

An embedding is a list of numbers — usually 384, 768, 1024, 1536, or 3072 of them — that represents a piece of text in a continuous space where **distance corresponds to meaning**. Two sentences that mean similar things have vectors that are close together (small angle, small distance). Two unrelated sentences are far apart.

The key intuitions:

- **You do not get to choose what each dimension means.** The model learned them. Dimension 247 might be "is this about sports" and dimension 503 might be "is the sentiment positive." You cannot peek, but the geometry is meaningful.
- **Cosine similarity is the standard distance metric.** Two unit vectors that point the same way → cosine = 1. Perpendicular → 0. Opposite → −1. Euclidean distance (L2) is also fine; inner product only if vectors are normalized.
- **The same embedding model must be used at write time and read time.** Mixing `text-embedding-3-small` (1536d) and `voyage-3` (1024d) will silently produce garbage. The geometry is not comparable across models.

```ts
// src/lib/rag/embeddings.ts
import { OpenAIEmbeddings } from "@langchain/openai";
import { VoyageEmbeddings } from "@langchain/community/embeddings/voyage";
import { Embeddings } from "@langchain/core/embeddings";

const cache = new Map<string, number[]>();

export const embeddings: Embeddings = new OpenAIEmbeddings({
  model: "text-embedding-3-small",      // 1536d, $0.02 / 1M tokens, MTEB ~62
});

// Local fallback for dev — zero cost, lower quality, 384d.
export const localEmbeddings: Embeddings = (await import(
  "@langchain/community/embeddings/transformers"
)).TransformersEmbeddings;

export async function embedCached(text: string): Promise<number[]> {
  const key = sha256(text);
  const hit = cache.get(key);
  if (hit) return hit;
  const v = await embeddings.embedQuery(text);
  cache.set(key, v);
  return v;
}
```

Embedding model comparison for late 2025:

| Model                       | Dim | $/1M tokens | MTEB score | Notes                            |
| --------------------------- | --- | ----------- | ---------- | -------------------------------- |
| OpenAI `text-embedding-3-small` | 1536 | 0.02 | 62.3 | Default; good price/quality       |
| OpenAI `text-embedding-3-large` | 3072 | 0.13 | 64.6 | Diminishing returns vs small      |
| Voyage `voyage-3`           | 1024 | 0.06 | 67.1 | Best-in-class retrieval           |
| Cohere `embed-english-v3.0` | 1024 | 0.10 | 64.5 | Good, input-type aware            |
| BGE-large-en-v1.5 (local)   | 1024 | 0     | 63.5 | Free, slower, infra you operate   |
| `all-MiniLM-L6-v2` (local)  | 384  | 0     | 58.0 | Smallest, fastest, lowest quality |

`MTEB` is the Massive Text Embedding Benchmark. The scores are not directly comparable across years, but the ordering is stable: Voyage and OpenAI are the two serious paid options; BGE is the serious open-source option.

### 4. Vector stores: the database for embeddings

A vector store is a database that indexes high-dimensional vectors and answers nearest-neighbor queries in milliseconds. There are five you will encounter in 2026. The differences matter less than the marketing suggests, and matter more than the marketing suggests, simultaneously.

| Store         | Hosted         | Self-host | Open source | Bill                  | Sweet spot                          |
| ------------- | -------------- | --------- | ----------- | --------------------- | ----------------------------------- |
| **Pinecone**  | Yes            | No        | No          | Per-pod, per-query    | Massive scale, zero ops             |
| **Weaviate**  | Yes (WCS)      | Yes       | Yes (BSD)   | Per-pod               | Hybrid + filters + GraphQL          |
| **Qdrant**    | Yes (Cloud)    | Yes       | Yes (Apache)| Per-pod               | Performance, payload filtering      |
| **Chroma**    | Yes (free tier)| Yes       | Yes (Apache)| Free / small          | Dev, prototypes, single-machine     |
| **pgvector**  | (your Postgres)| Yes       | Yes (MIT)   | Same as your Postgres | Already on Postgres, < 10M vectors  |

For the bootcamp SaaS, **pgvector wins** because you already run Postgres. Module 49 goes deep on it. For a Series B with 100M+ vectors, Pinecone or Qdrant is the right call. Decide based on `count(*)` and ops budget, not vibes.

```ts
// src/lib/rag/vectorstore.ts
import { MemoryVectorStore } from "langchain/vectorstores/memory";
import { PGVectorStore } from "@langchain/community/vectorstores/pgvector";
import { OpenAIEmbeddings } from "@langchain/openai";
import { Pool } from "pg";

// Dev: in-memory, zero setup, lost on restart.
export async function makeMemoryStore() {
  return new MemoryVectorStore(new OpenAIEmbeddings());
}

// Prod: pgvector, real persistence, integrates with the rest of the SaaS.
export async function makePgStore() {
  const pool = new Pool({ connectionString: process.env.DATABASE_URL });
  return await PGVectorStore.initialize(new OpenAIEmbeddings(), {
    pool,
    tableName: "documents",
    columns: {
      idColumnName: "id",
      contentColumnName: "content",
      metadataColumnName: "metadata",
      embeddingColumnName: "embedding",
    },
  });
}
```

### 5. Retrievers: the read path

A retriever is the read side of a vector store. Given a query string, return the top-k most relevant chunks. Three modes you will use:

```ts
// src/lib/rag/retrievers.ts
import { PGVectorStore } from "@langchain/community/vectorstores/pgvector";
import { OpenAIEmbeddings } from "@langchain/openai";

const store = await makePgStore();

// 1. Similarity search — the default. Top-k by cosine distance.
const top = await store.similaritySearch("how do refunds work?", 5);

// 2. MMR (Maximal Marginal Relevance) — trade relevance for diversity.
const diverse = await store.maxMarginalRelevanceSearch(
  "how do refunds work?",
  { k: 5, fetchK: 20, lambda: 0.5 },
);
// lambda=0   → pure diversity (no relevance penalty)
// lambda=1   → pure relevance (same as similarity search)
// lambda=0.5 → balanced (the default)

// 3. With metadata filter — the multi-tenant path. CRITICAL for SaaS.
const tenantTop = await store.similaritySearch(
  "how do refunds work?",
  5,
  { workspaceId: "ws_abc" },     // pgvector filter; pre-filter on the index
);
```

MMR is the trick that fixes "I asked about pricing and got five near-duplicate chunks from the pricing page." The `lambda` knob controls how much you penalize redundancy.

### 6. The full pipeline

```ts
// src/lib/rag/pipeline.ts
import { RecursiveCharacterTextSplitter } from "@langchain/textsplitters";
import { Document } from "@langchain/core/documents";
import { loadFromFile } from "./loaders";
import { makePgStore } from "./vectorstore";

const splitter = new RecursiveCharacterTextSplitter({
  chunkSize: 1_000, chunkOverlap: 200,
});

export async function ingest(args: {
  workspaceId: string;
  filePath: string;
  mime: string;
  documentId: string;
}) {
  const { docs } = await loadFromFile(args.filePath, args.mime);
  const chunks = await splitter.splitDocuments(docs);

  // Stamp every chunk with tenant info + provenance. This is how the
  // retriever's metadata filter stays correct.
  for (const c of chunks) {
    c.metadata = {
      ...c.metadata,
      workspaceId: args.workspaceId,
      documentId:  args.documentId,
    };
  }

  const store = await makePgStore();
  await store.addDocuments(chunks);

  return { count: chunks.length };
}

export async function query(args: {
  workspaceId: string;
  question: string;
  k?: number;
}) {
  const store = await makePgStore();
  return store.similaritySearch(
    args.question,
    args.k ?? 5,
    { workspaceId: args.workspaceId },
  );
}
```

That is the entire RAG engine in 80 lines. Module 50 will show you how to wire the result into a prompt and stream an answer. Module 49 will show you how to make pgvector fast.

## Common mistakes

- **Embedding with one model and querying with another.** The vectors live in incompatible spaces. The retriever returns noise and you will spend two days debugging before you notice.
- **Stripping metadata before splitting.** Once you split, you cannot reliably recover the source. Stamp provenance on the `Document` *before* you split.
- **Forgetting to scope the retriever by `workspaceId`.** This is the cross-tenant data leak. Test for it explicitly (Module 50 has the test).
- **Chunking the whole document into one piece.** Long context ≠ good context. The retriever cannot pick a paragraph out of a 30,000-token blob; it can pick it out of thirty 1,000-token chunks.
- **Chunking too small.** 200-character chunks have no semantic content. The embedding is noise. You will retrieve, but you will retrieve badly.
- **Trusting the default chunk size.** 1,000 / 200 is a fine default for English prose. It is wrong for code, wrong for tables, wrong for short FAQs. Measure.
- **Not caching embeddings.** Re-embedding the same chunk because a deploy restarted is a 10% line item in your OpenAI bill. An LRU on the chunk hash pays for itself in an afternoon.
- **Using Euclidean distance for non-normalized vectors.** Always normalize, or always use cosine. Mixing the two makes results harder to reason about.

## Exercise

Build a `POST /api/documents` and `POST /api/query` pair on the Mission 4 SaaS. Acceptance criteria:

1. `POST /api/documents` accepts a multipart upload (`pdf`, `docx`, `txt`, `csv`), runs `ingest()` from this module, and returns `{ documentId, chunkCount }`.
2. Chunks are stamped with `workspaceId` from the JWT and `documentId` from the upload.
3. Uses `RecursiveCharacterTextSplitter` with chunk size 1,000 and overlap 200.
4. `POST /api/query` accepts `{ question: string, k?: number }` and returns the top-k chunks from `query()` filtered by `workspaceId` from the JWT.
5. Includes an integration test: upload a known PDF, run a query for a phrase that appears on page 4, and assert that the top-1 result's metadata `page` is in `[3, 4, 5]`.
6. Includes a multi-tenant test: upload a doc to workspace A, upload a different doc to workspace B, query A, and assert that no chunks from B are returned.

Stretch: swap the recursive splitter for a semantic splitter (e.g., `SemanticChunker` from LangChain experimental) and compare retrieval recall on a 5-query eval set.

## AI Industry Note (2026)

Embeddings and vector search in 2026 are a solved primitive and an unsolved product. Three things to internalize:

1. **The "vector database" is mostly a commodity.** The five major stores all support the same primitive (top-k ANN with metadata filters), and pgvector has closed most of the performance gap. The decision is now driven by ops burden and ecosystem, not raw recall. Most startups in 2026 default to pgvector and only migrate when they hit 10M+ vectors.
2. **Chunking is the new feature engineering.** The single biggest quality bump in a RAG system in 2026 came not from a better embedding model but from a better chunking strategy: semantic boundaries, late chunking (embed the whole doc, then pool), or LLM-guided chunking. The default character splitter is the new "default 32 batches in SGD."
3. **Hybrid is the default.** Pure semantic search loses on exact-match queries (product names, SKUs, error codes). Pure keyword search loses on paraphrased questions. Every serious RAG system in 2026 runs both in parallel and fuses the results with reciprocal rank fusion. Module 49 will show you the pgvector implementation.
