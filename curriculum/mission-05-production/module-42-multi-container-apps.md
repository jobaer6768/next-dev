# Module 42: Building Multi-Container Applications

> Mission 5 — Ship Software to Production
> "One container, one process. One repo, one team. One system, one docker-compose.yml."

---

## 1. Why this module matters

In Module 41 you ran three containers with three `docker run` commands and a `docker network create`. That works for a tutorial. It does not work for a real product:

- The flags are unreadable after 30 of them.
- You can't reproduce someone else's setup from a Slack message.
- Healthchecks, dependencies, and ordering are all on you to manage.
- Secrets and ports are scattered across shell history.

This module is the bridge between "I can run containers" and "I can ship a real application." We will take the SaaS from Mission 4 — Next.js frontend, Node/Express API, PostgreSQL database — and produce production-grade Dockerfiles for each, with healthchecks, non-root users, multi-stage builds, and proper network isolation.

In Module 43 we will orchestrate them with Docker Compose. Today we just write the Dockerfiles correctly.

---

## 2. Learning objectives

- Apply the principle: one container, one process.
- Write a multi-stage Dockerfile for a Node.js API with a non-root user.
- Write a multi-stage Dockerfile for a Next.js app with `standalone` output.
- Configure the official Postgres image with environment variables, a volume, and a healthcheck.
- Implement network isolation so the database is not reachable from the public internet.
- Use a bind mount for dev hot-reload of the frontend.
- Explain why every production container should be small, non-root, and healthchecked.

---

## 3. The "one process per container" rule

A container is a process. A process should do one thing. The exceptions are rare and well-known (an init process like `tini` to reap zombies, a sidecar like a log shipper). For application code, the rule is firm:

| Service | Its own container? | Why |
|---|---|---|
| Next.js frontend | Yes | Separate deploy cadence, separate scaling, separate failure domain |
| Node/Express API | Yes | Same |
| PostgreSQL | Yes | Different update cadence; can be a managed service; has its own volume |
| Redis | Yes | Cache is a different concern |
| Nginx | Yes (or use a managed one) | Reverse proxy is a cross-cutting concern |

Putting the API and the frontend in the same container is a common beginner mistake. Don't. They have different dependencies, different ports, different scaling characteristics, and different bug surfaces.

---

## 4. The application we're containerizing

From Mission 4, the SaaS is a small platform with:

- **`web`** — a Next.js 14 app (App Router, server components) at `localhost:3000`.
- **`api`** — a Node 22 / Express 4 service at `localhost:3000` (inside its own container).
- **`db`** — a PostgreSQL 16 database.

The directory layout:

```
saas/
├── docker/
│   ├── api.Dockerfile
│   ├── api.Dockerfile.dev
│   ├── web.Dockerfile
│   └── web.Dockerfile.dev
├── apps/
│   ├── api/
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── server.ts
│   │       ├── db.ts
│   │       └── routes/
│   └── web/
│       ├── package.json
│       ├── next.config.js
│       └── app/
│           └── ...
└── README.md
```

We co-locate the Dockerfiles in a top-level `docker/` directory rather than scattering them. The build context for each image is the apps folder; the Dockerfile is reached via `-f`.

---

## 5. Dockerizing PostgreSQL

You almost never write a Dockerfile for Postgres. The official image is excellent and maintained by the Postgres team. You just run it correctly.

```bash
# The official image, pinned to a specific minor version
postgres:16.4-bookworm
```

### Run config

```bash
$ docker run -d --name db \
    -e POSTGRES_USER=app \
    -e POSTGRES_PASSWORD=dev \
    -e POSTGRES_DB=saas \
    -e PGDATA=/var/lib/postgresql/data/pgdata \
    -v pgdata:/var/lib/postgresql/data \
    --health-cmd="pg_isready -U app -d saas" \
    --health-interval=5s \
    --health-timeout=5s \
    --health-retries=10 \
    --restart unless-stopped \
    postgres:16.4-bookworm
```

What we just did:

- **Pinned the image** to a specific minor (`16.4`) and distro (`bookworm`).
- **Set credentials via env vars.** `POSTGRES_USER` and `POSTGRES_PASSWORD` are required to bootstrap. `POSTGRES_DB` creates a default database.
- **Set `PGDATA` to a subdirectory.** This is required for `pgdata` to be a sub-path of a volume. Avoids a permissions trap on some setups.
- **Mounted a named volume** so data survives container deletion.
- **Healthcheck.** `pg_isready` returns 0 if Postgres is accepting connections, 3 if it's not. Other services can wait for this.
- **Restart policy** so the DB comes back after a daemon restart.

