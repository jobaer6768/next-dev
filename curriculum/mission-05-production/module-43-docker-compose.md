# Module 43: Multi-Container Orchestration with Docker Compose and Makefile Automation

> Mission 5 — Ship Software to Production
> "Declarative infrastructure is the difference between a system you run and a system you ship."

---

## 1. Why this module matters

In Module 42 we built a real three-service application. We had to type — or paste — twelve `docker run` commands, four network management commands, and we had to remember which env var went where. If a new team member joins, they get a Slack message and a copy-paste ritual.

**Docker Compose replaces all of that with one file.** A `docker-compose.yml` declares every service, network, and volume your application needs. `docker compose up` brings the whole system up. `docker compose down` tears it down. The new engineer runs `make up` and is productive in 60 seconds.

In this module you will:

- Write a complete `docker-compose.yml` for the SaaS (Postgres, API, frontend).
- Use `depends_on` with healthchecks to control startup order.
- Implement dev/prod separation with override files.
- Write a `Makefile` that wraps the most common Compose commands.
- Understand `docker compose` vs the legacy `docker-compose` (Python) tool.

---

## 2. Learning objectives

- Explain what Docker Compose is and when to use it (vs raw `docker run` and vs Kubernetes).
- Author a multi-service `docker-compose.yml` with `services`, `networks`, `volumes`.
- Use `depends_on` with `condition: service_healthy` for proper startup order.
- Inject environment variables and `.env` files.
- Use `profiles` to distinguish dev vs prod targets.
- Override Compose files for environment-specific behavior.
- Author a `Makefile` that exposes the most common operations.
- Run database migrations and seeds as part of the orchestration.

---

## 3. Compose in one sentence

**Docker Compose is a declarative YAML file that describes a multi-container application, plus a CLI that brings it up, tears it down, and inspects it.**

The CLI is `docker compose` (with a space, no hyphen). The legacy `docker-compose` (hyphen, Python script) is deprecated as of 2023 and removed from Docker Desktop. If you see old blog posts using the hyphen, translate them.

---

## 4. The full `docker-compose.yml`

Here is the file that orchestrates the SaaS. Place it at the repo root.

