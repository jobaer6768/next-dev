# Reward Course 01 — System Design

> Reward module. Builds on Mission 2 (scalable backends) and Mission 4 (SaaS product). Estimated effort: 12–16 hours.

---

## Why this module matters

By Mission 4 you shipped a working SaaS in a single Node process with one Postgres database. That's a real achievement. But the moment a hiring manager asks "how would this scale to a million users?", the answer is not "buy a bigger box." It's a vocabulary, a set of trade-offs, and a way of thinking.

System design is that vocabulary. It is the single skill that separates a coder from an engineer. In 2026, every backend interview loop at a serious company includes a 45-minute system design round, and increasingly the AI-software-org interviews are also asking "how would you design the tool-use / RAG / agent loop" — which is just system design with a new vocabulary.

This module won't make you a staff engineer. It will make you fluent in the language, able to reason about trade-offs, and ready to design a real rate limiter, a real cache layer, and a real data model without freezing up.

---

## Learning objectives

By the end of this module you will be able to:

1. Distinguish system design from software architecture and HLD/LLD.
2. Choose between monolithic, modular monolith, and microservice topologies with reasons.
3. Capture functional and non-functional requirements before sketching a system.
4. Explain CAP and PACELC, and pick a consistency model that matches a real product need.
5. Design for availability using retries, backoff, and idempotency keys.
6. Add observability (metrics, logs, traces) to a distributed system without bolting it on after the fact.
7. Compare REST, GraphQL, gRPC, and webhooks and pick the right one for a given boundary.
8. Choose a SQL / NoSQL / NewSQL store with reasons, and design a sharding strategy.
9. Reason about trade-offs explicitly instead of defaulting to "the cloud-native way."
10. Design a working rate limiter (token bucket + sliding window) from scratch.

---

## 1. What is System Design?

**System design** is the discipline of turning requirements into a coherent blueprint for a software system: components, data flows, data stores, interfaces, failure modes, and the trade-offs between them. It is the "what" and "why" of how the pieces fit together.

It is *not*:

- **Software architecture** — architecture is a *subset* of system design focused on the structure of the code: modules, patterns, frameworks, boundaries. Architecture answers "how is the codebase organized?" System design answers "how does the running system behave under load?"
- **HLD (High-Level Design)** — HLD is the *output* of a system design exercise: a diagram with boxes and arrows, written for stakeholders. LLD (Low-Level Design) is the next step down: class diagrams, sequence diagrams, API contracts. System design is the *process*; HLD/LLD are the *artifacts*.

A useful mental model:

```
Requirements
   ↓
System Design (process)         ← decisions, trade-offs, reasoning
   ↓
HLD (diagram, 1 page)           ← for managers, PMs, new hires
   ↓
LLD (sequence, schema, API)     ← for the implementer
   ↓
Code
```

If you can defend each arrow, you are doing system design. If you skip from requirements to code, you are hacking.

---

## 2. System Design vs Software Architecture

| Dimension | System Design | Software Architecture |
|---|---|---|
| Question | Will it scale? Will it survive a disk failure? | Is the code organized so we can change it? |
| Audience | Cross-functional: PM, SRE, security | Mostly engineers |
| Time horizon | 1–5 years | 3–18 months |
| Granularity | Services, queues, data stores, SLOs | Modules, packages, interfaces, patterns |
| Failure focus | What happens when a node dies? | What happens when a team changes the contract? |
| Output | An HLD diagram + a decision log | A module map + dependency rules |

