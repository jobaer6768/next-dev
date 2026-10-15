# Next Level — Software Development with AI Engineering

> A complete, instructor-led walk-through of the **Programming Hero Next Level Batch 8** curriculum.
> Every mission, every module, every assignment is broken down into a single, focused `.md` lesson you can read end-to-end or skim for review.

---

## Who this is for

You are a working developer-in-training who wants to become a **Full-Stack engineer with AI engineering superpowers** — the kind of engineer companies *cannot* easily replace with AI alone, because you can:

- Ship production TypeScript with confidence.
- Design and reason about relational databases.
- Build a multi-tenant SaaS backend with proper auth, RBAC, and observability.
- Build a real Next.js frontend that talks to it.
- Containerize, deploy, and operate the whole thing.
- Layer **LLMs, RAG, and AI agents** on top — safely, at production scale.

You do not need a CS degree. You do need discipline, curiosity, and a willingness to actually *build* what you read.

---

## How to use this curriculum

1. **Work in order.** Missions build on each other. Skipping ahead will hurt.
2. **Code every example.** Reading is not learning. Type it, run it, break it, fix it.
3. **Do the assignments.** Each `assignment-XX.md` file is a graded, portfolio-worthy project.
4. **Read the AI Sidebar.** At the end of each lesson I add an "AI Industry Note" — how this topic is actually evolving in 2026 and what hiring managers look for.

---

## Mission Map

| # | Mission | What you build | Duration focus |
|---|---------|----------------|----------------|
| 1 | **Build with TypeScript** | Type-safe utility libraries & a small OOP domain | ~3 weeks |
| 2 | **Build Scalable Backends** | Raw Node HTTP API → Express CRUD → Layered REST API | ~3 weeks |
| 3 | **Design & Optimize Databases** | A production 3NF PostgreSQL schema from a written brief | ~3 weeks |
| 4 | **Build a Complete SaaS Product** | Full multi-tenant SaaS: backend (Part A) + frontend (Part B) | ~6 weeks |
| 5 | **Ship Software to Production** | Docker, Docker Compose, NGINX, CI/CD, VPS deploy | ~2 weeks |
| 6 | **Build AI-Powered Systems** | LLM chat API, RAG over your own docs, AI agents, full AI SaaS integration | ~6 weeks |
| — | **Agentic AI Masterclass (Live)** | Learn to *use* AI coding agents like Claude Code / OpenCode | continuous |
| — | **Reward Courses** | System Design, NestJS, Testing, AWS Deployment | parallel track |

---

## Folder layout

