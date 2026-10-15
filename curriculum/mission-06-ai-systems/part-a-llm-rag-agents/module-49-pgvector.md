# Module 49: Vector Databases: pgvector

## Why this module matters

Module 48 introduced vector stores in the abstract. This module is the concrete one: the database you will actually run, in the Postgres you already run, on the SaaS you already ship. pgvector is a Postgres extension that turns your existing tables into ANN-searchable vector stores, and in 2026 it is the default choice for any team that has not yet hit 10M+ vectors per table. The reasons are the boring ones: one less service to operate, one less credential to manage, one less place for tenant data to leak, and one transactional story for both the row and its embedding.

This module teaches you what pgvector actually does under the hood (HNSW, IVF, exact search), how to choose between them, and how to wire a multi-tenant `search_documents(workspaceId, query)` function that the rest of the mission can call. By the end you will have a real `documents` table, a real index, a real hybrid search, and the test that proves it does not bleed data across workspaces.

## Learning objectives

By the end of this module you will be able to:

1. Explain the three ANN algorithms pgvector supports (HNSW, IVF, exact) and when each is right.
2. Install pgvector, create an extension, and design a `documents` table with the right columns.
3. Choose and create the correct index for a given workload (`hnsw`, `ivfflat`, or none).
4. Write cosine, L2, and inner-product distance queries and explain when to use which.
5. Build a hybrid search (semantic + keyword) using `<=>` + `tsvector` and reciprocal rank fusion.
6. Implement metadata filtering that is safe under multi-tenancy.
7. Decide when pgvector is the right answer and when you should reach for a dedicated store.

## Conceptual model

### What pgvector actually is

pgvector is a Postgres extension that adds:

1. A new column type, `vector(N)`, that stores a fixed-dimension array of floats.
2. Three distance operators: `<->` (L2 / Euclidean), `<=>` (cosine), and `<#>` (inner product).
3. Two index types: `hnsw` and `ivfflat`, each of which makes the operators fast.
4. Aggregate functions for averaging vectors, useful for some hybrid search tricks.

It is a small, focused extension. It does not add a new server process, a new wire protocol, or a new backup story. It is your Postgres, with a couple of extra operators.

### The three distance operators

```sql
-- Cosine distance: 1 - cos(angle). The most common choice for normalized embeddings.
SELECT embedding <=> '[0.1, 0.2, ...]'::vector AS cosine_distance
FROM documents
ORDER BY cosine_distance ASC
LIMIT 5;

-- L2 (Euclidean) distance: the literal geometric distance.
SELECT embedding <-> '[0.1, 0.2, ...]'::vector AS l2_distance
FROM documents
ORDER BY l2_distance ASC
LIMIT 5;

-- Inner product (negative, because pgvector orders ascending by default).
-- Useful for non-normalized vectors where magnitude matters.
SELECT (embedding <#> '[0.1, 0.2, ...]'::vector) * -1 AS inner_product
FROM documents
ORDER BY inner_product DESC
LIMIT 5;
```

Rule of thumb: if you normalize your vectors (and you should), cosine and inner product are equivalent up to a sign, and both are the right choice. If you do not normalize, L2 is the safer default. `text-embedding-3-small` returns normalized vectors; check your model's docs.

### The three index types

This is the part that determines whether your query takes 5ms or 5 seconds. Choose wrong and your table will fall over at 1M rows. Choose right and 10M rows is a normal Tuesday.

```sql
-- 1. No index — exact, slow, but always correct. Good up to ~50k rows.
SELECT id FROM documents ORDER BY embedding <=> $1 LIMIT 5;

-- 2. IVFFlat — partitions vectors into clusters, searches the closest clusters.
-- Trains on existing data; re-train when the data drifts.
CREATE INDEX documents_embedding_ivf
  ON documents USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);     -- ~sqrt(rows) is a good default

-- 3. HNSW — a graph where every node links to its approximate neighbors.
-- Faster, more accurate, but more memory. The 2026 default.
CREATE INDEX documents_embedding_hnsw
  ON documents USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);
```

The choice matrix:

| Index          | Build time  | Query speed  | Recall    | Memory   | Use when                                   |
| -------------- | ----------- | ------------ | --------- | -------- | ------------------------------------------ |
| None (exact)   | 0           | Slow         | 100%      | 0        | < 50k rows, dev, "I need a correct answer" |
| IVFFlat        | Medium      | Medium       | 90–95%    | Low      | Large table, lower memory budget, batch     |
| HNSW           | Slow        | Fast         | 95–99%    | High     | Production RAG, the 2026 default            |