```yaml
# docker-compose.yml
# Run with:
#   docker compose --profile dev up
#   docker compose --profile prod up

name: saas

x-common-env: &common-env
  NODE_ENV: ${NODE_ENV:-production}
  LOG_LEVEL: ${LOG_LEVEL:-info}

services:
  # ---------- Database ----------
  db:
    image: postgres:16.4-bookworm
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-app}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-dev}
      POSTGRES_DB: ${POSTGRES_DB:-saas}
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-app} -d ${POSTGRES_DB:-saas}"]
      interval: 5s
      timeout: 5s
      retries: 10
    networks:
      - backend

  # ---------- API ----------
  api:
    build:
      context: ./apps/api
      dockerfile: ../../docker/api.Dockerfile
    image: saas-api:1.0.0
    restart: unless-stopped
    profiles: ["dev", "prod"]
    environment:
      <<: *common-env
      PORT: 3000
      DATABASE_URL: postgres://${POSTGRES_USER:-app}:${POSTGRES_PASSWORD:-dev}@db:5432/${POSTGRES_DB:-saas}
      JWT_SECRET: ${JWT_SECRET:?JWT_SECRET must be set}
      STRIPE_SECRET_KEY: ${STRIPE_SECRET_KEY:-}
    depends_on:
      db:
        condition: service_healthy
    networks:
      - backend
      - frontend
    healthcheck:
      test: ["CMD-SHELL", "node -e \"require('http').get('http://127.0.0.1:3000/health', r => process.exit(r.statusCode === 200 ? 0 : 1)).on('error', () => process.exit(1))\""]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 10s

  # ---------- Frontend ----------
  web:
    build:
      context: ./apps/web
      dockerfile: ../../docker/web.Dockerfile
    image: saas-web:1.0.0
    restart: unless-stopped
    profiles: ["dev", "prod"]
    environment:
      <<: *common-env
      PORT: 3000
      HOSTNAME: 0.0.0.0
      NEXT_PUBLIC_API_URL: ${NEXT_PUBLIC_API_URL:-http://api:3000}
    depends_on:
      api:
        condition: service_healthy
    networks:
      - frontend

  # ---------- Dev-only overrides ----------
  api-dev:
    build:
      context: ./apps/api
      dockerfile: ../../docker/api.Dockerfile.dev
    image: saas-api:dev
    profiles: ["dev"]
    environment:
      NODE_ENV: development
      DATABASE_URL: postgres://app:dev@db:5432/saas
    volumes:
      - ./apps/api/src:/app/src
      - ./apps/api/package.json:/app/package.json
      - api-node-modules:/app/node_modules
    depends_on:
      db:
        condition: service_healthy
    networks:
      - backend
      - frontend

  web-dev:
    build:
      context: ./apps/web
      dockerfile: ../../docker/web.Dockerfile.dev
    image: saas-web:dev
    profiles: ["dev"]
    environment:
      NODE_ENV: development
      NEXT_PUBLIC_API_URL: http://api:3000
    volumes:
      - ./apps/web:/app
      - web-node-modules:/app/node_modules
      - ./apps/web/.next:/app/.next
    depends_on:
      api-dev:
        condition: service_started
    networks:
      - frontend

  # ---------- One-shot migrations ----------
  migrate:
    image: saas-api:1.0.0
    profiles: ["tools"]
    command: ["node", "dist/migrate.js"]
    environment:
      DATABASE_URL: postgres://${POSTGRES_USER:-app}:${POSTGRES_PASSWORD:-dev}@db:5432/${POSTGRES_DB:-saas}
    depends_on:
      db:
        condition: service_healthy
    networks:
      - backend
    restart: "no"

  seed:
    image: saas-api:1.0.0
    profiles: ["tools"]
    command: ["node", "dist/seed.js"]
    environment:
      DATABASE_URL: postgres://${POSTGRES_USER:-app}:${POSTGRES_PASSWORD:-dev}@db:5432/${POSTGRES_DB:-saas}
    depends_on:
      migrate:
        condition: service_completed_successfully
    networks:
      - backend
    restart: "no"

networks:
  backend:
    driver: bridge
  frontend:
    driver: bridge

volumes:
  pgdata:
  api-node-modules:
  web-node-modules:
```

This file is doing a lot. Let me break it down.

---

## 5. Anatomy of the file

### `services`

Each top-level key under `services` is one container. We have seven: `db`, `api`, `web`, `api-dev`, `web-dev`, `migrate`, `seed`. Two of them (`api-dev` and `web-dev`) are dev-only; they share the production names with `profiles`.

### `x-common-env` and YAML anchors

```yaml
x-common-env: &common-env
  NODE_ENV: ${NODE_ENV:-production}
  LOG_LEVEL: ${LOG_LEVEL:-info}
```

The `x-` prefix marks Compose-internal variables, not services. `&common-env` is a YAML anchor. `<<: *common-env` later in the file merges those keys. Result: both the API and web share the same `NODE_ENV` and `LOG_LEVEL` defaults.

### `image` and `build`

```yaml
build:
  context: ./apps/api
  dockerfile: ../../docker/api.Dockerfile
image: saas-api:1.0.0
```

`build` says how to build the image. `image` says what to name the result. After `docker compose build`, the image is named `saas-api:1.0.0` and can be referenced elsewhere in the file (e.g. by `migrate` and `seed`).

### `profiles`

```yaml
profiles: ["dev", "prod"]
```

A service is only started when its profile is active. Profiles let you have one Compose file for everything.

- `docker compose --profile dev up` → starts `db`, `api-dev`, `web-dev`.
- `docker compose --profile prod up` → starts `db`, `api`, `web`.
- `docker compose --profile tools up migrate` → runs the migration job and exits.

### `depends_on` with healthchecks

```yaml
depends_on:
  db:
    condition: service_healthy
```

This is the killer feature. Without it, the API starts before Postgres is ready, and crashes. With it, Compose waits for the `db` healthcheck to succeed before starting the API. Three conditions:

- `service_started` — started, no healthcheck check.
- `service_healthy` — healthcheck passes (requires a `healthcheck` block on the dependency).
- `service_completed_successfully` — exited with code 0 (for one-shot jobs).

### `networks`

```yaml
networks:
  - backend
  - frontend
```

