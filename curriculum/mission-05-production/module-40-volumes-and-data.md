# Module 40: Docker Volumes and Data Management

> Mission 5 — Ship Software to Production
> "Containers are cattle, not pets. Except for your database — that's a special kind of cow."

---

## 1. Why this module matters

In Module 39 we built a container from a Node app and ran it. It worked. It felt like magic. Then we restarted the container, and... everything we wrote to disk inside it was gone.

That is by design. A container's writable layer is **ephemeral**. When the container is deleted, that layer goes with it. This is excellent for stateless web servers and absolutely catastrophic for databases.

This module is about how to keep the things that need to survive: database files, user uploads, generated artifacts, secrets. We will cover three mechanisms — the writable layer, **volumes**, and **bind mounts** — and you will learn when to use which.

By the end you can:

- Explain why the writable container layer is not a place to store data.
- Create and manage Docker volumes.
- Use bind mounts for development hot-reload.
- Decide between a volume and a bind mount in any given situation.
- Pass configuration to containers via environment variables and `.env` files.
- Use `.dockerignore` to keep your images small and your secrets out.

---

## 2. Learning objectives

- Identify the four categories of data in a containerized app.
- Explain the writable container layer and its limitations.
- Create, list, inspect, and remove Docker volumes.
- Create and use bind mounts for live code reload.
- Use environment variables with `docker run -e` and `--env-file`.
- Write a proper `.dockerignore`.
- Combine volumes and bind mounts to solve a real dev workflow.

---

## 3. The four categories of data

Any application has data. In a containerized app, sort it into four buckets:

| Category | Examples | Lifespan | Where to put it |
|---|---|---|---|
| **Application code** | `src/`, `dist/`, `package.json` | Lives in the image | Baked into the image at build time |
| **User-generated data** | Uploads, user profiles, content | Forever | Volume (or external DB) |
| **Generated data** | Logs, caches, build artifacts | Varies | Volume, or log driver |
| **Secrets & config** | API keys, DB URLs, feature flags | App lifetime | Env vars, secret store, or volume |

Code is immutable and goes in the image. Everything else needs a strategy.

---

## 4. The writable container layer (and why it's a trap)

When a container runs, Docker adds a thin writable layer on top of the read-only image layers. Any file change, new file, or deletion inside the running container lives here:

```
┌────────────────────────────┐
│ Writable container layer   │  <-- per-container, dies with container
├────────────────────────────┤
│ Image layers (read-only)   │
└────────────────────────────┘
```

You can verify this is real. Run a container, write a file, delete the container, run a new one from the same image — the file is gone.

```bash
$ docker run -it --rm alpine sh -c 'echo "hello" > /tmp/data.txt && cat /tmp/data.txt'
hello
$ docker run -it --rm alpine sh -c 'cat /tmp/data.txt'
cat: can't open '/tmp/data.txt': No such file or directory
```

### Why this is a problem

- **No persistence.** Restart = data loss.
- **Coupled to the container lifecycle.** Can't move data to a new container easily.
- **Performance.** Writes go through a copy-on-write filesystem driver (overlay2). Faster than it used to be, but still not as fast as a native filesystem.
- **No sharing.** Two containers can't share the writable layer.
- **Larger containers.** Big writes = a big writable layer = slow commits and pushes.

The writable layer is fine for short-lived state (a request being processed, a build artifact mid-compile). It is not fine for anything you want to keep.

---

## 5. Docker volumes (the right answer for most cases)

A **volume** is a directory stored in a location managed by Docker (typically `/var/lib/docker/volumes/` on Linux). It is:

- Created and managed by Docker.
- Independent of any container's lifecycle.
- Mountable into one or many containers simultaneously.
- Backable up, restored, and migrated with simple commands.
- Faster and more portable than bind mounts on most platforms.

### Create and use a volume