### Network isolation

The most important part: **do not publish Postgres's port to the host.** The default 5432 is for the API, which is on the same Docker network. If you publish 5432, you have made your database reachable from the entire internet (modulo firewall).

```bash
# Wrong: database exposed to the world
$ docker run -d -p 5432:5432 ... postgres:16

# Right: only reachable from other containers on the same network
$ docker run -d ... postgres:16   # no -p flag
```

We will codify this in Module 43 with Compose.

---

## 6. Dockerizing the Node API

The Node service has TypeScript source that needs to be compiled, npm dependencies, and a runtime. Multi-stage builds let us keep the final image small.

### `docker/api.Dockerfile` (production)

```dockerfile
# syntax=docker/dockerfile:1.7
# ---- Stage 1: build ----
FROM node:22.11.0-bookworm-slim AS build
WORKDIR /app

# Install ALL deps (including dev) for the build
COPY package.json package-lock.json ./
RUN npm ci

# Build
COPY tsconfig.json ./
COPY src ./src
RUN npm run build

# Prune dev dependencies for the runtime stage
RUN npm prune --omit=dev

# ---- Stage 2: runtime ----
FROM node:22.11.0-bookworm-slim AS runtime
WORKDIR /app

ENV NODE_ENV=production
ENV PORT=3000

# tini: a tiny init that reaps zombie processes and forwards signals
RUN apt-get update && apt-get install -y --no-install-recommends tini \
    && rm -rf /var/lib/apt/lists/*

# Non-root user
RUN groupadd --system --gid 1001 nodejs \
    && useradd --system --uid 1001 --gid nodejs appuser

# Copy built artifacts and prod node_modules from build stage
COPY --from=build --chown=appuser:nodejs /app/node_modules ./node_modules
COPY --from=build --chown=appuser:nodejs /app/dist ./dist
COPY --from=build --chown=appuser:nodejs /app/package.json ./package.json

USER appuser
EXPOSE 3000

# Healthcheck using node's built-in http (no extra deps)
HEALTHCHECK --interval=10s --timeout=3s --start-period=10s --retries=5 \
    CMD node -e "require('http').get('http://127.0.0.1:3000/health', r => process.exit(r.statusCode === 200 ? 0 : 1)).on('error', () => process.exit(1))"

ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["node", "dist/server.js"]
```

### Why each line is there

- **`AS build` / `AS runtime`.** Two stages. The build stage has TypeScript, dev deps, and the toolchain. The runtime stage has only what's needed to run. Final image is ~150MB instead of ~700MB.
- **`npm ci` instead of `npm install`.** Deterministic install from the lockfile.
- **`npm prune --omit=dev`.** Trims dev dependencies after build, so the runtime stage inherits only prod deps.
- **`tini`.** A 30KB init that handles `SIGTERM` properly and reaps zombie processes. Without it, your container can leak processes on restart.
- **Non-root user.** Defense in depth. If the app is compromised, the attacker is not root inside the container.
- **`HEALTHCHECK`.** Lets Docker (and Compose, and Kubernetes) know if the process is healthy.
- **`ENTRYPOINT ["/usr/bin/tini", "--"]` + `CMD ["node", ...]`.** The exec form ensures signals are forwarded to `node` (not wrapped in a shell that won't forward `SIGTERM`).

### `docker/api.Dockerfile.dev` (development with hot reload)

```dockerfile
FROM node:22.11.0-bookworm-slim
WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends tini \
    && rm -rf /var/lib/apt/lists/*

COPY package.json package-lock.json ./
RUN npm install

# Source is bind-mounted in dev, so we don't COPY it.

ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["npx", "nodemon", "--watch", "src", "--ext", "ts,json", "src/server.ts"]
```

This is faster to build (no TypeScript compile) and lets us bind-mount `src/` in dev for hot reload. We will use this in the Makefile later.

### `.dockerignore` for the API

```gitignore
node_modules
dist
coverage
.env
.env.*
!.env.example
.git
.vscode
.idea
*.log
README.md
```

---

## 7. Dockerizing the Next.js app

Next.js has a clever **standalone output mode** that produces a minimal server bundle, dropping `node_modules` and the framework internals you don't need. Combined with a multi-stage build, you get a tiny image.

### `next.config.js` (required to enable standalone)

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: "standalone",
  experimental: {
    outputFileTracingIncludes: {
      "/": ["./node_modules/@my-ui-kit/**/*"],
    },
  },
};
module.exports = nextConfig;
```

### `docker/web.Dockerfile` (production)

```dockerfile
# syntax=docker/dockerfile:1.7
# ---- Stage 1: deps ----
FROM node:22.11.0-bookworm-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# ---- Stage 2: build ----
FROM node:22.11.0-bookworm-slim AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# ---- Stage 3: runtime ----
FROM node:22.11.0-bookworm-slim AS runtime
WORKDIR /app

