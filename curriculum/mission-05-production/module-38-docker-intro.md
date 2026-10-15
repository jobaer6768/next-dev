# Module 38: Introduction to Docker

> Mission 5 — Ship Software to Production
> "From `works on my machine` to `works everywhere`."

---

## 1. Why this module matters

You have spent Missions 1–4 building a real product: a TypeScript SaaS with a Next.js front end, a Node/Express API, and a Postgres database. It runs on your laptop. Now imagine a junior engineer cloning your repo, running `npm install`, and getting an error about a missing Postgres extension. Or your teammate is on Windows; you're on macOS; CI is on Linux. Three operating systems, three slightly different C libraries, three different definitions of "everything works."

Docker kills that class of bug. It packages your app — code, runtime, system libraries, environment — into a single artifact that behaves identically on every machine that runs the Docker engine. This module is your on-ramp to that world.

By the end of this lesson you will be able to:

- Explain in plain English what Docker is and why it exists.
- Describe the architectural difference between virtual machines and containers.
- Diagram the Docker ecosystem (client, daemon, registry, image, container).
- Decide when Docker is the right answer and when a plain VM is still better.
- Understand how Docker runs on Linux, macOS, and Windows.

---

## 2. The classic problem: "works on my machine"

Picture the following conversation, which has happened in every engineering team in history:

> **Dev A:** "Hey, can you review my PR?"
> **Dev B:** "Sure, pulling it down now... `npm install`... error: `node: error while loading shared libraries: libssl.so.3: cannot open shared object file`"
> **Dev A:** "That's weird, it works for me. What OS?"
> **Dev B:** "Ubuntu 22.04."
> **Dev A:** "I'm on 24.04. Try upgrading OpenSSL?"
> **Dev B:** "Why don't I just throw my laptop in the ocean?"

The root cause: Dev A's application depends not just on `node`, but on a specific version of OpenSSL, a specific glibc, a specific Postgres client lib, a specific system font set. All of that is invisible to the application code but absolutely required for it to run. Dev A's machine happens to have the right combination; Dev B's does not.

Docker solves this by giving you a way to **declare** the entire runtime environment as code, ship it as an artifact, and run it on any host. The declaration is a `Dockerfile`. The artifact is an **image**. The running instance is a **container**.

This is not virtualization theater — it is reproducible builds as a first-class engineering practice.

---

## 3. What is Docker, in one sentence

**Docker is a tool for building, shipping, and running applications inside lightweight, portable, isolated environments called containers.**

A container looks like a tiny virtual machine, but it is not. The difference matters; we'll get to it.

---

## 4. Containers vs Virtual Machines

This is the most important diagram in the whole module. Memorize it.

```
┌──────────────────────────────┐  ┌──────────────────────────────┐
│        VIRTUAL MACHINES      │  │         DOCKER CONTAINERS    │
├──────────────────────────────┤  ├──────────────────────────────┤
│  App A   │  App B   │ App C  │  │  App A   │  App B   │ App C  │
│  Bins/Libs │ Bins/Libs│ ...  │  │  Bins/Libs│ Bins/Libs│ ...  │
├──────────────────────────────┤  ├──────────────────────────────┤
│  Guest OS │ Guest OS │Guest OS│  │                              │
│ (Ubuntu)  │(Alpine)  │(Win)   │  │  (No guest OS — share        │
├──────────────────────────────┤  │   the host kernel)           │
│         Hypervisor           │  ├──────────────────────────────┤
├──────────────────────────────┤  │     Docker Engine / runC     │
│         Host OS              │  ├──────────────────────────────┤
├──────────────────────────────┤  │         Host OS              │
│         Hardware             │  ├──────────────────────────────┤
└──────────────────────────────┘  │         Hardware             │
                                  └──────────────────────────────┘
```

**Key differences:**

| Aspect | Virtual Machine | Docker Container |
|---|---|---|
| Boot time | 30–60 seconds | < 1 second |
| Disk size | Gigabytes (full guest OS) | Megabytes (app + deps) |
| Performance | Near-native, ~5% overhead | Native, ~1% overhead |
| Isolation | Hardware-level via hypervisor | Process-level via kernel namespaces |
| Density | 10–20 per host | Hundreds to thousands per host |
| OS | Can run any OS on any host | Must share host kernel (Linux containers on Linux host; on Mac/Windows Docker runs a small Linux VM under the hood) |