```bash
# Create a named volume
$ docker volume create pgdata

# Run postgres with the volume mounted
$ docker run -d --name pg \
    -e POSTGRES_PASSWORD=dev \
    -v pgdata:/var/lib/postgresql/data \
    postgres:16

# Inspect it
$ docker volume inspect pgdata
[
  {
    "Name": "pgdata",
    "Driver": "local",
    "Mountpoint": "/var/lib/docker/volumes/pgdata/_data",
    ...
  }
]

# List volumes
$ docker volume ls

# Remove a volume (only if no container is using it)
$ docker volume rm pgdata

# Clean up unused volumes
$ docker volume prune
```

### Volume drivers

The default driver is `local`, which stores data on the host. There are also third-party drivers for NFS, AWS EBS, Azure Files, CIFS, and more. We won't use them in this mission, but know they exist.

### Anonymous vs named volumes

If you mount without a name, Docker creates an anonymous volume:

```bash
$ docker run -d -v /var/lib/postgresql/data postgres:16
```

The downside: anonymous volumes get random hashes, are hard to reference, and often end up orphaned. **Always name your volumes** in real projects.

---

## 6. Bind mounts (the right answer for dev hot-reload)

A **bind mount** mounts a specific path from your host into the container. The container sees your host's files in real time.

```bash
$ docker run -d -p 3000:3000 \
    -v "$(pwd)/src:/app/src" \
    saas-api:1.0.0
```

Now when you edit `/path/to/api/src/server.ts` on your host, the change appears immediately at `/app/src/server.ts` inside the container. Combined with `tsc --watch` or `nodemon`, you get **hot reload**: edit code, container restarts, browser refreshes.

### Why this is gold for development

- Edit code on your host with your editor, your extensions, your muscle memory.
- Container runs the actual production-like runtime (Node 22, real Postgres, real Nginx).
- No "sync to container" step. No "rebuild image" step. Iterate in seconds.

### The trade-off

Bind mounts couple the container to your host's filesystem. They are:

- **Not portable.** A `docker run -v /Users/alice/code:/app` command will not work on a Windows host. The host path is host-specific.
- **Slower on macOS/Windows.** The path has to be synced through the VM's filesystem shim (virtiofs in 2026, still not free).
- **Risky in production.** Mounting host paths into production containers couples them to the host. Use named volumes instead.

**Rule of thumb:** bind mounts for development, named volumes for production data.

---

## 7. The volume vs bind mount decision tree

```
Need to persist data?
  ├─ Yes
  │   ├─ It's dev code that should hot-reload → bind mount
  │   ├─ It's a database / file upload / cache → named volume
  │   └─ It's both? Use both: one bind mount for code, one volume for data
  └─ No
      └─ Don't mount anything; container's writable layer is fine
```

### A classic combo: code via bind mount, data via volume

```bash
$ docker run -d -p 3000:3000 \
    -v "$(pwd)/src:/app/src" \      # code: hot reload
    -v uploads:/app/uploads \       # user uploads: persistent
    saas-api:1.0.0
```

`./src` on the host is bind-mounted; `uploads` is a named volume. The container can be deleted and recreated without losing user uploads, but a code edit shows up instantly.

---

## 8. Environment variables and `.env` files

Configuration that varies between environments (DB URLs, API keys, log levels) should never be baked into the image. Pass it in at runtime.

### Inline env vars

```bash
$ docker run -d -p 3000:3000 \
    -e NODE_ENV=production \
    -e DATABASE_URL=postgres://user:pass@db:5432/app \
    -e LOG_LEVEL=info \
    saas-api:1.0.0
```

### `--env-file` for many variables

Create a `.env` file (we will look at `.dockerignore` next to make sure this does not get baked into the image):

```bash
# .env (NEVER commit this)
NODE_ENV=production
DATABASE_URL=postgres://app:secret@db:5432/saas
JWT_SECRET=change-me-in-prod
PORT=3000
```

```bash
$ docker run -d -p 3000:3000 --env-file .env saas-api:1.0.0
```

Inside the container, `process.env.DATABASE_URL` is set. The Node app reads it on startup, the same way it does in development.

### Common patterns