Each service joins one or more networks. We deliberately put the database on `backend` only. The frontend is on `frontend` only. The API bridges them. This is the network isolation we described in Module 42.

### `volumes`

```yaml
volumes:
  - pgdata:/var/lib/postgresql/data
```

A named volume for Postgres data. Survives `docker compose down` (it does not delete named volumes by default). For dev hot reload, anonymous volumes like `api-node-modules:/app/node_modules` keep host and container modules separated.

### `healthcheck`

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U app -d saas"]
  interval: 5s
  timeout: 5s
  retries: 10
```

Same as the `HEALTHCHECK` directive in a Dockerfile, but in Compose. Useful when you don't control the Dockerfile (e.g. official images).

### Environment variable interpolation

```yaml
JWT_SECRET: ${JWT_SECRET:?JWT_SECRET must be set}
DATABASE_URL: postgres://${POSTGRES_USER:-app}:${POSTGRES_PASSWORD:-dev}@db:5432/${POSTGRES_DB:-saas}
```

- `${VAR}` — required, fails if unset.
- `${VAR:-default}` — falls back to `default` if unset.
- `${VAR:?error message}` — fails with a clear message if unset.

This is your first line of defense against missing secrets in production.

---

## 6. The `.env` file

Compose reads a `.env` file at the repo root by default. **Do not commit it.**

```bash
# .env (committed; safe to share)
NODE_ENV=development
POSTGRES_USER=app
POSTGRES_DB=saas
LOG_LEVEL=debug
NEXT_PUBLIC_API_URL=http://localhost:3001

# .env.local (gitignored; your personal overrides)
POSTGRES_PASSWORD=hunter2
JWT_SECRET=dev-only-not-for-prod
STRIPE_SECRET_KEY=sk_test_...
```

```gitignore
# .gitignore
.env
.env.local
.env.*.local
!.env.example
```

Always commit a `.env.example` with placeholders so newcomers know what to set.

---

## 7. Day-to-day commands

```bash
# Start the dev stack
$ docker compose --profile dev up

# In the background
$ docker compose --profile dev up -d

# Tail logs from everything
$ docker compose --profile dev logs -f

# Just the API
$ docker compose --profile dev logs -f api-dev

# Stop everything (keeps volumes)
$ docker compose down

# Stop + remove volumes (DESTRUCTIVE: drops DB data)
$ docker compose down -v

# Rebuild images after a Dockerfile change
$ docker compose --profile dev build

# Rebuild and restart
$ docker compose --profile dev up -d --build

# Run a one-off command in a service's container
$ docker compose --profile dev exec api-dev sh
$ docker compose --profile dev exec db psql -U app -d saas

# Run a one-off command in a fresh container with the service's env+volumes
$ docker compose --profile dev run --rm api-dev npx prisma migrate deploy

# List running services
$ docker compose ps

# Validate the file
$ docker compose config
```

---

## 8. The Makefile

The Makefile is the developer-friendly wrapper. It collapses the Compose invocations to memorable verbs.

```makefile
# Makefile
# Run `make help` to list available targets.

SHELL := /bin/bash
.DEFAULT_GOAL := help
COMPOSE := docker compose
PROFILE_DEV := --profile dev
PROFILE_PROD := --profile prod
PROFILE_TOOLS := --profile tools

# Load .env if it exists
ifneq (,$(wildcard .env))
include .env
export
endif

.PHONY: help
help: ## Show this help
	@awk 'BEGIN {FS = ":.*##"; printf "Usage: make \033[36m<target>\033[0m\n\nTargets:\n"} \
		/^[a-zA-Z_-]+:.*?##/ { printf "  \033[36m%-15s\033[0m %s\n", $$1, $$2 }' \
		$(MAKEFILE_LIST)

.PHONY: up
up: ## Start the dev stack in the background
	$(COMPOSE) $(PROFILE_DEV) up -d --build

.PHONY: down
down: ## Stop the dev stack (keep volumes)
	$(COMPOSE) down

.PHONY: nuke
nuke: ## Stop the dev stack AND delete all volumes (DESTRUCTIVE)
	$(COMPOSE) down -v

.PHONY: logs
logs: ## Tail logs from all services
	$(COMPOSE) $(PROFILE_DEV) logs -f --tail=100

