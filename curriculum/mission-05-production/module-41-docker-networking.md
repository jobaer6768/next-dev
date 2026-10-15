# Module 41: Docker Networking and Cross-Container Communication

> Mission 5 — Ship Software to Production
> "If volumes are how containers remember, networking is how they talk."

---

## 1. Why this module matters

In Module 40 you ran a Postgres container and a Node API container, both on the same host. They are right next to each other. And yet, if the API tries to connect to Postgres at `localhost:5432`, it fails. Why? Because **the API's `localhost` is not the host's `localhost`.** Each container is its own little machine with its own network namespace.

This module is the missing piece between "I can run one container" and "I can run a real system." By the end, your API will talk to Postgres using a hostname instead of an IP address, your containers will be able to reach the internet for outbound API calls, and you will be able to read a `docker network inspect` output without flinching.

---

## 2. Learning objectives

- List the three categories of container communication and the right tool for each.
- Make a container reach the public internet (egress).
- Make a container reach the host machine (`host.docker.internal`).
- Make two containers on the same host talk to each other.
- Explain the difference between IP-based and DNS-based container discovery.
- Diagram the full Docker networking architecture: client, daemon, registries, networks.
- Read `docker network inspect` output.

---

## 3. Three kinds of communication

When you stand up a system of containers, they need to talk to each other and to the outside world. There are exactly three categories of communication:

```
                ┌────────────────────┐
                │  The Internet (WWW)│
                └─────────┬──────────┘
                          │ (3) egress
                          │
   ┌──────────────────────┴──────────────────────┐
   │  Your host machine                          │
   │                                             │
   │  ┌─────────────┐         ┌─────────────┐    │
   │  │ container A │◄───────►│ container B │    │
   │  └─────────────┘  (1)    └─────────────┘    │
   │        ▲                          ▲         │
   │        │       (2) host.docker.internal    │
   │        └──────────┬─────────────────┘       │
   │                   │                         │
   │              host machine                   │
   └─────────────────────────────────────────────┘
```

1. **Container ↔ Container.** The default bridge network, or user-defined bridge networks with DNS.
2. **Container ↔ Host.** The special hostname `host.docker.internal`.
3. **Container ↔ Internet.** Egress, by default, just works.

Let's go through each.

---

## 4. Container ↔ Internet (egress)

By default, Docker containers have outbound internet access. They can pull packages, hit APIs, fetch updates.

```bash
$ docker run --rm alpine sh -c "wget -qO- https://api.github.com/zen | head -1"
Speak like a human
```

This works because the default bridge network NATs container traffic to the host's IP. The container thinks it's reaching the internet directly; in reality, the Docker engine is doing the translation.

If you've ever had to lock down a container so it *can't* reach the internet (for security reasons), you'd use the `none` network or iptables rules. We won't do that in this mission, but the capability exists.

---

## 5. Container ↔ Host (`host.docker.internal`)

Sometimes a container needs to talk to a service that is running on the host directly, not in another container. The classic example: you have Postgres running natively on your Mac (not in Docker) for some legacy reason, and your Dockerized API needs to connect to it.

The special hostname **`host.docker.internal`** resolves to the host machine's IP from inside a container.

```bash
# Start a service on the host. For demo, run a simple HTTP server:
$ python3 -m http.server 8000

# From inside a container, reach it:
$ docker run --rm alpine wget -qO- http://host.docker.internal:8000
# ... serves files from the current directory
```

The actual mechanism: Docker adds a special DNS entry that points to a virtual IP on the host's `docker0` interface. This is **only on Docker Desktop** (macOS/Windows) by default. On Linux, you may need to add it explicitly:

```bash
$ docker run --add-host=host.docker.internal:host-gateway ...
```

### When to use this

- Connecting from a Dockerized dev tool to a host-installed service.
- Reaching a host-only service like `localhost:11434` (Ollama, the local LLM runtime — relevant in 2026).
- Hitting a service your team has not Dockerized yet (e.g. corporate LDAP).

### When NOT to use this

- Talking to other Docker containers on the same host. Use container-to-container networking, which is the next section.

---

## 6. Container ↔ Container (the heart of the matter)