- **`.env`** for local development (committed to repo as `.env.example`).
- **`production.env`** passed via `--env-file` from your CI/CD system.
- **Docker secrets** for highly sensitive data (we will not use in this mission; relevant when you adopt Swarm).
- **External secret store** (AWS Secrets Manager, HashiCorp Vault) for the real production answer. We will mention this in Module 44 but not implement it.

### Defaulting in the Dockerfile

Use `ENV` for sane defaults that the image author wants, while letting `docker run -e` override them:

```dockerfile
ENV NODE_ENV=production \
    PORT=3000
```

---

## 9. The `.dockerignore` file

This is the file everyone forgets, and it costs them. `.dockerignore` lives next to your `Dockerfile` and tells the build context what to exclude.

Why it matters:

1. **Speed.** Smaller build context = faster `docker build`. If you don't ignore `node_modules`, Docker will copy your entire 800MB of dependencies into the build context just to discard them.
2. **Correctness.** Without `.dockerignore`, a stray `.env` file in your project gets included in the build context, and `COPY . .` copies it into the image. You ship your DB password. This is one of the most common ways real production secrets leak.
3. **Reproducibility.** The image you build locally should match the image CI builds. Excluding local junk makes that reliable.

### A solid `.dockerignore` for a Node project

```gitignore
# Dependencies
node_modules
npm-debug.log
yarn-error.log
pnpm-debug.log

# Build output
dist
build
.next
out

# Environment
.env
.env.*
!.env.example

# Git
.git
.gitignore
.github

# IDE / OS noise
.vscode
.idea
.DS_Store
Thumbs.db

# Tests / coverage
coverage
*.test.ts
__tests__

# Docs
README.md
docs
```

Test your ignore by listing what the build context will contain:

```bash
$ docker build --no-cache --progress=plain -t test . 2>&1 | head -30
# Look for "transferring context" lines
```

Or simulate:

```bash
$ tar -cf - --exclude-from=.dockerignore . | wc -c
```

---

## 10. Hands-on: live-reload a Node app

Let's put it all together with a real workflow.

### The Dockerfile (dev mode, slightly different from prod)

```dockerfile
# Dockerfile.dev — for local development with hot reload
FROM node:22-bookworm-slim
WORKDIR /app

COPY package*.json ./
RUN npm install

# We do NOT copy source here; bind mount will provide it.
# But we set up nodemon for hot reload.
RUN npm install --save-dev nodemon ts-node

COPY nodemon.json ./

CMD ["npx", "nodemon", "--watch", "src", "src/server.ts"]
```

`nodemon.json`:

```json
{
  "watch": ["src"],
  "ext": "ts,json",
  "ignore": ["dist"],
  "exec": "ts-node src/server.ts"
}
```

### Run with bind mount

```bash
$ docker build -f Dockerfile.dev -t saas-api:dev .
$ docker run -d -p 3000:3000 \
    -v "$(pwd)/src:/app/src" \
    -v "$(pwd)/package.json:/app/package.json" \
    -v api-node-modules:/app/node_modules \
    saas-api:dev
```

Notice three mounts:

1. `./src` → bind mount, so your edits appear instantly.
2. `./package.json` → bind mount, so adding a dependency is reflected without rebuild.
3. `api-node-modules` → named volume, so the container's installed `node_modules` is not stomped on by the bind mounts (which don't include `node_modules`).

### Try it

1. Open `src/server.ts` in your editor.
2. Add a new route, save.
3. Watch the container logs: `docker logs -f <container>`.
4. Within a second or two, you see nodemon restart the process. Curl the new route.

This is the development experience you want for the rest of Mission 5.

---

## 11. Solving a real problem: "the dev loop is broken"

A junior engineer on your team complains: "I can't get hot reload working. I added a new package with `npm install axios`, but the container doesn't see it." Diagnosis:

The bind mount `./src:/app/src` overwrites `/app/src` inside the container, but the container's `/app/node_modules` is the one created at build time. When you add a new dep, you have to:

1. Stop the container.
2. Rebuild the image (so the `npm install` layer picks it up).
3. Restart.

