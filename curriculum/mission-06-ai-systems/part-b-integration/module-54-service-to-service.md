# Module 54: Service-to-Service Communication

## Why this module matters

A monolith can `import` a function. A microservice must make an HTTP request over an unreliable network with a hostile clock, an unknown version, and a peer that just restarted. Every attribute of a function call — types, errors, auth, timeouts — has to be re-engineered at the wire. Get this wrong and your "AI feature" becomes three days of "why is the user service 504ing on the agent service." Get it right and the system survives a Friday deploy.

This module is the connective tissue between Module 50–52 (where the AI lives) and Modules 55–58 (where the AI is exposed). It is also the module most bootcamps skip, and the one that separates a senior engineer from a mid.

## Learning objectives

1. Design a shared type contract (Zod) that survives serialization between services.
2. Implement three flavors of service-to-service auth (mTLS, service tokens, audience-scoped JWT) and pick the right one.
3. Apply the resilience triangle: timeouts, retries with jitter, and circuit breakers.
4. Propagate errors across the network in a shape the caller can actually handle.
5. Wire health and readiness endpoints that tell a load balancer when to send traffic.

## Conceptual model

Every service-to-service call has six ingredients. Skip any one and you will regret it in production.

| Ingredient | Question it answers | Failure mode if missing |
|---|---|---|
| Contract | What shape is the request/response? | "It works on my machine" (it parsed different JSON) |
| Auth | Who is calling? | Open S3 buckets |
| Timeout | How long do I wait? | Cascading failures |
| Retry | Do I try again? | Permanent failures from a transient blip |
| Circuit breaker | Should I stop trying? | Thundering herd against a downed service |
| Health | Is this instance OK? | Sticky traffic to a dead pod |

### 1. Shared contracts (Zod, generated types)

```ts
// packages/contracts/src/agent.ts
import { z } from "zod";

export const AgentRunRequest = z.object({
  workspaceId: z.string().uuid(),
  userId: z.string().uuid(),
  question: z.string().min(1).max(8000),
  threadId: z.string().uuid().optional(),
});
export type AgentRunRequest = z.infer<typeof AgentRunRequest>;

export const AgentRunResponse = z.object({
  threadId: z.string().uuid(),
  answer: z.string(),
  citations: z.array(z.object({ source: z.string(), chunkIndex: z.number() })),
  tokensUsed: z.number().int().nonnegative(),
});
export type AgentRunResponse = z.infer<typeof AgentRunResponse>;

export const HttpError = z.object({
  code: z.enum(["UNAUTHENTICATED", "FORBIDDEN", "NOT_FOUND", "RATE_LIMITED", "UPSTREAM_UNAVAILABLE", "INVALID_ARGUMENT", "INTERNAL"]),
  message: z.string(),
  retryable: z.boolean(),
  details: z.record(z.unknown()).optional(),
});
export type HttpError = z.infer<typeof HttpError>;
```

The discipline: **one package, many services**. Publish `packages/contracts` to a private npm registry. Every service `import { AgentRunRequest } from "@acme/contracts"`. The Zod schema is the source of truth; the TS type is just a hint.

Validate at the boundary:

```ts
// services/agent/src/server.ts
import { AgentRunRequest, HttpError } from "@acme/contracts";

app.post("/agent/run", async (req, res) => {
  const parsed = AgentRunRequest.safeParse(req.body);
  if (!parsed.success) {
    return res.status(400).json({
      code: "INVALID_ARGUMENT",
      message: "Bad request",
      retryable: false,
      details: { issues: parsed.error.issues },
    } satisfies HttpError);
  }
  // ... call the agent
});
```

### 2. Service-to-service auth: pick the right tool

| Mechanism | Use when | Pros | Cons |
|---|---|---|---|
| **Service token (shared secret)** | Internal cron jobs, low-risk reads | Simple, fast | Rotation is a deploy; leak is catastrophic |
| **mTLS** | High-trust internal mesh (Istio, Linkerd) | Cryptographic identity per pod | Operational complexity, cert rotation |
| **Audience-scoped JWT** | Public-ish internal APIs, multi-tenant | Stateless, expiring, auditable | Clock skew, key management |
| **OAuth2 client credentials** | Cross-org B2B | Standardized, revocable | Slow to set up |