This is the one you'll use 95% of the time. Two scenarios: IP-based and DNS-based.

### Option A: IP-based (works, but don't)

Every container on the default bridge network gets a private IP like `172.17.0.2`. You can find it:

```bash
$ docker run -d --name pg -e POSTGRES_PASSWORD=dev postgres:16
$ docker inspect pg --format '{{.NetworkSettings.IPAddress}}'
172.17.0.2

$ docker run --rm alpine sh -c "nc -z 172.17.0.2 5432 && echo OK"
OK
```

This works. The problem: container IPs change on restart. The day you replace the Postgres container, its IP might be `172.17.0.5` instead, and your API breaks. **Never hardcode container IPs.**

### Option B: DNS-based (the right answer)

When you create a **user-defined bridge network**, Docker runs a built-in DNS server. Containers can refer to each other by name, and Docker resolves the name to the current IP automatically.

```bash
# 1. Create a network
$ docker network create appnet

# 2. Run Postgres on it
$ docker run -d --name db --network appnet \
    -e POSTGRES_PASSWORD=dev postgres:16

# 3. Run the API on the SAME network
$ docker run -d --name api --network appnet \
    -e DATABASE_URL=postgres://postgres:dev@db:5432/postgres \
    saas-api:1.0.0

# 4. The API can now reach Postgres at hostname `db`
$ docker exec api sh -c "nc -z db 5432 && echo OK"
OK
```

This is the canonical pattern. Notice:

- The hostname is the **container name** (`db`), not the IP.
- The hostname is stable across container restarts as long as the name is stable.
- Both containers must be on the same user-defined network for DNS to work.

### Why user-defined networks are better than the default

The default `bridge` network does **not** provide automatic DNS resolution. You can ping by IP but not by name. User-defined networks do. They also provide automatic isolation: a container on `appnet` cannot see a container on `dbnet` unless you explicitly connect them.

### Inspect a network

```bash
$ docker network inspect appnet
```

You will see a JSON blob with a `Containers` section listing each connected container by ID, name, and IP. Bookmark this command; you will use it to debug "why can't these two containers see each other."

### Other useful network commands

```bash
$ docker network ls                          # list networks
$ docker network create --driver bridge appnet
$ docker network rm appnet
$ docker network connect appnet existing-container
$ docker network disconnect appnet existing-container
$ docker network prune                       # remove unused networks
```

---

## 7. Architecting a real network

For the SaaS app from Mission 4, you might end up with something like:

```bash
$ docker network create frontend
$ docker network create backend
```

Then:

- The Next.js frontend is connected to `frontend` (and the host, via port publishing).
- The Node API is connected to both `frontend` and `backend`.
- Postgres is connected to `backend` only — the frontend cannot reach it directly.

This is **network isolation as a security boundary**. The frontend is the part of the system that browsers talk to; the database is a tier that only the API should touch. By putting them on different networks, you make that constraint physically enforced.

We will codify this in Module 43 with Docker Compose.

---

## 8. Container ↔ WWW in code

A real Node API needs to call external services. Here is the SaaS API's outbound call to a payment provider (Stripe, in this case):

```ts
// src/server.ts
import express from "express";
import Stripe from "stripe";

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);
const app = express();

app.post("/api/checkout", async (req, res) => {
  const session = await stripe.checkout.sessions.create({
    mode: "payment",
    line_items: [{ price: "price_123", quantity: 1 }],
    success_url: "https://example.com/success",
    cancel_url: "https://example.com/cancel",
  });
  res.json({ url: session.url });
});

app.listen(3000, () => console.log("API on :3000"));
```

Inside the container, this just works. The DNS resolution for `api.stripe.com` and the egress traffic both flow through the default Docker network. From the host's perspective, traffic appears to come from the host's IP. From Stripe's perspective, they cannot tell you're inside a container.

### Gotcha: long-lived connections

If you have many short-lived outbound connections (e.g. a connection pool to an external DB), you may exhaust the ephemeral port range on the host. In practice, this only bites you at high concurrency (thousands of connections). Not a Module 41 problem; flag it for a future "production observability" lesson.

---

## 9. Host communication in code

If you have a sidecar service on the host (e.g. a local LLM at `localhost:11434`), connect to it from a containerized app:

```ts
// src/llm.ts
import OpenAI from "openai";

// host.docker.internal works on Docker Desktop.
// On Linux, pass --add-host=host.docker.internal:host-gateway
const client = new OpenAI({
  baseURL: "http://host.docker.internal:11434/v1",
  apiKey: "ollama", // Ollama ignores the key
});

export async function summarize(text: string) {
  const r = await client.chat.completions.create({
    model: "llama3.2",
    messages: [{ role: "user", content: `Summarize: ${text}` }],
  });
  return r.choices[0].message.content;
}
```

In 2026 this pattern is everywhere: developers run small local models via Ollama, LM Studio, or vLLM on their laptop, and their containerized apps reach them via `host.docker.internal`. Cool. Be aware of the security implications — anything listening on the host's loopback is reachable from any container.

---

## 10. Docker architecture, full picture

You have now seen every major piece of the Docker ecosystem. Let's draw it all together:

```
   Developer              docker CLI ────► dockerd (daemon) ────► containerd ────► runC
      │                                                  │              │
      │                                                  │              │
      ▼                                                  ▼              ▼
  Edit code                                        images,            kernel
  (host)                                           containers,        (namespaces,
                                                   volumes,           cgroups,
                                                   networks          overlayfs)
                                                      │
                                                      │ pull / push
                                                      ▼
                                              ┌───────────────┐
                                              │  Registry     │
                                              │  - Docker Hub │
                                              │  - GHCR       │
                                              │  - ECR / GCR  │
                                              └───────────────┘
```

The CLI is a thin client. All the real work happens in the daemon, which orchestrates images, containers, volumes, and networks. The daemon talks to `containerd`, which talks to `runC`, which actually uses Linux kernel features to create the isolated process.

You, the developer, almost never need to interact directly with `containerd` or `runC`. You talk to the CLI; the CLI talks to the daemon; the daemon does the work.

### Registries in 2026

- **Docker Hub** is the public default. Many official images live here. Free for public repos; private repos require a paid plan or a personal access token.
- **GitHub Container Registry (`ghcr.io`)** is increasingly common because it integrates with GitHub Actions and packages.
- **AWS ECR, GCP Artifact Registry, Azure ACR** for cloud-native workflows.
- **Self-hosted registries** (Harbor, Distribution) for air-gapped environments.

For Mission 5 we will use GHCR because every student in this cohort already has a GitHub account.

---

## 11. A worked example: full multi-container setup

Let's put it all together with manual `docker run` commands (in Module 43 we will replace this with Compose).

```bash
# Network
$ docker network create saas-net

# Postgres
$ docker run -d --name db \
    --network saas-net \
    -e POSTGRES_USER=app \
    -e POSTGRES_PASSWORD=dev \
    -e POSTGRES_DB=saas \
    -v pgdata:/var/lib/postgresql/data \
    --health-cmd="pg_isready -U app" \
    --health-interval=5s \
    postgres:16

# API
$ docker run -d --name api \
    --network saas-net \
    -e DATABASE_URL=postgres://app:dev@db:5432/saas \
    -e NODE_ENV=production \
    -p 127.0.0.1:3000:3000 \
    saas-api:1.0.0

# Frontend
$ docker run -d --name web \
    --network saas-net \
    -e NEXT_PUBLIC_API_URL=http://api:3000 \
    -p 127.0.0.1:3001:3000 \
    saas-web:1.0.0

$ docker ps
CONTAINER ID   IMAGE             ...   PORTS                      NAMES
abc123...      saas-api:1.0.0    ...   127.0.0.1:3000->3000/tcp   api
def456...      saas-web:1.0.0    ...   127.0.0.1:3001->3000/tcp   web
789abc...      postgres:16       ...                             db
```

Notice the API references Postgres as `db` (the container name) — DNS resolution via the user-defined network. The frontend references the API as `api`. Both bind to `127.0.0.1` on the host, not `0.0.0.0`, so they're not exposed to the world (we'll put an Nginx reverse proxy in front in Module 44).

### Sanity checks