There is a better pattern: **bind mount only `src`, and also bind mount `package.json` + `package-lock.json`. Re-run `npm install` inside the container when these change.**

Or use **Dev Containers** in VS Code, which handle this complexity for you and give you a terminal inside the container with your editor attached. Recommended in 2026 for any non-trivial project.

---

## 12. Common mistakes

- **Storing data in the writable layer.** "But it works for now." It will not, the day you need to move the container or scale it out.
- **Anonymous volumes everywhere.** Use named volumes; otherwise you cannot reason about your data.
- **Mounting the same path as both a volume and a bind mount.** Docker will not warn you; the second mount wins, and you will be confused. Don't.
- **Forgetting `.dockerignore`.** You will eventually leak a `.env` file or a `node_modules` directory and waste hours debugging.
- **Committing `.env` to git.** Even with `.dockerignore` protecting you, do not put secrets in the repo. Use `.env.example` with placeholders.
- **Bind mounting `node_modules`.** Causes weird issues on macOS/Windows where the host's `node_modules` is wrong architecture. Use a named anonymous volume: `-v node_modules:/app/node_modules`.
- **Hardcoding `localhost` in a multi-container setup.** If the API runs in a container and tries to reach Postgres at `localhost:5432`, it will fail. We will fix this in Module 41 with container networking.

---

## 13. Exercise

**Goal:** Set up a Postgres database that survives container restarts, and a Node API with live-reload code.

**Steps:**

1. Run Postgres with a named volume:
   ```bash
   $ docker volume create pgdata
   $ docker run -d --name pg \
       -e POSTGRES_PASSWORD=dev \
       -e POSTGRES_DB=saas \
       -v pgdata:/var/lib/postgresql/data \
       postgres:16
   $ docker exec -it pg psql -U postgres -d saas -c "CREATE TABLE todos(id serial, title text);"
   ```

2. Insert a row, then `docker stop pg` and `docker rm pg`. Recreate the container with the same `-v pgdata:/var/lib/postgresql/data`. The data is still there. Verify with `psql`.

3. Create a Node API that connects to Postgres. Use bind mounts so you can edit the code live:
   ```bash
   $ docker run -d -p 3000:3000 \
       -v "$(pwd)/src:/app/src" \
       -v api-node-modules:/app/node_modules \
       saas-api:dev
   ```

4. Edit a route, save, watch logs, curl the new route. Confirm hot reload.

5. List everything: `docker ps -a`, `docker volume ls`, `docker network ls`. Note which artifacts survive a restart.

**Deliverable:** A markdown file demonstrating the persistence test (data survives `docker rm`), and the hot-reload test (edited code shows up in the running container).

---

## 14. AI Industry Note (2026)

In 2026, the data-management lesson is the same as it was in 2022, but the stakes are higher. AI workloads generate a lot of state: model checkpoints, embedding caches, vector indices, training datasets. People who skip the volume lesson end up with terabytes of orphan data in anonymous volumes scattered across a fleet. **The vector DB tier in particular** — Pinecone, Weaviate, Qdrant, Milvus — is increasingly run as a self-hosted container with a volume, not a managed cloud service, especially for cost-sensitive startups. Knowing how to provision, back up, and migrate a volume-mounted data store is a real production skill.

Also in 2026: **Docker Desktop's file-sharing performance** (the Achilles' heel of bind mounts on macOS) is much better than it used to be, but you should still benchmark your dev loop. If `tsc` is slow, the answer is often to move `node_modules` into a volume (as we did above), not to buy a faster laptop.

One more: **never put model weights in your image.** A 70B-parameter model in safetensors is ~140GB. Pull it at runtime from S3, Hugging Face, or a private registry into a volume, not into a `COPY` in your Dockerfile.

---

## 15. Looking ahead

In Module 41 we connect containers together. Right now your API and your database are on the same host, but they don't know how to find each other. By the end of Module 41 they will — and you'll understand Docker networking well enough to debug any multi-container setup you encounter in the wild.