ENV NODE_ENV=production
ENV PORT=3000
ENV HOSTNAME=0.0.0.0

RUN apt-get update && apt-get install -y --no-install-recommends tini \
    && rm -rf /var/lib/apt/lists/*

RUN groupadd --system --gid 1001 nodejs \
    && useradd --system --uid 1001 --gid nodejs appuser

# Copy only the standalone output, static files, and public
COPY --from=build --chown=appuser:nodejs /app/.next/standalone ./
COPY --from=build --chown=appuser:nodejs /app/.next/static ./.next/static
COPY --from=app --chown=appuser:nodejs /app/public ./public

USER appuser
EXPOSE 3000

HEALTHCHECK --interval=10s --timeout=3s --start-period=15s --retries=5 \
    CMD node -e "require('http').get('http://127.0.0.1:3000/api/health', r => process.exit(r.statusCode === 200 ? 0 : 1)).on('error', () => process.exit(1))"

ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["node", "server.js"]
```

What is the `standalone` output? Next.js traces your code's actual dependencies and produces a `.next/standalone/` folder with a minimal `server.js` and only the files needed to run it. No `node_modules`, no source. The final image can be under 150MB.

### `docker/web.Dockerfile.dev` (development with hot reload)

```dockerfile
FROM node:22.11.0-bookworm-slim
WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends tini \
    && rm -rf /var/lib/apt/lists/*

COPY package.json package-lock.json ./
RUN npm install

ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["npm", "run", "dev"]
```

---

## 8. Running the dev stack with bind mounts

```bash
# Create a network
$ docker network create saas-net

# Postgres
$ docker run -d --name db \
    --network saas-net \
    -e POSTGRES_USER=app \
    -e POSTGRES_PASSWORD=dev \
    -e POSTGRES_DB=saas \
    -v pgdata:/var/lib/postgresql/data \
    postgres:16.4-bookworm

# API (dev mode, with hot reload)
$ docker run -d --name api \
    --network saas-net \
    -e DATABASE_URL=postgres://app:dev@db:5432/saas \
    -e NODE_ENV=development \
    -v "$(pwd)/apps/api/src:/app/src" \
    -v "$(pwd)/apps/api/package.json:/app/package.json" \
    -v api-node-modules:/app/node_modules \
    -p 127.0.0.1:3001:3000 \
    -f docker/api.Dockerfile.dev \
    apps-api:dev

# Frontend (dev mode)
$ docker run -d --name web \
    --network saas-net \
    -e NEXT_PUBLIC_API_URL=http://api:3000 \
    -v "$(pwd)/apps/web:/app" \
    -v web-node-modules:/app/node_modules \
    -p 127.0.0.1:3000:3000 \
    -f docker/web.Dockerfile.dev \
    apps-web:dev
```

Notice:

- The API references Postgres as `db` (DNS) and is referenced by the frontend as `api` (DNS).
- The frontend's `NEXT_PUBLIC_API_URL` is the in-cluster hostname, not `localhost`. This is the correct mental model: the browser will hit your reverse proxy, which will forward to the API, but the in-container communication happens over the Docker network.
- Bind mounts for `src` and `package.json` give you hot reload.
- Anonymous-but-named volumes `api-node-modules` and `web-node-modules` keep the host's modules from clobbering the container's.

This works, but typing it every time is painful. Module 43 replaces it with Compose.

---

## 9. Security layer for the database

Let's add the network isolation we discussed. We want Postgres on a private network that only the API can reach.

```bash
$ docker network create --internal backend
# --internal means no egress to the internet
```

Wait, we still need the API to reach the internet (for Stripe). So we use two networks:

- `backend` — Postgres, API (internal or not; doesn't matter).
- `frontend` — API, Web (where the API bridges).

```bash
$ docker network create backend
$ docker network create frontend

# Postgres only on backend
$ docker run -d --name db --network backend -v pgdata:/var/lib/postgresql/data ... postgres:16

# API on both networks
$ docker run -d --name api \
    --network backend \
    ... saas-api:1.0.0
$ docker network connect frontend api

# Frontend only on frontend
$ docker run -d --name web --network frontend -p 3000:3000 ... saas-web:1.0.0
```

The frontend can reach the API at `api:3000` (DNS over `frontend`). The API can reach the database at `db:5432` (DNS over `backend`). The frontend **cannot** reach the database at all, because it's not on the `backend` network. This is network-level defense in depth.

We will codify this in Module 43. Doing it manually is possible but error-prone.

---

## 10. Module summary

| Concern | Production pattern |
|---|---|
| One process per container | Yes, always |
| Image size | Multi-stage builds, slim base images, prune dev deps |
| User | Non-root with explicit UID/GID |
| Init process | `tini` to reap zombies and forward signals |
| Healthcheck | `HEALTHCHECK` directive in the Dockerfile |
| Signals | `ENTRYPOINT [tini, --]` so SIGTERM reaches the app |
| Database | Official image, named volume, healthcheck, no published port |
| Network isolation | Backend / frontend networks; database only on backend |
| Dev mode | Separate `*.Dockerfile.dev` with bind mounts and `nodemon` / `next dev` |
| Config | Env vars, not baked into the image |
| Secrets | `.env` files, never committed, never copied into image |

---

## 11. Common mistakes

- **One big container with everything inside.** Coupled lifecycle, security holes, slow deploys. Don't.
- **Running as root in production.** The default. Always override.
- **`CMD node server.js` without `tini`.** Signals don't reach your process; `docker stop` waits 10 seconds and SIGKILLs. Your in-flight requests are dropped.
- **`COPY . .` in the build stage when you only need `dist`.** Bloats intermediate stages, slows builds.
- **Postgres with `-p 5432:5432`.** Don't expose your database to the host's network. The reverse proxy is the only thing that should be public.
- **Bind mounting your host `node_modules` into the container.** Architecture mismatches and weird bugs. Use an anonymous volume for `node_modules`.
- **Hardcoding `localhost` in cross-container URLs.** Use the service name. DNS will resolve it.

---

## 12. Exercise

**Goal:** Build production-quality Docker images for all three services in the SaaS.

**Steps:**

1. Create the directory structure (`docker/`, `apps/api/`, `apps/web/`). Use your real SaaS code from Mission 4, or the sample code from earlier modules.
2. Write `docker/api.Dockerfile` (multi-stage, non-root, tini, healthcheck) and `docker/api.Dockerfile.dev`.
3. Write `docker/web.Dockerfile` (Next.js standalone output, non-root, tini) and `docker/web.Dockerfile.dev`.
4. Write `.dockerignore` for both.
5. Build the production images:
   ```bash
   $ docker build -f docker/api.Dockerfile -t saas-api:1.0.0 apps/api
   $ docker build -f docker/web.Dockerfile -t saas-web:1.0.0 apps/web
   ```
6. Inspect image sizes: `docker images`. Aim for under 200MB for each.
7. Inspect layers: `docker history saas-api:1.0.0`. Confirm the largest layer is not a `node_modules` copy.
8. Run the dev stack with bind mounts as in section 8. Verify hot reload works.

**Deliverable:** A markdown file with:
- All four Dockerfiles.
- The output of `docker images` after building.
- A `docker history` output annotated with what each layer does.

---

## 13. AI Industry Note (2026)

In 2026, **container security is AI security.** The major LLM APIs (OpenAI, Anthropic, Google) are accessed over HTTPS, so the egress story hasn't changed. But local LLM stacks (Ollama, vLLM, llama.cpp) often run in containers, and they pull multi-GB model weights. Patterns to know:

- **Mount model weights as a volume, not as a `COPY`.** A 70B safetensors file is ~140GB. You do not want that in your image layers.
- **Use a separate image for the model server** (often `ghcr.io/ggerganov/llama.cpp:server-cuda` or `vllm/vllm-openai:latest`) and connect to it from your application container over a private network.
- **Treat GPU memory like RAM.** Containers do not isolate GPU memory by default. Be deliberate about which containers can see which GPUs (`docker run --gpus '"device=0,1"' ...`).
- **Use `docker scout cves` or `trivy image` to scan your images for known vulnerabilities.** In 2026 most enterprise CI pipelines fail builds on critical CVEs. We'll do this in Module 44.

The skill you're building here — production Dockerfiles for a small distributed system — maps 1:1 onto the way every serious AI product is deployed.

---

## 14. Looking ahead

In Module 43 we replace every command in this module with a single `docker-compose.yml` file. We also add a `Makefile` so your day-to-day commands become `make up`, `make down`, `make logs`, `make migrate`. This is the last "by hand" lesson; after it, your work is declarative and reproducible.
