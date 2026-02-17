# Cloudflare Starter Skill

**Deploy fullstack web apps on Cloudflare — guided by your AI coding agent.**

An open skill that teaches any AI coding agent how to build and deploy fullstack applications on Cloudflare's edge network. Free tier, no credit card, scales to zero.

Built for students and beginners. You make the decisions — the AI executes.

Works with: Claude Code, Cursor, Windsurf, Codex, GitHub Copilot, Gemini CLI, Roo Code, and [25+ other tools](https://agentskills.io).

> **This is NOT a project template.** Don't clone this repo and build inside it. This is a *skill* — a knowledge pack you add to your AI coding tool. Your project lives in its own directory. See [How to Install](#how-to-install) below.

---

## What Is a "Skill"?

AI coding agents (Claude Code, Cursor, Copilot, etc.) are powerful, but they have a fundamental limitation: **they can only work with what fits in their context window**. That's the working memory the AI uses during a conversation — typically between 100K and 1M tokens depending on the model.

A real-world codebase, its documentation, and all the knowledge needed to deploy it correctly can easily exceed that limit. And even when things do fit, stuffing everything in at once makes the AI *worse* — irrelevant information competes for attention and degrades output quality.

### The problem

Imagine asking your AI agent to "deploy this app to Cloudflare." Without a skill, the agent has to:

1. Rely on training data that may be months or years out of date
2. Guess at current API patterns, CLI flags, and best practices
3. Search the web and hope it finds the right docs
4. Hold all of that in memory alongside your actual codebase

The result: hallucinated commands, outdated patterns, and a lot of back-and-forth.

### How skills solve it

A **skill** is a folder of markdown files that gives an AI agent specialized, up-to-date knowledge — loaded on demand, not all at once.

```
cloudflare-starter-skill/
├── SKILL.md              ← Agent reads this first (overview, decision matrix, rules)
└── references/
    ├── getting-started.md    ← Loaded only when setting up Cloudflare
    ├── database-storage.md   ← Loaded only when adding a database
    ├── auth-security.md      ← Loaded only when adding login
    └── ...8 more files       ← Each loaded only when relevant
```

This pattern is called **progressive disclosure**:

1. **Startup**: The agent sees only the skill name and description (a few tokens)
2. **Task match**: When the user asks about Cloudflare, the agent loads `SKILL.md` (~170 lines)
3. **Deep dive**: When the agent needs database setup, it loads *just* `database-storage.md`

The agent never holds all 10 reference files in memory at once. It pulls in exactly what it needs, when it needs it — keeping context focused and output quality high.

### Cross-tool compatibility

This skill follows the [**Agent Skills** open standard](https://agentskills.io) — an open format originally developed by Anthropic and now adopted by 25+ tools across the industry.

Every major AI coding tool has its own instruction file convention:

| Tool | Native format | Reads skills? |
|------|--------------|---------------|
| **Claude Code** | `CLAUDE.md` + `SKILL.md` | Native support |
| **GitHub Copilot** | `.github/copilot-instructions.md` | [Via Agent Skills](https://code.visualstudio.com/docs/copilot/customization/agent-skills) |
| **Cursor** | `.cursor/rules/*.mdc` | Via `SKILL.md` |
| **Windsurf** | `.windsurfrules` | Via `SKILL.md` |
| **Codex CLI** | `AGENTS.md` | Via `AGENTS.md` (included) |
| **Gemini CLI** | `GEMINI.md` / `AGENTS.md` | Via `AGENTS.md` (included) |
| **Roo Code / Cline** | `.clinerules` / `AGENTS.md` | Via `AGENTS.md` (included) |

This repo includes both `SKILL.md` (the standard) and `AGENTS.md` (a copy, for tools that look for that filename). No tool-specific config dirs needed.

---

## Why Cloudflare?

There are many ways to deploy a web app. Vercel, Netlify, AWS, GCP, Fly.io, Railway — all solid options. We chose Cloudflare for this skill because it hits a unique sweet spot for students and beginners:

**It's free.** Not "free trial" or "free for 30 days." The free tier includes compute, database, file storage, caching, queues, AI inference, and bot protection — with no credit card required. You can build and deploy a complete fullstack app without spending anything.

**Everything is in one place.** Most providers make you stitch together services from different platforms (e.g., Vercel for frontend + Supabase for database + S3 for files). Cloudflare provides all of it under one account, one CLI, one billing page.

**Everything is CLI-controlled.** A single tool (`wrangler`) handles creating databases, deploying code, managing secrets, running migrations, and tailing logs. No clicking through dashboards. This matters because AI coding agents work through the terminal — a CLI-first platform means the AI can do everything for you.

**It scales to zero.** You pay nothing when nobody's using your app. No idle servers, no minimum charges.

| Need | Cloudflare Service | Free Tier |
|------|-------------------|-----------|
| API / serverless functions | **Workers** | 100K req/day |
| Frontend SPA or SSR | **Workers + Static Assets** | Unlimited bandwidth |
| Relational database (SQL) | **D1** (SQLite) | 5M reads/day, 5GB |
| File/image/video storage | **R2** (S3-compatible) | 10GB, zero egress |
| Sessions, config, cache | **KV** (key-value) | 100K reads/day |
| User authentication | **Better Auth** (OSS library) | Free |
| Background jobs | **Queues** | 10K ops/day |
| AI inference | **Workers AI** | 10K neurons/day |
| Bot/spam protection | **Turnstile** | 1M req/month |

> **Note:** The patterns in this skill (domain-driven structure, progressive disclosure, ARCHITECTURE.md) work with *any* provider. If you later switch to Vercel or AWS, the architectural knowledge transfers.

---

<details>
<summary><h2>Glossary — Plain-English Tech Terms</h2></summary>

New to all of this? Here's every technical term used in this repo, explained like you're not a developer — because you might not be (yet).

### Building & Running

| Term | What it actually means |
|------|----------------------|
| **Frontend** | The part of an app you see and interact with — buttons, forms, pages. It runs in your browser. |
| **Backend** | The invisible part that does the work — processes logins, saves data, talks to the database. Runs on a server (or in our case, at the edge). |
| **Fullstack** | Frontend + backend together. A "fullstack app" is the whole thing, not just one half. |
| **API** (Application Programming Interface) | A set of URLs your frontend calls to talk to the backend. Like a restaurant menu — you don't go into the kitchen, you just order from the menu. Example: `GET /api/todos` returns your todo list. |
| **Database** | Where your app stores information permanently — users, posts, orders, etc. Think: a really organized spreadsheet that your code can read and write to. |
| **ORM** (Object-Relational Mapper) | A tool that lets you talk to the database using your programming language instead of writing raw SQL. Instead of `SELECT * FROM users`, you write `db.select().from(users)`. |
| **Schema** | The structure of your database — what tables exist, what columns they have, what type of data goes in each column. Like a blueprint before you build the house. |
| **Migration** | A file that changes your database structure. Adding a new column? That's a migration. They run in order so every copy of the database ends up the same. |

### Deployment & Infrastructure

| Term | What it actually means |
|------|----------------------|
| **Deploy / Deployment** | Putting your app on the internet so other people can use it. Before deployment, it only runs on your computer. After, it has a public URL anyone can visit. |
| **Server** | A computer that's always on and connected to the internet, waiting to respond to requests. Traditionally you'd rent one and manage it yourself. |
| **Serverless** | You don't manage any servers. You upload your code, and the provider runs it when someone makes a request. You pay per request, not per hour. |
| **Edge** | Instead of running your code in one data center (say, Virginia), it runs in 300+ locations worldwide. Users in Tokyo hit a Tokyo server, users in Berlin hit a Berlin server. Faster for everyone. |
| **CDN** (Content Delivery Network) | A network of servers around the world that caches your static files (images, CSS, JS) close to users. The edge is like a CDN, but for your code too. |
| **Scales to zero** | When nobody is using your app, you pay nothing. No idle servers burning money. Traffic spikes? It scales up automatically. |
| **Environment variables** | Secret settings (API keys, database passwords) stored outside your code. Your code reads them at runtime. This way secrets never end up in your GitHub repo. |
| **Secrets** | Same as environment variables, but specifically for sensitive values (passwords, tokens). Stored encrypted. |

### Tools & Workflow

| Term | What it actually means |
|------|----------------------|
| **CLI** (Command Line Interface) | A text-based way to control software by typing commands. Instead of clicking buttons in a dashboard, you type `wrangler deploy` in your terminal. AI agents love CLIs because they can type commands for you. |
| **Terminal** | The app where you type CLI commands. On Mac it's called Terminal, on Windows it's Command Prompt or PowerShell. |
| **Wrangler** | Cloudflare's CLI tool. One command to create databases, deploy code, manage secrets, run locally, tail logs — everything. |
| **npm** | A package manager for JavaScript. `npm install` downloads libraries your project needs. Think: an app store for code. |
| **Git** | A tool that tracks every change you make to your code. Like version history in Google Docs, but for an entire project. |
| **GitHub** | A website that hosts your Git repositories (projects) online. Also has tools for collaboration, CI/CD, and issue tracking. |
| **CI/CD** (Continuous Integration/Deployment) | Automation that tests and deploys your code every time you push to GitHub. You push code → tests run → if they pass → app deploys. No manual steps. |
| **GitHub Actions** | GitHub's built-in CI/CD system. You write a YAML file describing what should happen on push, and GitHub runs it for you. |

### Cloudflare Services

| Term | What it actually means |
|------|----------------------|
| **Workers** | Cloudflare's serverless functions. Your backend code runs here. Think of each Worker as a tiny server that spins up on demand at the edge. |
| **D1** | Cloudflare's SQL database (built on SQLite). Where you store structured data — users, posts, orders. |
| **R2** | Cloudflare's file storage (S3-compatible). For images, PDFs, videos, user uploads. Zero egress fees — you don't pay when people download files. |
| **KV** (Key-Value store) | A super-fast cache. Store simple data like session tokens or feature flags. Reads are nearly instant globally. |
| **Queues** | For background jobs. Instead of making the user wait while you send an email or process a file, you add a job to the queue and it runs later. |
| **Hyperdrive** | A proxy that makes existing Postgres/MySQL databases faster by caching connections at the edge. Use this when your database is hosted somewhere else. |
| **Workers AI** | Run AI models (text generation, image recognition) directly on Cloudflare's edge. No OpenAI key needed for basic inference. |
| **Turnstile** | Bot protection that replaces CAPTCHAs. Add it to forms to verify real humans without annoying "click all the traffic lights" puzzles. |

### Code Patterns

| Term | What it actually means |
|------|----------------------|
| **SPA** (Single-Page Application) | A web app that loads once and then updates dynamically without full page reloads. Gmail is an SPA. |
| **SSR** (Server-Side Rendering) | The server generates HTML before sending it to the browser. Faster initial load, better for SEO. |
| **Framework** | Pre-built code that gives you structure and shortcuts. React is a frontend framework, Hono is a backend framework. You build *on top of* them. |
| **Routing** | Deciding what to show based on the URL. `/about` shows the about page, `/todos/5` shows todo #5. |
| **State management** | How your app keeps track of things in memory — which user is logged in, what's in the shopping cart, whether a modal is open. |
| **Middleware** | Code that runs *between* receiving a request and sending a response. Common uses: check if the user is logged in, log the request, add security headers. |
| **Token** | A string of random characters that proves who you are. After login, the server gives you a token. You send it with every request so the server knows it's you. |
| **Binding** | Cloudflare's way of connecting a Worker to a service (D1, R2, KV). You declare bindings in `wrangler.toml` and access them in your code via `env.MY_DB`. |

</details>

---

## Architecture at a Glance

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

Everything runs at the edge (300+ data centers). No servers to manage.

---

## What's Inside

| File | What it covers |
|------|----------------|
| [`SKILL.md`](SKILL.md) | Main skill file — quick start, service matrix, architecture rules, agent execution flow |
| [`references/beginner-onboarding.md`](references/beginner-onboarding.md) | Brand new to web dev? Start here. Explains everything from scratch + project ideas |
| [`references/getting-started.md`](references/getting-started.md) | Cloudflare account setup, wrangler install, first deploy |
| [`references/tutorial-todo-app.md`](references/tutorial-todo-app.md) | Build a complete fullstack todo app step-by-step (the best starting point) |
| [`references/compute.md`](references/compute.md) | Workers, Python Workers, containers, runtime limits |
| [`references/database-storage.md`](references/database-storage.md) | D1 (SQL), R2 (files), KV (cache), Queues (jobs) |
| [`references/auth-security.md`](references/auth-security.md) | Better Auth, JWT, Turnstile, access control |
| [`references/deployment-cicd.md`](references/deployment-cicd.md) | CLI deploy, GitHub Actions, secrets management |
| [`references/js-patterns.md`](references/js-patterns.md) | Hono API, Drizzle ORM, React + Vite frontend patterns |
| [`references/python-patterns.md`](references/python-patterns.md) | FastAPI on Workers via Pyodide WASM |
| [`references/architecture-bestpractices.md`](references/architecture-bestpractices.md) | Project structure, state management, security, ARCHITECTURE.md template |
| [`references/debugging-logging.md`](references/debugging-logging.md) | Structured logging, error tracking, wrangler tail |

---

## How to Install

A skill is **not** something you install globally or run as a command. It's a folder of markdown files that you place inside your project so your AI coding tool can discover and read them.

**The workflow:**

```bash
# 1. Create your project (this is YOUR workspace — you build here)
npm create cloudflare@latest my-app
cd my-app

# 2. Add the skill to your project (the AI reads from here — you don't touch these files)
git clone https://github.com/SerjoschDuering/cloudflare-starter-skill.git .claude/skills/cloudflare

# 3. Open your AI coding tool and start building
# "I want to build a todo app with a database"
# The agent will automatically read the skill files and guide you.
```

That's it. The skill lives in `.claude/skills/cloudflare/` inside your project. Your AI tool auto-discovers it.

### Tool-specific paths

Different tools look for instruction files in different places. Clone the skill into the path your tool expects:

| Tool | Clone command |
|------|--------------|
| **Claude Code** | `git clone https://github.com/SerjoschDuering/cloudflare-starter-skill.git .claude/skills/cloudflare` |
| **Cursor** | `git clone https://github.com/SerjoschDuering/cloudflare-starter-skill.git .cursor/skills/cloudflare` |
| **Windsurf** | `git clone https://github.com/SerjoschDuering/cloudflare-starter-skill.git .windsurf/skills/cloudflare` |
| **Codex CLI** | `git clone https://github.com/SerjoschDuering/cloudflare-starter-skill.git .codex/skills/cloudflare` |
| **Gemini CLI** | `git clone https://github.com/SerjoschDuering/cloudflare-starter-skill.git .gemini/skills/cloudflare` |
| **Roo Code / Cline** | `git clone https://github.com/SerjoschDuering/cloudflare-starter-skill.git .roo/skills/cloudflare` |
| **Any other tool** | `git clone https://github.com/SerjoschDuering/cloudflare-starter-skill.git .skills/cloudflare` |

> **Tip:** Add the skill directory to your project's `.gitignore` so it doesn't get committed into your own repo:
> ```bash
> echo ".claude/skills/cloudflare" >> .gitignore
> ```

> **Alternative — git submodule:** If you *want* the skill tracked in your repo (e.g., for team projects), use a submodule instead:
> ```bash
> git submodule add https://github.com/SerjoschDuering/cloudflare-starter-skill.git .claude/skills/cloudflare
> ```

---

## Quick Start

Once the skill is loaded, just tell your AI agent what you want to build:

> "I want to build a todo app on Cloudflare"

> "Help me deploy a fullstack React app with a database"

> "I have no idea what to build — give me project ideas"

The agent reads the right reference files and guides you through every step — from account setup to production deployment.

---

## Recommended Stacks

### JavaScript/TypeScript

| Layer | Tool |
|-------|------|
| API | Hono (Workers-native, fully typed) |
| ORM | Drizzle (D1 support, generates Zod validators) |
| Frontend | React + Vite |
| Routing | TanStack Router (file-based) |
| Server state | TanStack Query |
| Client state | Zustand |
| Auth | Better Auth |

### Python

| Layer | Tool |
|-------|------|
| API | FastAPI (runs on Workers via Pyodide WASM) |
| CLI | `pywrangler` (wraps wrangler, uses `uv`) |

---

## Further Reading

### Agent Skills Standard
- [Agent Skills specification](https://agentskills.io/specification) — the open standard this skill follows
- [Agent Skills GitHub repo](https://github.com/agentskills/agentskills) — spec source code
- [Anthropic: Equipping agents with Agent Skills](https://www.anthropic.com/news/agent-skills) — original announcement
- [Anthropic: Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — why context management matters

### Tool Documentation
- [Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices)
- [VS Code Agent Skills support](https://code.visualstudio.com/docs/copilot/customization/agent-skills)
- [GitHub Copilot custom instructions](https://docs.github.com/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot)
- [Codex CLI — AGENTS.md guide](https://developers.openai.com/codex/guides/agents-md/)
- [Gemini CLI docs](https://developers.google.com/gemini-code-assist/docs/gemini-cli)

### Skill Directories & Collections
- [skills.sh](https://skills.sh/) — open directory of agent skills
- [Anthropic official skills](https://github.com/anthropics/skills)
- [awesome-agent-skills](https://github.com/skillmatic-ai/awesome-agent-skills) — curated list

---

## Work in Progress

This skill is **actively evolving**. Cloudflare ships fast — APIs change, new services launch, CLI flags get deprecated. The reference files may not always be perfectly up to date.

That's by design. Every AI agent that uses this skill is encouraged to **fix inaccuracies on the spot** — update a command, correct an import, adjust for a new API version. If you (or your agent) spot something wrong, you're improving it for everyone who comes after.

## Contributing

PRs and issues welcome. Common contributions:

- **Fix outdated patterns** — a CLI flag changed, an API was renamed
- **Add missing services** — Cloudflare launched something new
- **Improve explanations** — a reference file was confusing or incomplete
- **Report broken commands** — something that worked last month doesn't anymore

---

## License

MIT

---

Built with the [Agent Skills open standard](https://agentskills.io)