.PHONY: logs-api
logs-api: ## Tail logs from the API only
	$(COMPOSE) $(PROFILE_DEV) logs -f --tail=100 api-dev

.PHONY: logs-web
logs-web: ## Tail logs from the web only
	$(COMPOSE) $(PROFILE_DEV) logs -f --tail=100 web-dev

.PHONY: logs-db
logs-db: ## Tail logs from the DB only
	$(COMPOSE) $(PROFILE_DEV) logs -f --tail=100 db

.PHONY: ps
ps: ## List running services
	$(COMPOSE) $(PROFILE_DEV) ps

.PHONY: shell-api
shell-api: ## Open a shell in the API container
	$(COMPOSE) $(PROFILE_DEV) exec api-dev sh

.PHONY: shell-db
shell-db: ## Open psql in the DB
	$(COMPOSE) $(PROFILE_DEV) exec db psql -U $(POSTGRES_USER) -d $(POSTGRES_DB)

.PHONY: migrate
migrate: ## Run database migrations
	$(COMPOSE) $(PROFILE_TOOLS) up --build migrate

.PHONY: seed
seed: migrate ## Run database seeds
	$(COMPOSE) $(PROFILE_TOOLS) up --build seed

.PHONY: build
build: ## Build all images
	$(COMPOSE) $(PROFILE_DEV) build

.PHONY: pull
pull: ## Pull base images
	$(COMPOSE) $(PROFILE_DEV) pull

.PHONY: prod-up
prod-up: ## Start the production profile
	$(COMPOSE) $(PROFILE_PROD) up -d --build

.PHONY: prod-down
prod-down: ## Stop the production profile
	$(COMPOSE) $(PROFILE_PROD) down

.PHONY: test
test: ## Run tests in the API container
	$(COMPOSE) $(PROFILE_DEV) exec api-dev npm test

.PHONY: lint
lint: ## Lint everything
	$(COMPOSE) $(PROFILE_DEV) exec api-dev npm run lint
	$(COMPOSE) $(PROFILE_DEV) exec web-dev npm run lint
```

### Using it

```bash
$ make help
Usage: make <target>

Targets:
  up              Start the dev stack in the background
  down            Stop the dev stack (keep volumes)
  nuke            Stop the dev stack AND delete all volumes (DESTRUCTIVE)
  logs            Tail logs from all services
  ...
$ make up
$ make logs
$ make migrate
$ make shell-db
```

The Makefile is also the foundation for production deploy scripts in Module 44. We will add `make deploy` there.

---

## 9. Database migrations and seeds as part of the workflow

Migrations are tricky in containers because:

1. The DB needs to be healthy before migrations run.
2. The migration runner needs network access to the DB.
3. The migration should run exactly once, not in a loop.

Our pattern in Compose: a `migrate` service in the `tools` profile, with `restart: "no"` so it exits when done, and `depends_on: db: condition: service_healthy`. The migration script reads the same `DATABASE_URL` as the API.

```ts
// apps/api/src/migrate.ts
import { Client } from "pg";
import { readdirSync, readFileSync } from "node:fs";
import { join } from "node:path";

async function main() {
  const client = new Client({ connectionString: process.env.DATABASE_URL });
  await client.connect();
  await client.query(`
    CREATE TABLE IF NOT EXISTS _migrations (
      id SERIAL PRIMARY KEY,
      name TEXT UNIQUE NOT NULL,
      applied_at TIMESTAMPTZ NOT NULL DEFAULT now()
    )
  `);

  const dir = join(__dirname, "migrations");
  const files = readdirSync(dir).sort();
  for (const file of files) {
    const { rowCount } = await client.query(
      "SELECT 1 FROM _migrations WHERE name = $1",
      [file]
    );
    if (rowCount && rowCount > 0) {
      console.log(`✓ ${file} (already applied)`);
      continue;
    }
    const sql = readFileSync(join(dir, file), "utf-8");
    await client.query("BEGIN");
    try {
      await client.query(sql);
      await client.query(
        "INSERT INTO _migrations (name) VALUES ($1)",
        [file]
      );
      await client.query("COMMIT");
      console.log(`✓ ${file}`);
    } catch (e) {
      await client.query("ROLLBACK");
      throw e;
    }
  }
  await client.end();
}