```bash
$ docker exec api sh -c "nc -z db 5432 && echo 'db reachable'"
db reachable

$ docker exec web sh -c "wget -qO- http://api:3000/health"
{"ok":true}

$ docker logs db --tail 20
```

### Cleanup

```bash
$ docker stop api web db
$ docker rm api web db
$ docker network rm saas-net
$ docker volume rm pgdata   # only if you want to nuke the data
```

---

## 12. Common mistakes

- **Containers on the default bridge trying to reach each other by name.** Doesn't work. Make a user-defined network.
- **Hardcoding IPs.** They change. Always use container names (DNS) or service names (in Compose).
- **Publishing ports with `-p 3000:3000` on a server with no firewall.** That's a wide-open door. Use `-p 127.0.0.1:3000:3000` and put a reverse proxy in front (Module 44).
- **Reaching `localhost:5432` from inside a container to talk to a host database.** Use `host.docker.internal` instead.
- **Not checking `docker network inspect` first when debugging.** "Why can't container A reach container B?" 90% of the time the answer is "they're on different networks."
- **Putting the database on the same network as the frontend.** Splits in two: a `backend` net (db, api) and a `frontend` net (api, web). The API bridges them; the frontend never sees the database directly.

---

## 13. Module recap

You now know:

- Egress (container → internet) just works via NAT.
- Host communication uses `host.docker.internal`.
- Container-to-container uses user-defined bridge networks with DNS.
- The Docker architecture is CLI → daemon → containerd → runC → kernel.
- Registries are where images live, and there are several reasonable choices.

### Capstone project idea for this module

Run the full SaaS stack — Postgres, API, frontend — with three `docker run` commands on a user-defined network, then add a Redis cache the same way, and prove that the API can reach all three by name. Add a fourth service (an Nginx reverse proxy) that the API and frontend both talk to, demonstrating a service that bridges two networks.

---

## 14. Exercise

**Goal:** Stand up a 3-service system using only `docker run` and user-defined networks, and prove that DNS-based discovery works.

**Steps:**

1. Create a network: `docker network create labnet`.
2. Run an Nginx container: `docker run -d --name proxy --network labnet nginx`.
3. Run an Alpine container attached to the same network: `docker run -it --rm --network labnet alpine sh`.
4. From inside Alpine, run `wget -qO- http://proxy/`. You should see the Nginx welcome page — that proves DNS resolution and HTTP between containers.
5. Run a third container **not on the network**: `docker run -d --name isolated alpine sleep 3600`.
6. From a container on `labnet`, try to reach `isolated`. It should fail. Then `docker network connect labnet isolated` and try again. It should succeed.
7. Clean up: `docker rm -f proxy isolated` and `docker network rm labnet`.

**Deliverable:** Markdown file with the four `wget`/`nc` outputs proving DNS works on a user-defined network, and a one-paragraph explanation of why the same hostname pattern fails on the default bridge.

---

## 15. AI Industry Note (2026)

In AI infrastructure, multi-container networking is a daily concern. A typical 2026 RAG stack looks like:

- A vector DB (Qdrant, Weaviate, Milvus) on a private network.
- An embedding service (TEI, vLLM) on a private network.
- A Postgres database for metadata.
- The application API, which bridges the private networks and the public internet.
- A frontend, which talks only to the API.

All of this is just user-defined Docker networks and DNS. The same patterns. The same tools. The same debugging commands (`docker network inspect`).

One 2026-specific note: **GPU-aware networking** is becoming a thing. NVIDIA's `nvidia-container-toolkit` and the new Docker GPU runtime let containers share a GPU. When two containers need to talk to a shared model server (e.g. vLLM with a 70B model in safetensors), the same DNS-based discovery works — but you also need to make sure the GPU runtime is consistent across all containers that need it. `docker run --gpus all ...` is the entry point. We'll touch on this when we cover the `deploy` Makefile target in Module 43.

---

## 16. Looking ahead

You now know how to run multiple containers and have them talk. Doing it with `docker run` is fine for a lab; it is painful for a real system. Module 42 introduces the pattern of building a proper multi-container application (one container per process, with healthchecks and proper isolation). Module 43 then replaces the dozen `docker run` commands with a single declarative `docker-compose.yml` file that anyone on the team can run with `docker compose up`.