The two are tightly coupled. A bad system design forces bad architecture (microservices with shared databases = distributed monolith). A bad architecture forces a bad system design (a tangled monolith can't be split into services without rewriting everything).

**Heuristic:** do system design first at the whiteboard. Then design the architecture that supports it. Never the other way around.

---

## 3. Monolithic vs Microservices

### Monolith

One deployable unit, one database, one process tree. Simple to develop, test, debug, deploy. The default for any new product.

```text
┌─────────────────────────────┐
│         Monolith            │
│  ┌─────┐ ┌─────┐ ┌──────┐   │
│  │ Auth│ │Bills│ │Reports│   │   ← all modules, one process
│  └──┬──┘ └──┬──┘ └──┬───┘   │
│     └───────┼────────┘       │
│             ▼                │
│         Postgres             │
└─────────────────────────────┘
```

**When it's right:** 0–10 engineers, unclear domain boundaries, product-market-fit still being found.

### Microservices

Many small services, each owning its data, deployed independently. Independent scaling, independent failure, independent release cadence.

```text
┌──────┐  ┌──────┐  ┌──────┐
│ Auth │  │Bills │  │Reports│   ← each a process, each its DB
│  DB  │  │  DB  │  │  DB  │
└──┬───┘  └──┬───┘  └──┬───┘
   └────┬────┴────┬────┘
        ▼         ▼
      Kafka    Event Bus
```

**When it's right:** 30+ engineers, clear bounded contexts, regulatory per-service isolation (e.g., a billing service with PCI scope).

**The cost:** distributed systems are *hard*. You now have network failures, partial outages, eventual consistency, distributed tracing, and a need for a platform team. Most companies that "go microservices" regret it within 18 months and either roll back or hire a platform team of 20.

### The Modular Monolith (the middle path)

One deployable unit, but the codebase is partitioned into modules with strict dependency rules and a clear path to extract a module into a service later. This is what Sam Newman and Simon Brown have been recommending for a decade, and what most senior engineers will tell you to do first.

```text
┌──────────── Monolith (1 binary, 1 DB) ─────────────┐
│  ┌─────────┐  ┌──────────┐  ┌────────────┐         │
│  │ auth/   │  │ billing/ │  │ reports/   │         │
│  │  domain │  │  domain  │  │  domain    │         │
│  └────┬────┘  └─────┬────┘  └──────┬─────┘         │
│       └──── public API only, no cross-imports ───── │
│                     │                               │
│                     ▼                               │
│                Postgres (schemas per module)        │
└────────────────────────────────────────────────────┘
```

**The rule:** modules communicate through *published interfaces*, not direct imports. If `billing/` imports from `auth/` directly, you've leaked a boundary.

**Decision rule of thumb:**

- < 10 engineers → modular monolith
- 10–30 engineers, clear bounded contexts → modular monolith, extract the 1–2 services that need independent scaling
- 30+ engineers, regulated workloads → microservice topology with a platform team

---

## 4. Functional vs Non-Functional Requirements

**Functional requirements (FRs)** describe *what* the system does. They are testable as features.

> "Users can upload a PDF and ask questions about it."

**Non-functional requirements (NFRs)** describe *how well* the system does it. They are testable as SLOs (service-level objectives).

> "P95 latency for an answer is under 3 seconds. System survives a single AZ failure. P99 uptime is 99.9% over 30 days."

### The most important NFRs

| NFR | What it measures | Example SLO |
|---|---|---|
| Latency | How long a request takes | P95 < 300ms |
| Availability | % of time the system responds | 99.9% (43.8 min downtime/month) |
| Durability | % of data not lost | 99.999999% (S3's claim) |
| Throughput | Requests per second | 5,000 RPS sustained |
| Consistency | How soon all reads see the latest write | "Read-your-writes" |
| Cost | $ per 1M operations | < $0.50 per 1M tokens |

### Why NFRs come first

If you design before you know the NFRs, you will either over-engineer (Kafka, microservices, vector DB) or under-engineer (no retries, no caching, single AZ). Always write the NFRs down *before* drawing the HLD. This is the single most common reason system design interviews go badly: the candidate jumps to solutions.

---

## 5. CAP and PACELC

**CAP** (Eric Brewer, 2000): in a distributed system facing a network partition, you must choose between:

- **C**onsistency — every read sees the latest committed write
- **A**vailability — every request gets a response (even if stale)
- **P**artition tolerance — the system continues to operate despite network failures between nodes

In practice, P is forced — networks fail. So CAP is really about C vs A *during* a partition.

**PACELC** (Daniel Abadi, 2012) extends this: **E**lse (when there is no partition), the system trades **L**atency against **C**onsistency.

| System | Partition behavior | Else |
|---|---|---|
| RDBMS (single-leader) | C — refuses writes if replica unreachable | C — strong consistency, higher latency |
| DynamoDB / Cassandra | A — accepts writes, reconciles later | L — low latency, eventual consistency |
| Cosmos DB | Configurable per request | Configurable per request |

**The lesson for you:** in 2026, the default is AP (available + partition-tolerant) for user-facing systems, because user-perceived availability matters more than strict consistency. But for *money* — billing, payments, inventory — you want CP. The mistake is using one database for both.

---

## 6. Availability and Fault Tolerance

Availability = uptime / total time. Industry shorthand is the "nines":

| SLO | Downtime/month | What it costs |
|---|---|---|
| 99% (two nines) | 7h 18m | Hobby project |
| 99.9% (three nines) | 43m 49s | Most SaaS |
| 99.99% (four nines) | 4m 23s | Banks, telecom |
| 99.999% (five nines) | 26s | Carrier-grade, hard |

**Fault tolerance** is the property of staying available *despite* component failures. A system with no redundancy has 0% fault tolerance; one with 2 replicas in different AZs can survive one AZ outage.

### Patterns

- **Redundancy** — N+1, N+2 replicas. The "N" is the count you need; +1 means you can lose one.
- **Failover** — primary/replica with automatic promotion (RDS Multi-AZ, Patroni, etcd leader election).
- **Bulkhead** — isolate failure domains so one slow downstream doesn't take down the whole system (separate connection pools, separate worker pools).
- **Circuit breaker** — stop calling a failing downstream for a cooldown period. (We used `opossum` in Mission 2.)
- **Graceful degradation** — when a feature is down, serve *something* instead of an error. Cache fallback, static fallback, "this feature is temporarily unavailable."

---

## 7. Retry Mechanisms, Backoff, and Idempotency

Naive code:

```ts
// NEVER DO THIS
async function charge(card) {
  for (let i = 0; i < 3; i++) {
    try { return await stripe.charge(card); }
    catch (e) { /* swallow */ }
  }
}
```

This will charge the customer's card three times during a transient network blip. The fix is three things together.

### Backoff

Wait longer between retries. **Exponential** with **jitter** is the standard:

```ts
function backoffMs(attempt: number, base = 200, cap = 30_000) {
  const exp = Math.min(cap, base * 2 ** attempt);
  return exp / 2 + Math.random() * (exp / 2); // full jitter
}
// attempt 0: 100–200ms
// attempt 1: 200–400ms
// attempt 2: 400–800ms
// attempt 3: 800–1600ms
// capped at 30s
```

### Idempotency

An idempotent operation can be applied multiple times without changing the result beyond the first application. The contract: client sends an `Idempotency-Key` header. Server stores (key → response) for 24h. If the same key arrives again, return the stored response.

```ts
app.post("/charges", async (req, res) => {
  const key = req.header("Idempotency-Key");
  if (!key) return res.status(400).json({ error: "missing key" });

  const cached = await redis.get(`idem:${key}`);
  if (cached) return res.json(JSON.parse(cached));

  const result = await stripe.charge(req.body);
  await redis.set(`idem:${key}`, JSON.stringify(result), "EX", 86400);
  res.json(result);
});
```

### Retry budget

Don't retry everything. Set a budget: in any 10-second window, only retry 10% of total requests. Otherwise a slow downstream turns your retry storm into a self-DDoS.

```ts
// pseudo
if (metrics.retryRate() > 0.1) return; // give up
```

---

## 8. Observability: Metrics, Logs, Traces

A distributed system you cannot see is a distributed system you cannot operate. Observability has three pillars — and they answer different questions.

| Pillar | Question it answers | Example |
|---|---|---|
| **Metrics** | How much / how fast? | `http_requests_total{status="500"}` |
| **Logs** | What happened to *this* request? | `2026-01-15T10:23:01Z ERROR payment failed orderId=42` |
| **Traces** | Where did the time go across services? | A span tree: `api-gateway → auth → billing → stripe (800ms)` |

### What to instrument

In your SaaS from Mission 4, the minimum is:

```ts
// metrics
http_requests_total{method, route, status}
http_request_duration_seconds{method, route} histogram
db_query_duration_seconds{query} histogram
queue_lag_seconds{queue}

// logs (structured, JSON, with traceId)
logger.info({ traceId, userId, action: "charge", result: "ok" }, "charge succeeded");

// traces
span("db.query", { "db.statement": "SELECT ..." });
```

### The cardinal rule

Every log line and every span gets the same `traceId`. Propagate it via headers (`traceparent` in W3C Trace Context, `X-Request-Id` for app-level). This is what turns "logs" and "traces" into a story you can read.

---

## 9. API Architectural Styles

| Style | Strengths | Weaknesses | Use when |
|---|---|---|---|
| **REST** | Universal, cacheable, simple | Over/under-fetching, chatty | Public APIs, CRUD, defaults |
| **GraphQL** | Client specifies shape; one endpoint | Hard to cache, N+1, complexity | Mobile apps,聚合 views, BFF |
| **gRPC** | Fast (binary), streaming, typed | Not human-readable, browser-hostile | Service-to-service, mobile→backend |
| **Webhooks** | Push, no polling, decoupled | Inbox problem, retries | Outbound notifications, Stripe events |

In your SaaS you will likely use:

- **REST** (or tRPC) for the browser → API boundary
- **gRPC** (or just HTTP/JSON) for service-to-service if you split anything out
- **Webhooks** for outbound integrations (Stripe, GitHub, etc.)

Don't use GraphQL unless you have a real reason. It has real costs.

---

## 10. SQL, NoSQL, and NewSQL

| Family | Examples | Strengths | Trade-off |
|---|---|---|---|
| **SQL** | Postgres, MySQL | Strong consistency, joins, mature | Vertical scaling ceiling; schema migration pain |
| **NoSQL — document** | MongoDB, Firestore | Flexible schema, easy to start | Joins in app code; weak consistency |
| **NoSQL — key-value** | Redis, DynamoDB | Latency, scale, simple | No joins, no rich queries |
| **NoSQL — wide-column** | Cassandra, HBase | Massive write throughput | Query patterns baked in at design time |
| **NewSQL** | CockroachDB, Spanner, TiDB | SQL + horizontal scale | Operational complexity, cost |

**Decision rule:** start with Postgres. Move to DynamoDB or a document store only when you have a *specific* query or scale problem Postgres cannot solve. The "MongoDB because JSON" era is over; Postgres has had JSONB for a decade and it is excellent.

---

## 11. Database Partitioning and Sharding

**Partitioning** (vertical or horizontal) is splitting a single DB's tables for performance or organization. **Sharding** is partitioning *across machines* — each shard holds a subset of the data.

### Sharding strategies

- **Hash sharding** — `shard = hash(user_id) % N`. Even distribution. Hard to rebalance when N changes.
- **Range sharding** — `shard = user_id in [0..1M), [1M..2M)`. Easy range queries. Hot spots if one range is busy.
- **Geographic sharding** — `shard = region`. Low latency per region. Cross-region queries are expensive.
- **Directory sharding** — a lookup table maps entity → shard. Flexible. The lookup is a single point of failure unless you make it HA.

**The painful part:** once you shard, joins across shards are expensive. You denormalize aggressively. This is why most teams avoid sharding until they *have* to, and instead scale vertically (bigger Postgres) or use read replicas.

---

## 12. System Design Trade-Offs

Every decision in system design is a trade-off. The senior-engineer move is to **name the trade-off explicitly** when you propose a design.

Examples:

> "I'll use Postgres for the primary store because we need ACID transactions for payments. Trade-off: we accept a vertical scaling ceiling around ~10TB. We mitigate that with read replicas for the reporting workload."

> "I'll use a message queue (SQS) between the upload service and the PDF parser. Trade-off: the user no longer gets a synchronous 'PDF parsed' response; the API returns a jobId and the client polls or subscribes via WebSocket. We accept that for resilience."

> "I'll cache the LLM response in Redis for 1 hour keyed by hash(prompt). Trade-off: identical prompts from different users share a cache, which means we must strip PII before hashing. We accept the engineering cost for the cost savings."

If you can't name the trade-off, you don't understand the decision yet.

---

## 13. Worked Example: Rate Limiter for a SaaS API

### Requirements

- **FRs:** Each API key can make N requests per minute. Returns 429 when exceeded.
- **NFRs:** P99 latency < 5ms for the check itself. Survives a single Redis node failure. Accurate to within 1% of true count.

### Algorithms

**Token bucket** — each key has a bucket of tokens, refilled at rate R. Each request costs 1 token. Smooth, allows bursts up to bucket size.

**Sliding window log** — store a list of timestamps per key. Count entries in the last 60s. Most accurate, most memory.

**Sliding window counter** — keep two counters: current minute, previous minute. Estimate with weighted average. Cheaper, ~99% accurate.

We will combine: **token bucket for burst tolerance, sliding window counter for accuracy.**

### Implementation

```ts
// redis-keys.ts
export const k = {
  bucket: (key: string) => `rl:bucket:${key}`,
  window: (key: string, windowSec: number, now: number) =>
    `rl:win:${key}:${Math.floor(now / windowSec)}`,
};

// token-bucket.ts
import { redis } from "./redis";
import { k } from "./redis-keys";

export async function takeToken(
  key: string,
  capacity: number,
  refillPerSec: number,
): Promise<{ allowed: boolean; remaining: number; retryAfter: number }> {
  const now = Date.now() / 1000;
  const lua = `
    local b = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
    local tokens = tonumber(b[1]) or tonumber(ARGV[1])
    local ts = tonumber(b[2]) or tonumber(ARGV[3])
    local delta = math.max(0, tonumber(ARGV[3]) - ts)
    tokens = math.min(tonumber(ARGV[1]), tokens + delta * tonumber(ARGV[2]))
    if tokens >= 1 then
      tokens = tokens - 1
      redis.call('HMSET', KEYS[1], 'tokens', tokens, 'ts', ARGV[3])
      redis.call('EXPIRE', KEYS[1], 3600)
      return {1, tokens, 0}
    else
      redis.call('HMSET', KEYS[1], 'tokens', tokens, 'ts', ARGV[3])
      return {0, tokens, (1 - tokens) / tonumber(ARGV[2])}
    end
  `;
  const [allowed, remaining, retryAfter] = await redis.eval(
    lua, 1, k.bucket(key), capacity, refillPerSec, now,
  );
  return {
    allowed: allowed === 1,
    remaining,
    retryAfter: Math.ceil(retryAfter),
  };
}
```

**Why Lua?** It runs atomically on the Redis server, so we don't have a race between "read tokens" and "write tokens" across two Node instances.

**Where to enforce:** an Express middleware that runs *before* the route handler. A `429` response includes `Retry-After` (seconds) and `X-RateLimit-Remaining` headers.

**Trade-offs to call out:**
- **Token bucket** is approximate during clock skew. We accept it; the alternative is a global clock.
- **Redis single point of failure.** Mitigation: use a Redis cluster with a replica; in a partition, fail *open* (allow the request) rather than fail *closed* — we have decided availability matters more than perfect limiting.
- **Per-key only.** A single bad actor can still hammer us with many API keys. We need a second layer (per-IP) at the edge.

This is the kind of 3-paragraph "trade-off narrative" you should be able to give for any design decision in an interview.

---

## Common mistakes

1. **Jumping to solutions.** "We'll use Kafka, Postgres, Redis, and a vector database" — but you haven't said what the system does.
2. **Ignoring NFRs.** Without numbers, "scalable" is meaningless. Always write SLOs.
3. **Microservices by default.** Most teams should not start there. Modular monolith first.
4. **CAP confusion.** Forgetting that P is forced. CAP is really C vs A *during* a partition.
5. **No idempotency on retries.** The classic double-charge bug. Every retried write needs a key.
6. **Observability as a phase-2 project.** If you bolt it on later, you will never have it. Instrument from day 1.
7. **Picking the database first.** Pick the *query patterns* first; the database follows.
8. **Hand-waving at scale.** "It will work for a million users" — show the math: QPS, storage, bandwidth, cost.

---

## Exercise

**Build a rate limiter for the Mission 4 SaaS.**

1. Add a `rate_limit` column to the `api_keys` table: `requests_per_minute INT`.
2. Implement the token-bucket Lua script above as a middleware in your Express API.
3. Add three headers to every response: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`.
4. Return `429` with a `Retry-After` header when exceeded. The body should be `{"error":"rate_limited","retryAfter":12}`.
5. Write a load test with `autocannon` or `k6` that proves the limit holds within 1% accuracy.
6. Add Prometheus metrics: `rate_limited_total{key}`, `rate_limit_check_duration_seconds` histogram.
7. **Stretch:** add a per-IP layer at NGINX using `limit_req_zone` so the limiter is enforced even before traffic hits Node.

Submit: the PR, the load-test report, and a 1-paragraph postmortem of any bugs you found along the way.

---

## AI Industry Note (2026)

In 2026, system design interviews at AI-native companies (Anthropic, OpenAI, Cursor, Vercel, Replit) increasingly include a second round called **"agent design"** or **"tool-use design."** It uses the same vocabulary — idempotency, retries, observability, trade-offs — but the components are: an LLM, a tool-call API, a context window, a vector store, and a memory layer. The candidate who can fluently reason about CAP, sharding, and idempotency keys is the candidate who can also reason about how to make a tool-calling agent safe and debuggable.

The meta-skill is the same: **name the trade-off, justify the decision, name the failure mode.** That is system design, and it transfers.

---

*Next: Reward Course 02 — NestJS Backend, where we apply these layered ideas with a real framework.*