The 2026 default for most teams: **service mesh (mTLS) for east-west, JWT for north-south**. If you do not have a service mesh, audience-scoped JWTs are the next best thing.

```ts
// packages/auth/src/service-jwt.ts
import { SignJWT, jwtVerify } from "jose";

const secret = new TextEncoder().encode(process.env.SERVICE_JWT_SECRET!);

export async function signServiceToken(svc: string, audience: string) {
  return new SignJWT({ svc })
    .setProtectedHeader({ alg: "HS256" })
    .setIssuer("acme-gateway")
    .setAudience(audience)
    .setIssuedAt()
    .setExpirationTime("5m")
    .sign(secret);
}

export async function verifyServiceToken(token: string, expectedAud: string) {
  const { payload } = await jwtVerify(token, secret, { audience: expectedAud, issuer: "acme-gateway" });
  return payload as { svc: string; aud: string };
}
```

A call:

```ts
// services/billing/src/lib/agent-client.ts
import { signServiceToken, verifyServiceToken } from "@acme/auth";
import { AgentRunRequest, AgentRunResponse } from "@acme/contracts";

export async function callAgent(req: AgentRunRequest): Promise<AgentRunResponse> {
  const token = await signServiceToken("billing", "agent");
  const res = await fetch(`${process.env.AGENT_URL}/agent/run`, {
    method: "POST",
    headers: { authorization: `Bearer ${token}`, "content-type": "application/json" },
    body: JSON.stringify(req),
  });
  if (!res.ok) throw await asHttpError(res);
  return AgentRunResponse.parse(await res.json());
}
```

### 3. The resilience triangle

```ts
// packages/resilience/src/fetcher.ts
import pRetry from "p-retry";
import { CircuitBreaker } from "opossum";
import { HttpError } from "@acme/contracts";

type FetchOpts = RequestInit & { timeoutMs?: number; idempotencyKey?: string };

export function buildSafeFetch(baseUrl: string, defaultTimeout = 5_000) {
  const raw = async (path: string, opts: FetchOpts = {}) => {
    const ctrl = new AbortController();
    const t = setTimeout(() => ctrl.abort(), opts.timeoutMs ?? defaultTimeout);
    try {
      const res = await fetch(baseUrl + path, { ...opts, signal: ctrl.signal });
      if (!res.ok) throw await asHttpError(res);
      return res;
    } finally {
      clearTimeout(t);
    }
  };

  // Retry: only on retryable errors, with exponential backoff + jitter, max 3.
  const withRetry = (path: string, opts: FetchOpts) =>
    pRetry(() => raw(path, opts), {
      retries: 3,
      minTimeout: 200,
      maxTimeout: 2_000,
      factor: 2,
      randomize: true,
      shouldRetry: (err) => err instanceof HttpError && err.retryable,
    });

  // Circuit breaker: open after 5 failures in 10s; half-open after 30s.
  const breaker = new CircuitBreaker(withRetry, {
    timeout: 10_000,
    errorThresholdPercentage: 50,
    resetTimeout: 30_000,
  });

  return (path: string, opts: FetchOpts) => breaker.fire(path, opts) as Promise<Response>;
}
```

The three rules:

1. **Always set a timeout.** `fetch` defaults to *no* timeout, which means *forever*. Five seconds is a sensible default for an internal call.
2. **Retry only retryable errors.** Retry a 502. Do NOT retry a 400. Use the `retryable` field in `HttpError`.
3. **Idempotency keys for non-idempotent ops.** A POST that creates a row must carry an `Idempotency-Key` header so the retry does not double-bill.

### 4. Error propagation

The shape that survives the boundary:

```ts
// packages/errors/src/http.ts
import { HttpError } from "@acme/contracts";

export async function asHttpError(res: Response): Promise<HttpError> {
  try {
    const body = HttpError.parse(await res.json());
    return body;
  } catch {
    return {
      code: res.status >= 500 ? "UPSTREAM_UNAVAILABLE" : "INVALID_ARGUMENT",
      message: `Upstream returned ${res.status}`,
      retryable: res.status >= 500,
    };
  }
}
```

Always preserve `retryable` and `code`. The caller's circuit breaker and retry policy depend on them. A bare `throw new Error(res.statusText)` is a debugging nightmare.

### 5. Health, readiness, and the load balancer

