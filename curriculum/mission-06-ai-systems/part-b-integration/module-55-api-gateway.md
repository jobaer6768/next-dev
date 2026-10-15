# Module 55: API Gateway & Request Routing

## Why this module matters

The moment your SaaS has more than one backend — your Next.js app, your agent service, your kb service, your billing service — you have a routing problem. The user has one URL; you have five origins. Without a gateway, every frontend feature hardcodes five URLs, every CORS exception becomes a meeting, and the security team writes the same auth check in six different places.

The API gateway is the chokepoint. It is where authentication, rate limiting, tracing, and versioning happen *once* instead of *everywhere*. It is also the file that, if you get it wrong, takes the whole product down. This module is the one your future on-call self will thank you for.

## Learning objectives

1. Explain the gateway pattern and when to use one (and when not to).
2. Route requests across backend, AI, and worker services by path, method, and header.
3. Implement gateway-level auth: verify the JWT once, attach the user/workspace to every downstream call.
4. Apply request and response transformations (header rewriting, legacy adapters, version normalization).
5. Build centralized rate limiting, logging, and tracing.
6. Version APIs with `/api/v1/`, `/api/v2/` and the deprecation playbook.

## Conceptual model

A gateway is a reverse proxy plus a policy engine. The 2026 stack is usually one of:

| Choice | When to use |
|---|---|
| **Cloud-native** (AWS API Gateway, GCP API Gateway, Cloudflare Workers) | Cloud-only, want zero ops, OK with per-request cost |
| **Self-hosted (Envoy + custom control plane)** | Kubernetes, need full control, can pay the complexity |
| **App-level (Next.js middleware + tRPC)** | Small surface, single team, want one deploy |
| **Edge (Vercel/Cloudflare + Next.js Middleware)** | Public B2C, latency-sensitive, geographic routing |

For a Mission 4 SaaS with three internal services, the right answer in 2026 is usually **edge middleware for the public surface, internal gateway for the service mesh**. We'll build the latter.

### 1. Routing across services

```ts
// gateway/src/router.ts
type Route = { match: (req: Request) => boolean; upstream: string; rewritePath?: (p: string) => string };

const routes: Route[] = [
  {
    match: (req) => req.url.startsWith("/api/v1/agent") || req.url.startsWith("/api/v1/threads"),
    upstream: process.env.AGENT_URL!,
    rewritePath: (p) => p.replace(/^\/api\/v1/, ""), // strip the version prefix
  },
  {
    match: (req) => req.url.startsWith("/api/v1/kb"),
    upstream: process.env.KB_URL!,
    rewritePath: (p) => p.replace(/^\/api\/v1\/kb", ""),
  },
  {
    match: (req) => req.url.startsWith("/api/v1/billing"),
    upstream: process.env.BILLING_URL!,
  },
  {
    // Catch-all for the Next.js app itself
    match: () => true,
    upstream: process.env.WEB_URL!,
  },
];

export function pickRoute(req: Request): Route {
  return routes.find((r) => r.match(req))!;
}
```

The key decision: **path-based** routing is the default; **host-based** or **header-based** routing is for advanced cases (e.g., enterprise customers with dedicated backends).

### 2. Auth at the gateway, identity propagation downstream

The gateway verifies the JWT once. Downstream services never see the raw token; they see a trusted header that says "this is user X in workspace Y."

```ts
// gateway/src/middleware/auth.ts
import { jwtVerify } from "jose";

const key = new TextEncoder().encode(process.env.JWT_PUBLIC_KEY!);

export async function attachIdentity(req: Request): Promise<Request> {
  const auth = req.headers.get("authorization");
  if (!auth?.startsWith("Bearer ")) throw new HttpError(401, "UNAUTHENTICATED");

  const { payload } = await jwtVerify(auth.slice(7), key, { issuer: "acme-auth" });
  const userId = payload.sub!;
  const workspaceId = payload.workspace_id as string;

  // Forward trusted identity headers. NEVER forward the original JWT to internal services.
  const headers = new Headers(req.headers);
  headers.set("x-internal-user-id", userId);
  headers.set("x-internal-workspace-id", workspaceId);
  headers.set("x-internal-request-id", crypto.randomUUID());
  // Strip the original auth header so it cannot leak downstream.
  headers.delete("authorization");
  return new Request(req, { headers });
}
```