HNSW has two tuning knobs:

- `m` — the number of bi-directional links per node. Higher = better recall, more memory. Default 16; 32 for high-recall workloads.
- `ef_construction` — the size of the dynamic candidate list during build. Higher = better index, slower build. Default 64; 128 for one-shot builds.

And one runtime knob:

```sql
-- Per-query: trade speed for recall. Default is 40.
SET hnsw.ef_search = 100;   -- higher → better recall, slower query
```

If your recall eval is off, bump `ef_search` before you rebuild the index. It is the cheapest knob.

### The schema

```sql
-- migrations/0015_documents.sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE documents (
  id           uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id uuid        NOT NULL,
  uploaded_by  uuid        NOT NULL,
  source       text        NOT NULL,                 -- file path or URL
  mime         text        NOT NULL,
  title        text,
  content      text        NOT NULL,                 -- raw text
  embedding    vector(1536) NOT NULL,                -- text-embedding-3-small
  metadata     jsonb       NOT NULL DEFAULT '{}'::jsonb,
  created_at   timestamptz NOT NULL DEFAULT now()
);

-- Multi-tenancy first. Every query MUST go through this index.
CREATE INDEX documents_workspace_idx
  ON documents (workspace_id, created_at DESC);

-- Vector index. HNSW with cosine, since we normalize.
CREATE INDEX documents_embedding_hnsw_idx
  ON documents USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);

-- Optional: full-text search column for hybrid retrieval.
ALTER TABLE documents
  ADD COLUMN content_tsv tsvector
  GENERATED ALWAYS AS (to_tsvector('english', content)) STORED;
CREATE INDEX documents_content_tsv_idx
  ON documents USING gin (content_tsv);
```

Three indexes, three purposes:

1. `documents_workspace_idx` — filters by tenant. Every query must include `workspace_id = $1`; this index makes that filter O(log n) instead of O(n).
2. `documents_embedding_hnsw_idx` — speeds up the `<=>` operator inside a tenant.
3. `documents_content_tsv_idx` — speeds up keyword search for hybrid retrieval.

The `GENERATED ALWAYS AS ... STORED` column is the 2026 way to maintain a `tsvector` from `content`: Postgres computes it on insert/update, no triggers, no application code, no drift.

### The function

Wrap the query in a SQL function so the call site is one line and the test surface is one function. Always return the full row (or at least `id`, `content`, `metadata`, and the distance) so the caller can cite sources.

```sql
-- migrations/0016_search_documents.sql
CREATE OR REPLACE FUNCTION search_documents(
  p_workspace_id uuid,
  p_query_embedding vector(1536),
  p_query_text     text,
  p_match_count   int DEFAULT 5,
  p_filter        jsonb DEFAULT '{}'::jsonb
)
RETURNS TABLE (
  id           uuid,
  content      text,
  metadata     jsonb,
  source       text,
  semantic_rank int,
  keyword_rank  int,
  combined_score float
)
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  WITH sem AS (
    SELECT d.id, row_number() OVER (ORDER BY d.embedding <=> p_query_embedding) AS rk
    FROM documents d
    WHERE d.workspace_id = p_workspace_id
      AND (p_filter = '{}'::jsonb OR d.metadata @> p_filter)
    ORDER BY d.embedding <=> p_query_embedding
    LIMIT p_match_count * 4                -- over-fetch for fusion
  ),
  kw AS (
    SELECT d.id, row_number() OVER (
      ORDER BY ts_rank_cd(d.content_tsv, plainto_tsquery('english', p_query_text)) DESC
    ) AS rk
    FROM documents d
    WHERE d.workspace_id = p_workspace_id
      AND d.content_tsv @@ plainto_tsquery('english', p_query_text)
      AND (p_filter = '{}'::jsonb OR d.metadata @> p_filter)
    LIMIT p_match_count * 4
  ),
  fused AS (
    SELECT
      COALESCE(s.id, k.id)        AS id,
      s.rk                        AS semantic_rank,
      k.rk                        AS keyword_rank,
      (
        COALESCE(1.0 / (60 + s.rk), 0) +
        COALESCE(1.0 / (60 + k.rk), 0)
      )::float                    AS combined_score
    FROM sem s
    FULL OUTER JOIN kw k ON s.id = k.id
  )
  SELECT
    d.id, d.content, d.metadata, d.source,
    f.semantic_rank::int, f.keyword_rank::int, f.combined_score
  FROM fused f
  JOIN documents d ON d.id = f.id
  ORDER BY f.combined_score DESC
  LIMIT p_match_count;
END;
$$;
```