```ts
// services/agent/src/health.ts
import { checkpointer } from "@acme/agents/memory";
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

export async function liveness() {
  // Am I running? Used to kill a stuck pod.
  return { ok: true };
}

export async function readiness() {
  // Should I get traffic? Checks downstream dependencies.
  const checks = await Promise.allSettled([
    pool.query("SELECT 1").then(() => "postgres"),
    checkpointer.ping().then(() => "checkpointer"),
    fetch("https://api.openai.com/v1/models", { headers: { authorization: `Bearer ${process.env.OPENAI_API_KEY}` } }).then((r) => r.ok ? "openai" : Promise.reject()),
  ]);
  const failing = checks.filter((c) => c.status === "rejected").length;
  return { ok: failing === 0, checks };
}
```

Expose them:

```ts
app.get("/healthz", async (_req, res) => res.json(await liveness()));
app.get("/readyz", async (_req, res) => {
  const r = await readiness();
  res.status(r.ok ? 200 : 503).json(r);
});
```

Kubernetes / Fly / Render will use `/readyz` to decide when to send traffic. If your checkpointer is down, the pod takes itself out of the pool. That is how a cascading failure becomes a graceful degradation.

## Common mistakes

1. **No timeout on `fetch`.** The single most common production bug in service-to-service code. Set it always.
2. **Retrying 4xx.** A 400 is the caller's fault; retrying just spams logs. Only retry 5xx and 429.
3. **Sharing the user JWT across services.** The user's token should not flow east-west. Mint a short-lived service token for the call.
4. **Trusting `res.ok`.** A 200 with a malformed body is still a 200. Always Zod-parse the response.
5. **No idempotency key on POSTs.** A retry of a "send email" call sends two emails. The user does not appreciate the joke.
6. **Health endpoint that lies.** `/healthz` returns 200 because the process is alive, but the DB is down. Separate liveness from readiness; make readiness check downstream.
7. **Tight coupling on error strings.** `"Agent returned an error"` is useless. Use a code (`UPSTREAM_UNAVAILABLE`) and a retryable flag.

## Exercise

Refactor your Mission 4 SaaS so the `agent`, `kb`, and `billing` services are deployed independently and talk over HTTP.

**Acceptance criteria**

1. A shared `packages/contracts` package defines Zod schemas for every cross-service call.
2. Every inter-service call uses a 5-second timeout, retries only on `retryable: true`, and includes an `Idempotency-Key` for POSTs.
3. Each service exposes `/healthz` and `/readyz`; readiness fails if Postgres or the checkpointer is unreachable.
4. A circuit breaker around the agent client opens after 5 failures in 10 seconds and half-opens after 30.
5. Add a `pnpm run chaos` script that kills one service for 30 seconds; verify the others surface a clean `UPSTREAM_UNAVAILABLE` and the user sees a "service is busy, retrying" UI, not a stack trace.

**Stretch**

- Add OpenTelemetry tracing across all three services and visualize one user request as a flame graph in Jaeger.
- Add mTLS with Linkerd or Istio; remove the JWT and prove the same code still works.

## AI Industry Note (2026)

Service-to-service communication is unsexy and unglamorous and the reason every senior engineer is paid what they are paid.

- **The "service mesh tax" has paid off.** In 2026, most production Kubernetes clusters ship Linkerd or Istio by default. mTLS is the floor, not the ceiling. Newer projects (Cilium's service mesh) are pushing it into the kernel.
- **Idempotency keys are now expected at the framework level.** Stripe popularized them; every internal API gateway (Module 55) now requires them. `bullmq` even has an `attempts` model that fits the same pattern.
- **Resilience is a feature.** "How does your system fail?" is now a first-class architecture question, not a war-room question. Companies publish "graceful degradation" pages the way they publish pricing pages.
- **OpenTelemetry won the tracing war.** Jaeger, Honeycomb, Datadog, Lightstep, and Grafana Tempo all consume OTLP. Wire it once.
- **The new bottleneck is schema drift.** Multi-service apps change fast. `protobuf`, `Connect`, and `tRPC` are eating the "REST + Zod" pattern for greenfield; Zod still wins for browser-to-server contracts where the bundle size matters.
- **AI workloads add a new failure mode: the cost cap.** A misbehaving agent can burn through a quarter's LLM budget in 10 minutes. Module 52's `HARD_TOKEN_CAP` is a service-to-service concern too — every LLM call from a microservice must check the budget before it calls.