The discipline: **the gateway is the only place that knows the user's token**. The agent service receives `x-internal-user-id` and trusts it. If someone manages to call the agent service directly, they cannot impersonate; the agent refuses any request without the internal header.

### 3. Request and response transformation

A common case: the agent service returns `{ answer, citations, tokensUsed }` but an old mobile app expects `{ response, sources, usage }`. The gateway is the right place to translate.

```ts
// gateway/src/transform/agent.ts
export async function adaptAgentResponse(req: Request, res: Response): Promise<Response> {
  const isLegacyMobile = req.headers.get("x-client") === "mobile-legacy";
  if (!isLegacyMobile) return res;

  const body = await res.json() as { answer: string; citations: { source: string }[]; tokensUsed: number };
  return new Response(
    JSON.stringify({
      response: body.answer,
      sources: body.citations.map((c) => c.source),
      usage: { tokens: body.tokensUsed },
    }),
    { status: res.status, headers: res.headers }
  );
}
```

The rule: **transformations live in the gateway, not in every service**. If a new client needs a new shape, the gateway grows; the agent service stays clean.

### 4. Centralized rate limiting

```ts
// gateway/src/middleware/rateLimit.ts
import { Redis } from "ioredis";

const redis = new Redis(process.env.REDIS_URL!);

type Bucket = { capacity: number; refillPerSec: number };

const BUCKETS: Record<string, Bucket> = {
  free:    { capacity: 10,   refillPerSec: 0.1 },
  pro:     { capacity: 100,  refillPerSec: 1   },
  enterprise: { capacity: 1000, refillPerSec: 10 },
};

export async function rateLimit(req: Request): Promise<void> {
  const userId = req.headers.get("x-internal-user-id");
  const plan = (req.headers.get("x-internal-plan") ?? "free") as keyof typeof BUCKETS;
  const key = `rl:${plan}:${userId}`;

  const bucket = BUCKETS[plan];
  const tokens = await redis.eval(
    `local key=KEYS[1]; local cap=tonumber(ARGV[1]); local refill=tonumber(ARGV[2]);
     local last=redis.call('hmget', key, 'tokens', 'ts')
     local cur=tonumber(last[1]) or cap
     local now=tonumber(ARGV[3])
     local elapsed=math.max(0, now - tonumber(last[2] or now))
     cur=math.min(cap, cur + elapsed*refill)
     if cur < 1 then return 0 end
     cur=cur-1
     redis.call('hmset', key, 'tokens', cur, 'ts', now)
     redis.call('expire', key, 60)
     return 1`,
    1, key, String(bucket.capacity), String(bucket.refillPerSec), String(Date.now()/1000)
  );

  if (tokens === 0) {
    throw new HttpError(429, "RATE_LIMITED", { retry_after_seconds: 1 });
  }
}
```

A token bucket per (user, plan) is the 2026 default. Sliding-window logs are fine for analytics; token buckets are simpler to reason about and behave well under burst.

### 5. Centralized logging and tracing

```ts
// gateway/src/middleware/observe.ts
import { trace, context, propagation } from "@opentelemetry/api";
import { pino } from "pino";

const log = pino();

export async function observe(req: Request, next: () => Promise<Response>): Promise<Response> {
  const reqId = req.headers.get("x-internal-request-id") ?? crypto.randomUUID();
  const tracer = trace.getTracer("gateway");
  const span = tracer.startSpan("gateway.request", {
    attributes: {
      "http.method": req.method,
      "http.target": req.url,
      "x.request_id": reqId,
    },
  });

  const t0 = performance.now();
  try {
    // Propagate W3C trace context downstream
    const headers = { ...Object.fromEntries(req.headers), traceparent: ... };
    const res = await context.with(trace.setSpan(context.active(), span), () => next());
    span.setAttribute("http.status_code", res.status);
    log.info({ reqId, status: res.status, ms: performance.now() - t0 }, "request");
    return res;
  } catch (e) {
    span.recordException(e as Error);
    throw e;
  } finally {
    span.end();
  }
}
```

The `traceparent` header is the magic that makes a single user request show up as one flame graph across five services. Without it, you are guessing.

### 6. API versioning

Three strategies:

| Strategy | Example | Pros | Cons |
|---|---|---|---|
| **URL path** | `/api/v1/agent` | Crystal clear, easy to route | Uglier URLs |
| **Header** | `Accept: application/vnd.acme.v2+json` | Clean URLs | Harder to debug in browser |
| **Query param** | `/api/agent?version=2` | Easy to test in cURL | Cache-unfriendly |

The default in 2026 is still **URL path**. It is the only one that works in every browser dev tool, every curl, every load tester.

```ts
const routes: Route[] = [
  { match: (r) => r.url.startsWith("/api/v1/agent"), upstream: process.env.AGENT_V1_URL! },
  { match: (r) => r.url.startsWith("/api/v2/agent"), upstream: process.env.AGENT_V2_URL! },
];
```

The deprecation playbook:

1. Ship `/v2` alongside `/v1`; the gateway routes both.
2. Set `Deprecation: true` and `Sunset: Sat, 01 Mar 2026 00:00:00 GMT` headers on `/v1` responses.
3. Email the customers still using `/v1` with their traffic numbers.
4. After the sunset date, the gateway returns `410 Gone` for `/v1`.

## Common mistakes

1. **Putting business logic in the gateway.** The gateway should not call the database. It should not "enrich" the response. It is plumbing.
2. **Forwarding the user JWT downstream.** Once a token leaves the gateway, you have no idea who can read it. Mint internal identity headers instead.
3. **Per-route rate limits only.** Add a *global* per-IP limit too, or one misbehaving tenant will DOS everyone.
4. **No correlation IDs.** Every request that doesn't carry an ID across services is a future 2am debugging session.
5. **Hardcoding upstream URLs.** Use env vars; make blue/green deploys a config change, not a deploy.
6. **Versioning by date.** `v2025-04-01` is unreadable. Use `v1`, `v2`, `v3`.
7. **Skipping the readiness gate on new versions.** A `/v2` that is 50% broken is worse than a `/v2` that is not yet routed to.

## Exercise

Stand up an API gateway in front of the three services from Module 54.

**Acceptance criteria**

1. The gateway routes `/api/v1/agent/*`, `/api/v1/kb/*`, and `/api/v1/billing/*` to the right upstream.
2. The gateway verifies the user JWT, mints a short-lived internal service token, and forwards `x-internal-user-id` and `x-internal-workspace-id`.
3. Per-workspace rate limiting at 10 RPS for the free plan, 100 RPS for pro, 1000 for enterprise. A request over the limit returns 429 with `Retry-After`.
4. Every request gets a `x-request-id`. The same ID is logged at the gateway, the agent, the KB, and the billing service.
5. Add a `/api/v2/agent` route that proxies to a *new* agent service you deploy as a side-by-side preview. The gateway routes 10% of traffic to v2 (canary), 90% to v1.

**Stretch**

- Add WAF-style rules: block any request whose body contains a known prompt-injection pattern (Module 52's regex list).
- Wire OpenTelemetry to Jaeger and visualize one user request across all four services as a single trace.

## AI Industry Note (2026)

The gateway is the most boring file in your repo. It is also the file that, when it goes down, takes the company with it.

- **AI gateways are a category now.** Cloudflare's AI Gateway, Portkey, and Helicone have turned "proxy your LLM calls with caching, fallbacks, cost tracking" into a product. Most teams now run an *AI gateway* inside their *API gateway*, with the AI gateway handling prompt logging, model fallbacks, and per-tenant token budgets.
- **Edge gateways won the public-surface war.** Vercel, Cloudflare, and Fastly ship a gateway out of the box. Most SaaS apps do not run their own public gateway in 2026; they run *one* (Cloudflare) and a self-hosted gateway for the internal mesh.
- **mTLS is table stakes.** In 2026, a service mesh without mTLS is a security review finding, full stop. Linkerd and Istio make it a one-line config.
- **Auth at the edge is the new pattern.** Verify the JWT at the edge, attach identity headers, and let internal services trust them. This is the inverse of the 2010s pattern of "every service re-verifies."
- **API versioning has gone quiet.** Most SaaS companies now version *the data model*, not the URL. `/v1` in the path is a 2010s habit; the 2026 pattern is `/api/agent` with a `version` field in the request body. We are teaching the path version because it is still the most legible, but be aware it is on its way out.