```
curriculum/
├── README.md                    <- you are here
├── 00-orientation.md            <- how to think like a senior engineer in 2026
├── mission-01-typescript/
│   ├── module-01-typescript-foundations.md
│   ├── module-02-advanced-types.md
│   ├── module-03-oop-in-typescript.md
│   └── assignment-01.md
├── mission-02-scalable-backends/
│   ├── module-05-nodejs-runtime.md
│   ├── module-06-nodejs-core-modules.md
│   ├── module-07-express-rest-api.md
│   ├── module-08-postgresql-essentials.md
│   ├── module-09-clean-architecture.md
│   ├── assignment-02.md
│   └── projects/
│       ├── project-1-raw-node-notes-api.md
│       └── project-2-express-pg-layered-api.md
├── mission-03-databases/
│   ├── module-11-relational-model.md
│   ├── module-12-normalization.md
│   ├── module-13-postgres-fundamentals.md
│   ├── module-14-sql-deep-dive.md
│   ├── module-15-joins-and-aggregations.md
│   ├── module-16-advanced-queries.md
│   └── assignment-03.md
├── mission-04-saas-product/
│   ├── part-a-backend/
│   │   ├── module-18-drizzle-orm.md
│   │   ├── module-19-drizzle-relations.md
│   │   ├── module-20-saas-multi-tenant-design.md
│   │   ├── module-21-project-bootstrap-and-validation.md
│   │   ├── module-22-auth-jwt.md
│   │   ├── module-23-rbac-authorization.md
│   │   ├── module-24-workspaces-teams.md
│   │   ├── module-25-data-and-file-management.md
│   │   ├── module-26-search-filter-pagination.md
│   │   ├── module-27-production-backend.md
│   │   └── assignment-04.md
│   └── part-b-frontend/
│       ├── module-29-nextjs-fundamentals.md
│       ├── module-30-server-components-and-data.md
│       ├── module-31-frontend-auth.md
│       ├── module-32-dashboard-layout.md
│       ├── module-33-forms-and-file-upload.md
│       ├── module-34-tables-search-pagination.md
│       ├── module-35-charts-and-analytics.md
│       ├── module-36-frontend-architecture.md
│       └── assignment-05.md
├── mission-05-production/
│   ├── module-38-docker-intro.md
│   ├── module-39-images-and-containers.md
│   ├── module-40-volumes-and-data.md
│   ├── module-41-docker-networking.md
│   ├── module-42-multi-container-apps.md
│   ├── module-43-docker-compose.md
│   └── module-44-vps-deploy-cicd.md
├── mission-06-ai-systems/
│   ├── part-a-llm-rag-agents/
│   │   ├── module-45-llm-fundamentals.md
│   │   ├── module-46-production-chat-api.md
│   │   ├── module-47-langchain.md
│   │   ├── module-48-embeddings-and-vector-stores.md
│   │   ├── module-49-pgvector.md
│   │   ├── module-50-rag-system.md
│   │   ├── module-51-langgraph-agents.md
│   │   ├── module-52-advanced-agents.md
│   │   └── assignment-06.md
│   └── part-b-integration/
│       ├── module-54-service-to-service.md
│       ├── module-55-api-gateway.md
│       ├── module-56-streaming-workers-caching.md
│       ├── module-57-frontend-ai-features.md
│       └── module-58-agent-dashboard.md
├── live-agentic-ai-masterclass.md
├── reward-courses/
│   ├── reward-01-system-design.md
│   ├── reward-02-nestjs.md
│   ├── reward-03-testing-and-quality.md
│   └── reward-04-aws-deployment.md
└── capstone/
    └── multi-tenant-ai-saas.md   <- the Advanced Full-Stack Multi-Tenant AI SaaS Product
```

---

## How each lesson is structured

Every lesson `.md` file follows the same template so you can navigate quickly:

1. **Why this matters** — the problem in the real world.
2. **Learning objectives** — exactly what you'll be able to do after.
3. **Conceptual model** — the mental model a senior engineer uses.
4. **Code along** — minimal, runnable examples. No copy-paste walls.
5. **Common mistakes** — the bugs I see in every cohort.
6. **Assignment / Exercise** — small, focused practice.
7. **AI Industry Note (2026)** — how this topic is actually moving, and what to learn next.

---

## A note from your instructor

I've been writing TypeScript professionally since the 1.x days, deploying Node backends long before "DevOps" was a job title, and building AI products since the first wave of GPT APIs in 2023. The honest truth about the 2026 industry:

- **AI does not replace engineers** — it replaces *tasks*. The engineers who win are the ones who can architect systems where AI does the 80% of well-understood work, and a human designs the 20% that matters (data models, security boundaries, eval harnesses, multi-tenant isolation, on-call runbooks).
- **The bottleneck moved.** It used to be "writing the code." Now it's "deciding what the code should do, validating that it works, and operating it safely." This curriculum is built around that shift.
- **TypeScript + Postgres + Next.js + Docker + LLM APIs** is the most defensible stack you can learn right now. Everything we do is grounded in those tools.

Treat this like a 6-month apprenticeship. Show up, do the work, and at the end of it you will not just "know" software development — you will be able to *ship* it.

Let's go.

— Your instructor
