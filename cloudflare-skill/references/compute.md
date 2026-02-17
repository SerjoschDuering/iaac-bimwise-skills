# Edge Compute: Workers, Static Assets & Containers

## What Are Workers?

Serverless functions running on Cloudflare's global network (300+ data centers).
Code runs in V8 isolates (not containers) - near-zero cold starts for JS/TS
(~5ms isolate startup, effectively zero with pre-warming), ~1 second for Python. Code executes close to the user, everywhere in the world.

**Official Docs:** https://developers.cloudflare.com/workers/

---

## Workers (JavaScript / TypeScript)

The primary and most mature way to run backend code on Cloudflare.

### Pricing

| | Free | Paid ($5/mo) |
|---|---|---|
| Requests | 100,000/day | 10M/month incl, then $0.30/M |
| CPU time | 10ms/request | 30M CPU-ms/month incl, then $0.02/M |
| Script size | 3MB compressed | 10MB compressed |
| Memory | 128MB | 128MB |
| Workers/account | 100 | 500 |
| Subrequests | 50/request | 1,000/request |
| Cron triggers | 5 | 250 |
| Egress | **Free** | **Free** |

**Docs:** https://developers.cloudflare.com/workers/platform/pricing/
**Limits:** https://developers.cloudflare.com/workers/platform/limits/

### Minimal Example

```typescript
// src/index.ts
export default {
  async fetch(request, env, ctx) {
    return new Response('Hello World!')
  }
}
```

### With Hono (Recommended Framework)

```typescript
import { Hono } from 'hono'
import { cors } from 'hono/cors'

type Bindings = {
  DB: D1Database
  BUCKET: R2Bucket
  CACHE: KVNamespace
}

const app = new Hono<{ Bindings: Bindings }>()
app.use('/*', cors())

app.get('/', (c) => c.json({ status: 'running' }))
app.get('/users', async (c) => {
  const db = createDb(c.env.DB)
  const users = await db.select().from(usersTable).all()
  return c.json({ data: users })
})

export default app
```

### Wrangler Config

```toml
name = "my-api"
main = "src/index.ts"
compatibility_date = "2025-01-01"
compatibility_flags = ["nodejs_compat"]

[vars]
ENVIRONMENT = "production"
```

**Docs:** https://developers.cloudflare.com/workers/wrangler/configuration/

### Key CLI Commands

```bash
wrangler dev                  # Local dev server (localhost:8787)
wrangler dev --remote         # Local dev with real CF bindings (REQUIRED for AI, Queues consumer)
wrangler deploy               # Deploy to production
wrangler deploy --env preview # Deploy to preview environment
wrangler tail                 # Live-stream logs
wrangler whoami               # Check auth status
```

> **When `--remote` is required:** Workers AI has NO local emulation — `wrangler dev`
> without `--remote` returns unhelpful errors. Queue consumers also only fire in
> production (not during local dev). Always use `--remote` when testing AI or Queues.

**Docs:** https://developers.cloudflare.com/workers/wrangler/commands/

---

## Workers (Python)

**Status: Open Beta** (since April 2024, actively developed)

Runs CPython compiled to WebAssembly via Pyodide. Only async frameworks work (FastAPI).
Flask/Django are NOT supported.

**Docs:** https://developers.cloudflare.com/workers/languages/python/

### Pricing

Same as JS Workers (shares the same limits/quotas).

### How It Works

1. Your Python code is compiled to WASM via Pyodide
2. At deploy time, Cloudflare snapshots your imports into memory
3. On cold start, snapshot is restored (~1 sec vs ~10 sec without)
4. All CF bindings (D1, R2, KV, etc.) accessible via JS FFI bridge

**Docs:** https://developers.cloudflare.com/workers/languages/python/how-python-workers-work/

### Setup

```bash
uv tool install workers-py         # Install pywrangler
uv run pywrangler init my-api      # Scaffold project
uv run pywrangler dev              # Local dev
uv run pywrangler deploy           # Deploy
uv run pywrangler types            # Generate type stubs for IDE
```

### Quick Example

```python
from workers import WorkerEntrypoint, Response

class Default(WorkerEntrypoint):
    async def fetch(self, request):
        return Response("Hello from Python!")
```

For full FastAPI examples, D1/R2/KV usage from Python, and detailed setup:
see **`references/python-patterns.md`**

### Key Constraints

- Only async frameworks (FastAPI). Flask/Django do NOT work.
- Packages with native C extensions must be in Pyodide's supported list.
- 128MB memory limit includes Python runtime + your code.

**Supported packages:** https://developers.cloudflare.com/workers/languages/python/packages/

---

## Workers AI — Edge Inference

Run LLMs, embeddings, image generation, and more directly from Workers. No API key, no credit card, no external service — just a wrangler binding.

### Pricing