The trade-off: containers are lighter and faster, but they assume the kernel is compatible. A Linux container runs on a Linux kernel. A Windows container runs on a Windows kernel. You can't run a Linux container natively on a Windows kernel without help — and "help" on macOS/Windows is a tiny Linux VM managed by Docker Desktop.

---

## 5. The Docker ecosystem in 2026

Docker is not one program; it is a small constellation of cooperating components.

```
                    ┌──────────────┐
   You (terminal)  │ docker CLI   │   <-- the only thing you type at
                    └──────┬───────┘       (e.g. `docker run`)
                           │ REST API
                           ▼
                    ┌──────────────┐
                    │ dockerd      │   <-- the daemon, the long-running
                    │ (daemon)     │       background process that
                    └──────┬───────┘       actually does the work
                           │
                ┌──────────┼──────────┐
                ▼          ▼          ▼
          ┌────────┐  ┌────────┐  ┌────────┐
          │image   │  │container│  │volume  │ ...etc
          └────────┘  └────────┘  └────────┘
                           │
                           │ pull / push
                           ▼
                    ┌──────────────┐
                    │  Registry    │   <-- a place images live
                    │ (Docker Hub, │       (Docker Hub, GHCR, ECR...)
                    │  GHCR, ECR)  │
                    └──────────────┘
```

**The CLI** is what you invoke. It speaks to the daemon over a local REST socket.

**The daemon (`dockerd`)** is the long-running service that does the heavy lifting: building images, starting containers, talking to the kernel, pulling from registries.

**The registry** is a storage service for images. Docker Hub is the public default. GitHub Container Registry (`ghcr.io`), Amazon ECR, Google Artifact Registry, and others are private alternatives.

**The image** is a read-only template — a frozen snapshot of a filesystem plus metadata.

**The container** is a running instance of an image. You can have many containers from one image.

---

## 6. Where Docker is used in the wild

- **Local development.** The same Postgres, Redis, and Node versions for every teammate. No more "it works for me."
- **CI/CD pipelines.** GitHub Actions, GitLab CI, and Jenkins build images once and ship them. The artifact that passed tests is the artifact that deploys.
- **Microservices.** Each service has its own image. Teams deploy independently. Rollback is "run the previous image tag."
- **Serverless platforms.** AWS Fargate, Google Cloud Run, and Fly.io all run your container image. You don't even manage the host.
- **ML / data science.** A model trained in a notebook is packaged as an image and served behind an API.
- **Legacy rescue.** "It runs on this one CentOS 7 box from 2014" → capture it as an image, run it anywhere.

---

## 7. How Docker runs on each OS

This is the part that confuses people most often. Let's clear it up.

### Linux (native, the easy case)

Docker is a Linux tool at its heart. On a Linux host, the Docker engine uses kernel features (`cgroups`, `namespaces`, `overlayfs`) directly. There is no extra virtualization layer. Performance is native. This is why production servers almost always run Linux.

```bash
# On a Linux host
$ sudo apt-get install docker.io
$ sudo usermod -aG docker $USER
$ docker run hello-world
```

### macOS (Docker Desktop, with a VM)