main().catch((e) => {
  console.error(e);
  process.exit(1);
});
```

Run it:

```bash
$ make migrate
# Equivalent to:
# docker compose --profile tools up --build migrate
```

The `seed` service depends on `migrate` with `condition: service_completed_successfully`, so seeds never run on an un-migrated DB.

---

## 10. Override files: `docker-compose.override.yml`

By default, Compose reads `docker-compose.yml` **and** `docker-compose.override.yml` together, with the override taking precedence. This is how you separate "what the system is" from "what's different on my machine."

`docker-compose.override.yml` (gitignored, personal):

```yaml
name: saas

services:
  api-dev:
    environment:
      LOG_LEVEL: debug
      DEBUG: "true"
    ports:
      - "127.0.0.1:9229:9229"   # Node debugger

  web-dev:
    environment:
      NEXT_PUBLIC_API_URL: http://api.localtest.me:3001
```

You don't need to commit this file. Compose will silently use it if it exists. To ignore it globally, set `COMPOSE_IGNORE_ORPHANS=1` or just not have the file.

---

## 11. Common mistakes

- **Using `docker-compose` (hyphen) instead of `docker compose` (space).** The hyphen form is dead.
- **Not pinning image tags.** `postgres:latest` will eventually become Postgres 17 and break your schema assumptions.
- **Forgetting `profiles`.** Running `docker compose up` without a profile may start nothing if all services have profiles. Pick a sensible default or set one without a profile (e.g. `db` is always-on).
- **Committing `.env` to the repo.** Rotate the leaked secrets immediately. Then `.gitignore` it.
- **Not using healthchecks on dependencies.** You'll spend half an hour debugging "API won't connect to DB" when the real problem is startup order.
- **Using `docker compose down -v` casually.** This deletes named volumes. Your database goes with it. Use it only when you mean it.
- **Running a one-shot job with `restart: always`.** It will keep running forever, re-doing its work.
- **Exposing Postgres to the host** with `ports: ["5432:5432"]`. Don't. Database stays on the internal `backend` network.

---

## 12. Exercise

**Goal:** Replace your manual `docker run` setup with Compose + Makefile, and prove it works.

**Steps:**

1. Take the SaaS from Mission 4. Build the directory layout described in section 4.
2. Write the full `docker-compose.yml`. Use profiles: `db` is always-on; `api`, `web` are `prod`; `api-dev`, `web-dev` are `dev`; `migrate`, `seed` are `tools`.
3. Write the `Makefile` from section 8.
5. Create `.env` (local, gitignored) and `.env.example` (committed).
6. Run:
   ```bash
   $ make up
   $ make logs
   $ make migrate
   $ make seed
   ```
7. Verify the app works end-to-end in your browser.
8. Edit a file in `apps/api/src/`. Confirm hot reload.
9. Run `make down`, then `make up`. Confirm volumes are preserved (data still there).
10. Run `make nuke`. Confirm everything is gone (data deleted, services stopped).

**Deliverable:** A working repo where `make up && make migrate && make seed` brings the entire system up and is reproducible by anyone with Docker.

---

## 13. AI Industry Note (2026)

`docker-compose.yml` is the unit of reproducibility for AI side projects in 2026. The standard "AI starter" repo you see on GitHub has:

- A `docker-compose.yml` with a vector DB, an embedding service, an LLM server, and the app.
- A `Makefile` with `make up`, `make logs`, `make shell-llm`.
- A `.env.example` that lists every external API key.

If you can write this file, you can ship a RAG product to a paying customer in a week.

In production at scale, you graduate from Compose to **Kubernetes** (or a managed equivalent: ECS, Cloud Run, Fly.io, Railway, Render). The Compose file is a stepping stone, not a destination. But the patterns you learned here — networks, volumes, healthchecks, profiles — are the same concepts in different syntax. A `Deployment` is a service. A `PersistentVolumeClaim` is a volume. A `Service` is a load balancer that fronts pods. Compose teaches you the vocabulary that Kubernetes requires.

---

## 14. Looking ahead

In Module 44 we take this same `docker-compose.yml`, add an Nginx reverse proxy, and deploy the whole thing to a real Virtual Private Server. You will also write a GitHub Actions workflow that builds the images on every push and deploys on every tag. By the end of the mission, your SaaS is on the internet for real.