| | Free | Paid |
|---|---|---|
| Neurons | 10,000/day | $0.011/1,000 neurons |
| Token pricing (LLM) | Same free allowance | Llama 3.1 8B: ~$0.03/M input, ~$0.20/M output |
| Credit card | **Not required** | Not required |
| API key | **Not required** — `env.AI` binding | Same |

### Wrangler Config

```toml
[ai]
binding = "AI"
```

### Usage

```typescript
const response = await c.env.AI.run('@cf/meta/llama-3.1-8b-instruct', {
  messages: [
    { role: 'system', content: 'You are a helpful assistant.' },
    { role: 'user', content: userPrompt },
  ],
  max_tokens: 2048,
  temperature: 0.7,
})
const text = response.response // string
```

### Key Models (Feb 2026)

| Model | ID | Use case |
|---|---|---|
| Llama 4 Scout 17B | `@cf/meta/llama-4-scout-17b-16e-instruct` | Best quality, multimodal |
| Llama 3.3 70B | `@cf/meta/llama-3.3-70b-instruct-fp8-fast` | High quality text gen |
| Llama 3.1 8B | `@cf/meta/llama-3.1-8b-instruct` | Fast, cheap, good for most tasks |
| Llama 3.2 11B Vision | `@cf/meta/llama-3.2-11b-vision-instruct` | Image understanding |
| BGE Base EN | `@cf/baai/bge-base-en-v1.5` | Text embeddings |

Full catalog: https://developers.cloudflare.com/workers-ai/models/

### Local Dev

Workers AI has **no local emulation**. You must use `wrangler dev --remote` to test AI calls. Without `--remote`, AI calls return unhelpful errors.

**Docs:** https://developers.cloudflare.com/workers-ai/

---

## Static Assets (Workers)

Workers can serve static files (HTML, CSS, JS, images) directly.
This is the **recommended approach for new projects** (replaces Pages).

```toml
# wrangler.toml
name = "my-fullstack-app"
main = "src/index.ts"
compatibility_date = "2025-01-01"

[assets]
directory = "./public"    # Serve static files from ./public/
```

- Unlimited bandwidth, free
- Static assets served first, then falls through to Worker code
- Can combine SPA frontend + API backend in a single Worker

**Docs:** https://developers.cloudflare.com/workers/static-assets/

---

## Pages (Deprecated April 2025 - Migrating to Workers)

Static site hosting with Git integration. Formally deprecated in April 2025.
Existing Pages projects continue to work and are not being shut down, but Pages
is no longer receiving new features. All new projects should use Workers + Static Assets.
Cloudflare provides migration guides for moving Pages projects to Workers.

Use Pages if: you want zero-config Git deploy for a simple static site.
Use Workers for everything else.

| | Free | Paid |
|---|---|---|
| Builds | 500/month | 5,000/month |
| Bandwidth | Unlimited | Unlimited |
| Preview deploys | Unlimited | Unlimited |

**Docs:** https://developers.cloudflare.com/pages/
**Migration guide:** https://developers.cloudflare.com/workers/static-assets/migration-guides/migrate-from-pages/

---

## Containers (Public Beta)

Full Docker container support on Cloudflare. Scale-to-zero.

Use when Workers can't do the job: native dependencies, long-running processes,
existing Docker images.

| Type | vCPU | RAM | Disk |
|------|------|-----|------|
| lite | 1/16 | 256 MiB | 2 GB |
| basic | 1/4 | 1 GiB | 4 GB |
| standard-1 | 1/2 | 4 GiB | 8 GB |
| standard-2 | 1 | 6 GiB | 12 GB |
| standard-3 | 2 | 8 GiB | 16 GB |
| standard-4 | 4 | 12 GiB | 20 GB |

**Docs:** https://developers.cloudflare.com/containers/
**Pricing:** https://developers.cloudflare.com/containers/pricing/

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Workers overview | https://developers.cloudflare.com/workers/ |
| Workers pricing | https://developers.cloudflare.com/workers/platform/pricing/ |
| Workers limits | https://developers.cloudflare.com/workers/platform/limits/ |
| Wrangler CLI | https://developers.cloudflare.com/workers/wrangler/ |
| Wrangler commands | https://developers.cloudflare.com/workers/wrangler/commands/ |
| Wrangler config | https://developers.cloudflare.com/workers/wrangler/configuration/ |
| Environments | https://developers.cloudflare.com/workers/wrangler/environments/ |
| Python Workers | https://developers.cloudflare.com/workers/languages/python/ |
| Python packages | https://developers.cloudflare.com/workers/languages/python/packages/ |
| FastAPI on Workers | https://developers.cloudflare.com/workers/languages/python/packages/fastapi/ |
| Static assets | https://developers.cloudflare.com/workers/static-assets/ |
| Pages | https://developers.cloudflare.com/pages/ |
| Containers | https://developers.cloudflare.com/containers/ |
| Framework guides | https://developers.cloudflare.com/workers/framework-guides/ |
| Getting started | https://developers.cloudflare.com/workers/get-started/guide/ |