macOS is not Linux. It uses the XNU kernel. Docker Desktop for Mac runs a tiny Linux VM under the hood (using Apple's Virtualization framework, which replaced the older HyperKit). Your containers run inside that VM. The file sharing and networking are bridged so it feels native.

```
┌─────────────────────────────────┐
│  macOS host                     │
│  ┌───────────────────────────┐  │
│  │  Linux VM (managed by     │  │
│  │  Docker Desktop)          │  │
│  │  ┌─────────────────────┐  │  │
│  │  │  your containers    │  │  │
│  │  └─────────────────────┘  │  │
│  └───────────────────────────┘  │
│  /var/run/docker.sock (bridge)  │
└─────────────────────────────────┘
```

Performance is good but not Linux-native. File sync between host and container is the slowest part. In 2026, the new `docker init` and improved `virtiofs` file sharing have made this much faster, but you should still expect some I/O penalty for heavy `node_modules` work.

### Windows (Docker Desktop with WSL2)

Same idea as macOS, different VM backend. Modern Docker Desktop on Windows uses **WSL2** (the Windows Subsystem for Linux) as a tiny Linux VM. Your containers run inside the WSL2 distro. File paths are awkward (`/mnt/c/Users/you/...`), but performance is decent.

For Windows Server in production, there is also a native Windows Container mode that runs Windows-based containers (for legacy .NET Framework apps, mostly). Outside that niche, everything is Linux containers.

### Why this matters for your team

- **Production servers will be Linux.** Always. Get comfortable with Linux commands.
- **Your laptop is for development.** Pick macOS or Windows with Docker Desktop; both work. Linux is the smoothest dev experience but the most friction to set up if you've never used it.
- **CI is Linux.** GitHub Actions runners are Ubuntu. Test there before you trust a deploy.

---

## 8. A 30-second hands-on

After installing Docker, this single command proves the whole stack works:

```bash
$ docker run hello-world
```

What happens, step by step:

1. The CLI contacts the daemon: "Please run the image `hello-world`."
2. The daemon doesn't have the image locally, so it pulls it from Docker Hub.
3. The daemon unpacks the image into a container.
4. The container runs a tiny program that prints a multi-paragraph success message.
5. The container exits.

If you see that message, your client, daemon, registry connection, container runtime, and kernel features are all working. You're ready for Module 39.

---

## 9. Common mistakes

- **"Docker is just a lightweight VM."** No. It shares the kernel. It is fundamentally a process-isolation primitive. If you don't internalize this, you'll be confused by weird behaviors later (e.g. you can't run a Windows container on a Linux host without nested tricks).
- **"I should Dockerize my OS."** Docker is for applications, not for replacing your OS. Don't `docker run ubuntu` and try to use it as a daily driver.
- **"Containers are less secure than VMs because they share the kernel."** True in theory, mitigated in practice by seccomp, AppArmor, and rootless mode. We'll cover hardening in Module 44.
- **Confusing images and containers.** An image is a class. A container is an instance. You can run five containers from one image, each with different state.
- **Running everything as root inside containers.** We'll fix this in Module 42. For now, just know it's a thing.

---

## 10. Exercise

**Goal:** Prove your Docker installation is real and that you understand the client/daemon split.

**Steps:**

1. Install Docker Desktop (macOS/Windows) or `docker.io` (Linux).
2. Open a terminal and run `docker version`. Copy the output. You should see both a **Client** and a **Server** section. The Server version proves the daemon is running.
3. Run `docker run hello-world`. Copy the output.
4. Run `docker run -it ubuntu:24.04 bash`. You are now inside a container. Run `cat /etc/os-release` and `uname -a`. Notice the kernel is your host's kernel. Type `exit` to leave.
5. Run `docker ps -a` to see your two exited containers.

**Deliverable:** A short markdown file with all four outputs and a one-paragraph answer to: "Why does `uname -a` inside the Ubuntu container show your host's kernel, not Ubuntu's?"

---

## 11. AI Industry Note (2026)

In 2026, containers are the lingua franca of AI deployment. Every serious model-serving platform — vLLM, Triton Inference Server, Text Generation Inference, Ollama — ships as a container image. When OpenAI, Anthropic, and Google serve inference at scale, the unit of deployment is a container orchestrating GPUs. Tools like **Modal**, **Replicate**, and **Beam** are essentially "containers as a service" with a developer experience built for ML workloads. Even training jobs increasingly run on Kubernetes-scheduled containers with attached GPU pods.

For you, the practical takeaway: any AI feature you build — a RAG pipeline, a fine-tuned classifier, a vector embedding service — will be deployed as a container. The Docker skills in this mission are not a side quest; they are the prerequisite to shipping AI products.

Also in 2026: **Docker Scout** and **Trivy** are now standard in CI pipelines for vulnerability scanning. **Rootless Docker** has matured and is the default in many security-sensitive orgs. **WebAssembly containers** (via `runwasi`) are an emerging alternative for ultra-fast cold starts but have not displaced Linux containers for general workloads. We will touch on image security in Module 44 when we set up the production deploy.

---

## 12. Looking ahead

In Module 39 we install the toolchain, write our first Dockerfile, build an image, and run a container. You'll go from "I have heard of Docker" to "I shipped a Node app in a container" by the end of the next lesson.
