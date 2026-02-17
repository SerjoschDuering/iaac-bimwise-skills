---
name: cloudflare
description: >
  This skill should be used when the user asks to "deploy to cloudflare", "set up cloudflare workers",
  "create a fullstack app on cloudflare", "set up D1 database", "configure R2 storage",
  "deploy python workers", "set up wrangler", "add edge functions", "cloudflare CI/CD",
  "serverless deployment", "set up cloudflare pages", "debug cloudflare worker",
  "cloudflare auth", "D1 migrations", "wrangler deploy", "what can I build on cloudflare",
  "help me start a project", or mentions Cloudflare services (Workers, D1, R2, KV, Queues,
  Turnstile, Hyperdrive). Covers JS/TS and Python backends, static frontends, databases,
  object storage, auth, secrets, GitHub Actions, debugging, project ideas, and architecture
  best practices. Primary audience: AI coding agents building on behalf of students.
  Contains executable patterns, not just documentation. Suitable for complete beginners
  with no deployment experience — students make architecture decisions, the AI executes.
version: 0.1.0
---

# Cloudflare Fullstack Deployment

Deploy fullstack web applications on Cloudflare's edge network. Everything runs at the edge
(300+ data centers), scales to zero, and has a generous free tier requiring no credit card (except R2, which is free but requires a credit card on file).

## Quick Start

```bash
# 1. Sign up at https://dash.cloudflare.com (free, no credit card)
# 2. Install CLI
npm install -g wrangler
# 3. Authenticate (opens browser)
wrangler login
# 4. Scaffold a project
npm create cloudflare@latest my-app
# 5. Run locally
cd my-app && wrangler dev
# 6. Deploy globally
wrangler deploy
```

## Architecture Overview

```
┌──────────────────────────────────────────────────┐
│              Cloudflare Edge (300+ DCs)           │
│                                                    │
│  ┌────────────┐  ┌────────────┐  ┌─────────────┐  │
│  │  Workers    │  │  Static    │  │  Workers AI  │  │
│  │  (API/SSR)  │  │  Assets    │  │  (inference) │  │
│  └─────┬──────┘  └────────────┘  └─────────────┘  │
│        │                                           │
│  ┌─────┴──────┐  ┌────────────┐  ┌─────────────┐  │
│  │  D1 (SQL)  │  │  R2 (S3)   │  │  KV (cache)  │  │
│  └────────────┘  └────────────┘  └─────────────┘  │
└──────────────────────────────────────────────────┘
```

## Service Decision Matrix

| Need | Service | Free Tier |
|------|---------|-----------|
| API / serverless functions | **Workers** | 100K req/day |
| Frontend SPA or SSR | **Workers + Static Assets** | Unlimited bandwidth |
| Relational database (SQL) | **D1** (SQLite) | 5M reads/day, 5GB |
| File/image/video storage | **R2** (S3-compatible) | 10GB, zero egress (credit card required) |
| Sessions, config, cache | **KV** (key-value) | 100K reads/day |
| User authentication | **Better Auth** (OSS) | Free |
| Background jobs | **Queues** | 10K ops/day |
| Existing Postgres/MySQL | **Hyperdrive** (proxy) | Included (100K queries/day free) |
| AI inference | **Workers AI** | 10K neurons/day, no credit card, no API key — just `[ai]` binding |
| Bot/spam protection | **Turnstile** | 1M req/month |

## Dashboard vs CLI

Almost everything is CLI-only via `wrangler`. The dashboard is only needed for:

- **Account signup** (one-time) at dash.cloudflare.com
- **API tokens for CI/CD** at Profile > API Tokens
- **Custom domain setup** at Workers > Settings > Domains
- **Monitoring / analytics** for logs and metrics

All resource creation (D1, R2, KV), deployments, migrations, and secrets are handled by `wrangler`.

## Recommended Stacks

### JavaScript/TypeScript (Production-ready)

| Layer | Tool |
|-------|------|
| API | Hono (lightweight, Workers-native, fully typed) |
| ORM | Drizzle (D1 support, auto-generates Zod validators) |
| Frontend | React + Vite (or Astro, SvelteKit for SSR) |
| Routing | TanStack Router (file-based) |
| Server state | TanStack Query |
| Client state | Zustand |
| Auth | Better Auth (D1 + Drizzle + Hono adapter) |

