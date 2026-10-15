# Module 39: Docker Image and Container

> Mission 5 — Ship Software to Production
> "If Module 38 was the map, this is where you start driving."

---

## 1. Why this module matters

Module 38 was conceptual. This one is mechanical. By the end of it you will have:

- Installed Docker on your machine.
- Configured VS Code with the Docker extension.
- Written a real Dockerfile from scratch.
- Built an image, run a container from it, debugged it, and pushed it to Docker Hub.

The single most important thing you will do in this module is internalize the difference between an **image** (a frozen, immutable artifact) and a **container** (a running instance with mutable state). Everything else flows from that.

---

## 2. Learning objectives

By the end of this module you can:

- Install Docker on Windows, macOS, or Linux and verify the daemon is reachable.
- Configure VS Code with the official Docker extension.
- Explain images, layers, copy-on-write, and image history.
- Use pre-built images from Docker Hub with `docker pull` and `docker run`.
- Write a Dockerfile with `FROM`, `RUN`, `COPY`, `CMD`, `WORKDIR`, `EXPOSE`, and `ENV`.
- Build an image with `docker build` and tag it.
- Run a container with `docker run` and manage it with `ps`, `logs`, `exec`, `stop`, `rm`.
- Use restart policies (`no`, `on-failure`, `always`, `unless-stopped`).
- Push an image to Docker Hub and pull it from a different machine.

---

## 3. Installation

### macOS

1. Download **Docker Desktop for Mac** (Apple Silicon or Intel, depending on your Mac) from `https://www.docker.com/products/docker-desktop/`.
2. Drag the `.app` to Applications. Open it. Approve the privileged helper tool.
3. Confirm the whale icon in your menu bar says "Docker Desktop is running."
4. Verify:

```bash
$ docker --version
Docker version 27.x, build ...
$ docker run hello-world
```

### Windows

1. Install **WSL2** first: open PowerShell as admin and run `wsl --install`. Reboot.
2. Install **Docker Desktop for Windows** from `docker.com`. It will detect WSL2 and use it.
3. In Docker Desktop settings, confirm "Use the WSL 2 based engine" is checked.
4. Open a terminal and run `docker run hello-world`.

### Linux (Ubuntu 24.04 example)

```bash
# Uninstall any old versions
$ sudo apt-get remove docker docker-engine docker.io containerd runc

# Set up the repository
$ sudo apt-get update
$ sudo apt-get install ca-certificates curl gnupg
$ sudo install -m 0755 -d /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
$ sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

$ sudo apt-get update
$ sudo apt-get install docker-ce docker-ce-cli containerd.io \
    docker-buildx-plugin docker-compose-plugin

# Allow your user to run docker without sudo
$ sudo usermod -aG docker $USER
newgrp docker
$ docker run hello-world
```

**Sanity check on every OS:**

```bash
$ docker version        # Should show Client AND Server
$ docker info           # Should show server version, storage driver, etc.
$ docker run hello-world
```

---

## 4. VS Code + the Docker extension

Open VS Code. Install the **Docker** extension by Microsoft (search "docker" in the Extensions sidebar). It gives you:

- A `DOCKER` panel in the sidebar with `Containers`, `Images`, `Volumes`, `Networks`, and `Registries` views.
- Right-click on a container to see logs, attach a shell, inspect, or stop it.
- Right-click on an image to run it interactively.
- Inline `Dockerfile` linting and IntelliSense.
- A `>Docker: Add Docker Files to Workspace` command that scaffolds a `Dockerfile` and `.dockerignore` for Node, Python, Go, etc. Use it as a starting point; do not blindly trust it.

Also install **Remote - Containers** (now called "Dev Containers"). It lets you open a folder inside a container as a full dev environment. We will use it in Module 40 for hot-reload development.

---

## 5. Images and containers: the mental model

### Image

An image is a **read-only, layered filesystem plus metadata**. It is a snapshot. It does not run. You can't `cd` into an image. You can only build it, push it, pull it, and run it.

### Container

A container is a **running instance of an image**. It has:

- The image's layers (read-only, shared with other containers from the same image).
- A thin **writable layer** on top (the "container layer") that captures any changes made at runtime.
- Its own process namespace, network namespace, and mount namespace.
- A lifecycle: created → running → paused → stopped → deleted.

### Layers and copy-on-write

Every line in a Dockerfile creates a new layer. Each layer is a diff against the layer below it. When you start a container, Docker stacks these layers and adds a writable top layer:

```
┌─────────────────────────┐
│  Writable layer         │  <-- container-specific changes
├─────────────────────────┤
│  Layer 5: COPY app.js   │
├─────────────────────────┤
│  Layer 4: RUN npm ci    │
├─────────────────────────┤
│  Layer 3: COPY package  │
├─────────────────────────┤
│  Layer 2: RUN apt ...   │
├─────────────────────────┤
│  Layer 1: FROM node:22  │
└─────────────────────────┘
```

Two big implications:

1. **Sharing.** If you have ten containers running `node:22`, they all share the same read-only base layer. Only the writable layer is per-container. This is why images are so small on disk relative to how much you can do with them.
2. **Caching.** When you rebuild an image, Docker reuses unchanged layers from the cache. Order your Dockerfile so that the things that change least are at the top.

---

## 6. Using pre-built images

You don't have to build every image from scratch. The Docker Hub registry has thousands of official images you can pull.

```bash
$ docker pull nginx
$ docker pull redis
$ docker pull postgres:16
$ docker pull mcr.microsoft.com/dotnet/aspnet:8.0
```

Notice the optional `:tag` syntax. `nginx` is shorthand for `nginx:latest`. For reproducibility, **always pin a specific tag** in your Dockerfiles and CI.

### Run an Nginx web server

```bash
$ docker run -d -p 8080:80 --name my-nginx nginx
```

Let's decode this:

- `-d` — run in the background (detached).
- `-p 8080:80` — publish port 80 in the container to port 8080 on your host. Open `http://localhost:8080` to see the Nginx welcome page.
- `--name my-nginx` — give it a name so you can refer to it later.
- `nginx` — the image to run.

Visit `http://localhost:8080`. You should see "Welcome to nginx!"

### Common one-liners

```bash
$ docker run --rm -it alpine sh        # throwaway shell
$ docker run -d --name pg -e POSTGRES_PASSWORD=dev postgres:16
$ docker run --rm -v $(pwd):/data alpine ls /data
```

---

## 7. Your first Dockerfile

We are going to containerize the Node/Express API you built in Mission 4. Say the project looks like this:

```
api/
├── package.json
├── tsconfig.json
├── src/
│   └── server.ts
└── Dockerfile          <-- we will write this
```

`package.json` (excerpt):

```json
{
  "name": "saas-api",
  "version": "1.0.0",
  "scripts": {
    "build": "tsc -p .",
    "start": "node dist/server.js"
  },
  "dependencies": {
    "express": "^4.21.0"
  },
  "devDependencies": {
    "typescript": "^5.6.0",
    "@types/express": "^4.17.0"
  }
}
```

Here is a clean, production-ready Dockerfile for it:

```dockerfile
# syntax=docker/dockerfile:1.7
# ----- Stage 1: deps + build -----
FROM node:22-bookworm-slim AS build
WORKDIR /app

# Install only prod deps first for better caching
COPY package*.json ./
RUN npm ci

# Copy source and build
COPY tsconfig.json ./
COPY src ./src
RUN npm run build

# ----- Stage 2: runtime -----
FROM node:22-bookworm-slim AS runtime
WORKDIR /app

ENV NODE_ENV=production
ENV PORT=3000

# Create non-root user
RUN groupadd --system --gid 1001 nodejs \
    && useradd --system --uid 1001 --gid nodejs appuser

# Copy only what we need to run
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY --from=build /app/dist ./dist

USER appuser
EXPOSE 3000

CMD ["node", "dist/server.js"]
```

This is a **multi-stage build**. Stage 1 compiles TypeScript; stage 2 ships only the compiled output and production dependencies. The final image is typically 5–10× smaller than a naive single-stage build.

### Build and run

```bash
$ cd api
$ docker build -t saas-api:1.0.0 .
```

The `-t` flag tags the image. The trailing `.` says "the build context is the current directory."

```bash
$ docker run -d -p 3000:3000 --name api saas-api:1.0.0
$ curl http://localhost:3000/health
{"ok":true}
```

---

## 8. Common commands

### Images

```bash
$ docker images                 # list local images
$ docker image ls --format "{{.Repository}}:{{.Tag}} {{.Size}}"
$ docker history saas-api:1.0.0 # show layers and their sizes
$ docker rmi saas-api:1.0.0     # remove an image
$ docker image prune -a         # remove all unused images
```

### Containers

```bash
$ docker ps                # running only
$ docker ps -a             # all, including stopped
$ docker logs api          # stdout/stderr of the container
$ docker logs -f api       # follow (tail -f style)
$ docker exec -it api sh   # run a shell inside the running container
$ docker stop api          # SIGTERM, then SIGKILL after 10s
$ docker start api         # start a stopped container
$ docker rm api            # remove a stopped container
$ docker rm -f api         # force remove (stop + rm)
$ docker container prune   # remove all stopped containers
```

### Inside a container

```bash
$ docker exec api ps aux         # what's running?
$ docker exec api cat /etc/os-release
$ docker exec api env            # see environment variables
$ docker exec api ls /app        # see the filesystem
```

---

## 9. Restart policies

When your server reboots, or the Docker daemon restarts, what should happen to your containers? You set this with `--restart`:

| Policy | Behavior |
|---|---|
| `no` | (default) Do not restart. |
| `on-failure[:max-retries]` | Restart only if the container exits with a non-zero code. |
| `always` | Always restart. If the daemon restarts, the container starts. |
| `unless-stopped` | Like `always`, but does not restart if you manually stopped it. |

Examples:

```bash
# Production web server
$ docker run -d --restart unless-stopped -p 80:80 nginx

# Job that should fail loudly if it dies
$ docker run -d --restart on-failure:5 my-batch-job

# Local dev: don't restart
$ docker run -d --restart no my-app
```

In 2026 we almost never set restart policies on individual `docker run` commands in dev. In production we use Compose or Kubernetes, which have their own restart semantics. But for quick tests and one-offs, the flag is there.

---

## 10. Pushing to and pulling from Docker Hub

### Create a Docker Hub account

1. Sign up at `https://hub.docker.com/`.
2. Create a repository, e.g. `yourname/saas-api`.

### Login from the CLI

```bash
$ docker login
# prompts for username + password (or token)
```

A token, not your password. Generate one in Docker Hub → Account Settings → Security → New Access Token. Treat it like any other credential.

### Tag and push

```bash
$ docker tag saas-api:1.0.0 yourname/saas-api:1.0.0
$ docker push yourname/saas-api:1.0.0
```

The `tag` command creates a new pointer to the same image with a new name. The `push` command uploads the layers that aren't already on the registry.

### Pull from anywhere

```bash
# On another machine
$ docker pull yourname/saas-api:1.0.0
$ docker run -d -p 3000:3000 yourname/saas-api:1.0.0
```

That's the whole point: a teammate, a CI runner, or a production server can run your app with no `npm install`, no Postgres, no system libraries to install.

### Versioning tags

Use semver or SHA tags. Never push `:latest` for production-critical images — `latest` is a moving target and breaks reproducibility.

```bash
$ docker build -t yourname/saas-api:1.0.0 .
$ docker build -t yourname/saas-api:1.0 .
$ docker build -t yourname/saas-api:sha-$(git rev-parse --short HEAD) .
$ docker push yourname/saas-api:1.0.0 --all-tags
```

---

## 11. Common mistakes

- **Confusing `docker run` and `docker start`.** `run` creates a new container from an image. `start` starts an existing, stopped container.
- **Not pinning tags.** `FROM node` becomes a different image every time node publishes. Use `node:22.11-bookworm-slim` or pin a digest `node@sha256:...`.
- **Putting `COPY . .` before `npm ci`.** That invalidates the dependency cache on every code change. Always copy manifests first, install, then copy source.
- **Running as root.** Production images should drop to a non-root user. We'll reinforce this in Module 42.
- **Forgetting `--rm` during experiments.** You will accumulate dozens of stopped containers. `docker run --rm` cleans up automatically when the container exits.
- **Pushing `.env` to Docker Hub.** Never. We'll handle secrets properly in Module 40 and 42.

---

## 12. Exercise

**Goal:** Containerize the SaaS API you built in Mission 4, run it, push it to Docker Hub.

**Steps:**

1. Pick the Node/Express API from Mission 4. If you don't have one, use this minimal one:

```ts
// src/server.ts
import express from "express";
const app = express();
app.get("/health", (_req, res) => res.json({ ok: true, ts: Date.now() }));
app.listen(3000, () => console.log("API on :3000"));
```

2. Write a multi-stage `Dockerfile` modeled on the one in section 7. Build it: `docker build -t saas-api:1.0.0 .`.
3. Run it: `docker run -d -p 3000:3000 --name api saas-api:1.0.0`. Curl `/health` and confirm a 200.
4. Inspect the image: `docker history saas-api:1.0.0`. Identify the biggest layers.
5. Use `docker exec -it api sh` to look around inside the running container. Confirm the app user is not root (`whoami`).
6. Push to Docker Hub: tag with your username, then `docker push`.
7. On a different machine (or after removing the local image with `docker rmi saas-api:1.0.0`), pull and run it.

**Deliverable:** A markdown file with:
- The final `Dockerfile`.
- Output of `docker history`.
- A screenshot or curl output showing `/health` working both locally and after a fresh pull.

---

## 13. AI Industry Note (2026)

The single most common production bug for AI startups in 2026 is still "it worked on my laptop." LLM-based apps in particular depend on a moving target of CUDA versions, model weights, and tokenizer data. Containers are the only sane way to ship these. If you ever fine-tune a model, the standard practice is: train it in a container, evaluate in a container, serve in a container. Tools like **Modal** and **Replicate** are essentially opinionated takes on "give me a Python script, I'll run it in the right container with a GPU."

Also in 2026: **multi-stage builds are table stakes**. If you see an AI/ML Dockerfile that copies `node_modules` into a runtime stage without pruning dev deps, run. The image is 4GB and you will pay for it in CI minutes, registry storage, and pull latency.

One more: **SBOMs (Software Bills of Materials)** are now often required by enterprise customers. `docker buildx imagetools inspect yourname/saas-api:1.0.0` can produce an SBOM. We'll use this in Module 44 when we set up the production pipeline.

---

## 14. Looking ahead

In Module 40 we cover data: how to persist a Postgres database, how to mount your source code for hot reload, and how to handle secrets. Containers are ephemeral — if you don't learn this lesson now, you will lose data and not understand why.