The fusion formula `1 / (60 + rank)` is reciprocal rank fusion (RRF) with the standard `k=60`. It is the boring, well-studied, works-pretty-well choice. If you have an eval set, you can tune `k`; do not tune it without one.

The `p_filter` parameter uses the `jsonb @> jsonb` operator for "contains" — a clean way to pass `{ "documentType": "policy" }` and have it match every document whose metadata contains that key/value pair.

### Calling it from the SaaS

```ts
// src/lib/rag/search.ts
import { sql } from "drizzle-orm";
import { db } from "@/db";
import { OpenAIEmbeddings } from "@langchain/openai";

const embeddings = new OpenAIEmbeddings({ model: "text-embedding-3-small" });

export type SearchHit = {
  id: string;
  content: string;
  metadata: Record<string, unknown>;
  source: string;
  semanticRank: number | null;
  keywordRank: number | null;
  combinedScore: number;
};

export async function searchDocuments(args: {
  workspaceId: string;
  query: string;
  matchCount?: number;
  filter?: Record<string, unknown>;
}): Promise<SearchHit[]> {
  const [queryEmbedding] = await embeddings.embedDocuments([args.query]);

  const rows = await db.execute(sql`
    SELECT * FROM search_documents(
      ${args.workspaceId}::uuid,
      ${JSON.stringify(queryEmbedding)}::vector,
      ${args.query},
      ${args.matchCount ?? 5},
      ${JSON.stringify(args.filter ?? {})}::jsonb
    )
  `);

  return rows as unknown as SearchHit[];
}
```

`embedDocuments` (plural) is the right method here — it is the batch endpoint and it is what the LangChain `PGVectorStore` uses internally.

### Hybrid search: when and why

Pure semantic search loses on **exact match** queries: product names, SKUs, error codes, function names, proper nouns. Pure keyword search loses on **paraphrased** queries: "how do I get my money back" vs "refund policy." Hybrid wins on both because it gets two chances to rank a chunk.

```ts
// Pure semantic — fast, good for paraphrased queries
const a = await searchDocuments({ workspaceId, query: "how do I get a refund" });

// Pure keyword — fast, good for exact-match queries
const b = await searchDocuments({ workspaceId, query: "SKU-12345 defective" });

// Hybrid — the function above — wins on both
const c = await searchDocuments({ workspaceId, query: "refund for SKU-12345" });
```

In 2026 the production RAG default is hybrid with RRF fusion. The query above gives you both signals in a single round trip; the function is your one call.

### Multi-tenancy: the non-negotiable

Every query MUST scope by `workspace_id`. There is no exception, no "we will add it later," no "this is the admin endpoint." Multi-tenant data leaks are a P0, a lawsuit, and a career-ender. The pattern is:

1. Encode `workspace_id` in the JWT (Mission 4 did this).
2. Extract it in a middleware, attach it to `c.get("auth")`.
3. Pass it as the first argument to every query.
4. Write a test that proves a JWT for workspace A returns zero rows from workspace B's documents.

```ts
// src/routes/search.ts
app.post("/api/search", async (c) => {
  const { workspaceId } = c.get("auth");      // from JWT — do NOT trust the body
  const body = SearchBody.parse(await c.req.json());
  const hits = await searchDocuments({
    workspaceId,
    query:     body.query,
    matchCount: body.matchCount,
    filter:    body.filter,
  });
  return c.json({ hits });
});
```

The test that catches the leak:

```ts
test("search does not leak across workspaces", async () => {
  await ingest({ workspaceId: "ws_A", content: "the secret launch is in Q3" });
  await ingest({ workspaceId: "ws_B", content: "the secret launch is in Q4" });

  const aHits = await searchDocuments({ workspaceId: "ws_A", query: "secret launch" });
  expect(aHits.every((h) => h.content.includes("Q3"))).toBe(true);
  expect(aHits.some((h) => h.content.includes("Q4"))).toBe(false);
});
```

If this test ever fails, you have a bug that is a worse class of incident than an outage. Block the deploy.

### Choosing a vector store for production

A decision matrix, end-of-module:

| Situation                                         | Use                   |
| ------------------------------------------------- | --------------------- |
| You are on Postgres, < 10M vectors                | **pgvector**          |
| You have > 10M vectors, you have a DBA            | pgvector with HNSW, possibly sharded |
| You have > 100M vectors, you need 10ms p99         | Qdrant or Pinecone    |
| You need zero ops, you are pre-PMF                | Pinecone (managed)    |
| You need rich filters + GraphQL + hybrid built in | Weaviate              |
| You are prototyping on your laptop                | Chroma (in-memory)    |

The cost axis is the one founders get wrong. A "free" Pinecone plan is a marketing surface; the moment you exceed it, the bill is per-pod-hour and per-query. pgvector is "free" because you are already paying for Postgres — the marginal cost is storage and a bit of CPU, both negligible compared to the LLM calls that will happen on top.

## Common mistakes

- **Forgetting `ORDER BY embedding <=> $1 LIMIT N`.** A query without an `ORDER BY` and a `LIMIT` is not a vector query; it is a table scan. The index will not be used.
- **Building the HNSW index before bulk-loading data.** HNSW build is O(N log N) on the existing data. If you create the index, then insert 10M rows, Postgres will do it 10M times. Insert first, build the index after. Or use `CREATE INDEX CONCURRENTLY` and accept the slower build.
- **Indexing the wrong column or the wrong operator class.** `vector_cosine_ops`, `vector_l2_ops`, `vector_ip_ops` are three different indexes. Pick one (or three) and stick with them. Mixing operators across indexes is silently slow.
- **Querying without `workspace_id` in the `WHERE` clause.** Even with an index, the planner can pick a bad plan. The filter is also a correctness issue, not just performance.
- **Not normalizing vectors before cosine search.** pgvector's `<=>` does not normalize for you. Either normalize in the application or use `<->` (L2) on normalized vectors (where the orderings are equivalent).
- **Using `SELECT *` from a vector column.** A 1536-dim vector is ~12KB per row. If you return 100 rows, that is 1.2MB of floats. Project the columns you need (`id`, `content`, `metadata`, and the distance).
- **Setting `ef_search` to 1000 because "higher = better."** It is, until your p95 latency doubles. The right value is the smallest one that hits your recall target on your eval set.
- **Re-running the same `embedQuery` 100 times in a loop.** The `embedDocuments` batch endpoint is faster and cheaper per call. Use it.

## Exercise

Wire pgvector into the Mission 4 SaaS end to end. Acceptance criteria:

1. Migration `0015_documents.sql` creates the `documents` table, the HNSW index, the GIN index on `content_tsv`, and the `search_documents` function from this module.
2. `POST /api/documents` (already from Module 48) writes to this table via Drizzle.
3. `POST /api/search` calls `searchDocuments({ workspaceId, query, matchCount, filter })` and returns the hits.
4. Includes a migration test: a 10,000-row synthetic dataset is loaded; `search_documents` is called for 50 random queries; the top-1 result is judged by hand against a known answer for at least 5 queries; recall@5 is at least 0.8.
5. Includes the cross-tenant test from this module.
6. Includes a benchmark: `EXPLAIN ANALYZE` a `search_documents` call before and after creating the HNSW index; commit both plans to the PR description.

Stretch: add a `p_match_threshold` parameter to the function that drops results with `combined_score < p_match_threshold`, and use it to power a "no relevant docs found" refusal in Module 50.

## AI Industry Note (2026)

The vector database category in 2026 looks like the OLTP database category did in 2010: one general-purpose engine (Postgres + pgvector) and a small number of specialized engines (Pinecone, Qdrant, Weaviate) that exist because the general-purpose one eventually runs out of headroom. Three shifts to internalize:

1. **pgvector closed the performance gap.** HNSW in pgvector is within 20% of Qdrant and Pinecone on the standard ANN benchmarks as of 2026, and the gap is shrinking every release. The reason to leave pgvector is now an operational reason, not a performance reason.
2. **The "vector database" is becoming a feature, not a product.** The interesting question in 2026 is not "which vector DB" but "which combination of vector index + metadata index + filter planner + hybrid fusion." Postgres is the only store that lets you iterate on all four in one place.
3. **Hybrid is the default, and RRF is the standard fusion.** The single biggest quality improvement available to a RAG system in 2026 is "add a `tsvector` column and combine it with your vector search." Most teams leave 20–40% recall on the table by not doing this. The function in this module is the version you should ship.