### Python (Open Beta)

| Layer | Tool |
|-------|------|
| API | FastAPI (runs on Workers via Pyodide WASM) |
| CLI | `pywrangler` (wraps wrangler, uses `uv` for deps) |
| Note | Flask/Django NOT supported - only async frameworks |

## Key Architecture Rules

1. **Max 300-400 lines per file.** Split aggressively.
2. **Domain-driven folders.** Each feature is a self-contained directory.
3. **No hardcoding.** Config in env vars or constants files.
4. **Rich comments.** Explain the *why*, not just the *what* - target beginners.
5. **Log everything.** Use structured logging so debugging is possible.
6. **Zustand early.** Introduce state management before complexity hits.
7. **Validate all inputs.** Never trust client-sent data.
8. **ARCHITECTURE.md at root — MANDATORY.** Once the user decides on an architecture,
   immediately create `ARCHITECTURE.md` with: overview, mermaid diagram, services table,
   folder structure, data flow, and domain map. **Keep it up to date** — if the architecture
   changes, update this file FIRST. See `references/architecture-bestpractices.md` for template.

## Reference Docs

| Read when... | File |
|---|---|
| Brand new, no idea what any of this means | `references/beginner-onboarding.md` |
| Need project ideas or help deciding what to build | `references/beginner-onboarding.md` |
| First time using Cloudflare (ready to set up) | `references/getting-started.md` |
| **Building a fullstack app from scratch (start here)** | **`references/tutorial-todo-app.md`** |
| Choosing or configuring compute (Workers, Python, containers) | `references/compute.md` |
| Setting up database, file storage, cache, or queue | `references/database-storage.md` |
| Adding login/signup, bot protection, or access control | `references/auth-security.md` |
| Deploying via CLI, GitHub Actions, or managing secrets | `references/deployment-cicd.md` |
| Writing JS/TS backend + frontend (Hono, Drizzle, React) | `references/js-patterns.md` |
| Writing a Python backend (FastAPI on Workers) | `references/python-patterns.md` |
| Structuring projects, state management, security | `references/architecture-bestpractices.md` |
| Using Workers AI (LLMs, embeddings, image gen) | `references/workers-ai.md` |
| Something broke and need to debug | `references/debugging-logging.md` |

## Work in Progress — Self-Correcting Skill

This skill is **actively evolving**. If during execution any reference file contains
outdated APIs, incorrect imports, broken commands, or patterns that don't work:

1. **Fix it immediately** in the reference file, or
2. **Ask the user** if they'd like the skill files updated with the correction.

Don't silently work around inaccuracies — surface them and fix the source so future
sessions benefit. Treat every execution as a chance to improve the skill.

## Agent Execution Flow

This skill is designed for AI coding agents to **execute**, not just reference.
Every reference file contains copy-pasteable code and runnable commands.

**Step-by-step for the AI agent:**

1. **Understand the user's goal.** If they have no idea, read `beginner-onboarding.md` — help them
   pick a project from the ideas list. Ask what interests them, what they want to learn.

2. **Discuss architecture.** Once a project is chosen, discuss which services are needed
   (use the Service Decision Matrix above). Let the user make the call on stack/features.

3. **Document architecture IMMEDIATELY.** Create `ARCHITECTURE.md` at project root with:
   - Overview paragraph
   - Mermaid diagram (`references/architecture-bestpractices.md` has template)
   - Services table (service, purpose, binding name)
   - Folder structure
   - Data flow description
   This is the source of truth. **Update it every time the architecture changes.**

4. **Build the app.** Follow `tutorial-todo-app.md` as the base pattern, adapting for the
   user's chosen project. Pull specific patterns from:
   - `js-patterns.md` / `python-patterns.md` — code patterns
   - `database-storage.md` — D1, R2, KV, Queues setup
   - `auth-security.md` — Better Auth, JWT, Turnstile
   - `architecture-bestpractices.md` — structure, security, state management

5. **Deploy.** Follow `deployment-cicd.md` for CLI deploy + GitHub Actions setup.

6. **Keep ARCHITECTURE.md current.** After adding any new service, domain, or changing
   data flow — update `ARCHITECTURE.md` and its mermaid diagram before moving on.
