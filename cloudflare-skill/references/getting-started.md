# Getting Started with Cloudflare

## Prerequisites

Before starting, make sure you have:

1. **Node.js 18+** installed — download from [nodejs.org](https://nodejs.org/) (LTS version). This also installs `npm`.
   - Verify: `node --version` (should print `v18.x.x` or higher)
2. **A code editor** — [VS Code](https://code.visualstudio.com/) is recommended (free)
3. **A terminal** — Terminal (Mac: Applications > Utilities > Terminal) or PowerShell (Windows: search "PowerShell")
4. **Git** (optional, needed for CI/CD later) — [git-scm.com](https://git-scm.com/)

> **Package manager note:** This guide uses `npm` (comes with Node.js). All commands also work with `bun` — see [bun.sh](https://bun.sh/) if you prefer a faster alternative. Stick with one throughout.

## 1. Create a Cloudflare Account

1. Go to [dash.cloudflare.com/sign-up](https://dash.cloudflare.com/sign-up)
2. Sign up with email or SSO (Google/Apple)
3. **No credit card required** for free tier
4. Free tier is very generous - enough to build and run real apps

## 2. Install Wrangler CLI

```bash
# Via npm (recommended)
npm install -g wrangler

# Or via bun
bun add -g wrangler

# Verify
wrangler --version
```

**System requirements:** macOS 13.5+, Windows 11, Linux (glibc 2.35+), Node.js 16.17+

## 3. Authenticate

```bash
wrangler login          # Opens browser for OAuth
wrangler whoami         # Verify you're logged in
```

**What happens:** Your browser opens a Cloudflare authorization page. Click "Allow" and return to your terminal. You should see "Successfully logged in."

> If the browser doesn't open (WSL, SSH, headless Linux), use `wrangler login --browser false` and paste the URL manually.

This creates a token stored at `~/.wrangler/config/default.toml` (your home directory).

For CI/CD, create an API token instead:
1. Go to [dash.cloudflare.com/profile/api-tokens](https://dash.cloudflare.com/profile/api-tokens)
2. Click "Create Token"
3. Use template **"Edit Cloudflare Workers"** (covers Workers, Pages, D1, R2, KV)
4. Save the token - you'll need it for GitHub Actions

## 4. Scaffold a Project

### Option A: Use C3 (create-cloudflare)
```bash
npm create cloudflare@latest my-app
# Interactive prompts:
# - Choose "Hello World" for basics, or pick a framework
# - TypeScript? Yes
# - Deploy now? Yes/No
```

### Option B: Manual setup
```bash
mkdir my-app && cd my-app
npm init -y
npm install hono
npm install -D wrangler @cloudflare/workers-types
```

Create `wrangler.toml`:
```toml
name = "my-app"
main = "src/index.ts"
compatibility_date = "2025-01-01"
compatibility_flags = ["nodejs_compat"]
```

Create `src/index.ts`:
```typescript
import { Hono } from 'hono'
const app = new Hono()
app.get('/', (c) => c.json({ hello: 'world' }))
export default app
```

### Option C: Python Worker
```bash
# Install pywrangler
pip install workers-py
# Or with uv (recommended)
uv tool install workers-py

# Initialize
uv run pywrangler init my-python-app
cd my-python-app
```

## 5. Local Development

```bash
# JS/TS
wrangler dev                    # Starts at http://localhost:8787
wrangler dev --remote           # Uses real CF bindings (D1, KV, R2) — skip for now

# Python
uv run pywrangler dev
```

**What you see:** Terminal prints something like `Ready on http://localhost:8787`. Open that URL in your browser — you should see your app's response (e.g., `{"hello":"world"}`).

Hot reload is built-in. Changes to your code restart the Worker automatically.

## 6. Deploy

```bash
# JS/TS
wrangler deploy                 # Deploys to https://my-app.<your-subdomain>.workers.dev

# Python
uv run pywrangler deploy
```

**What you see:** Terminal prints a URL like `https://my-app.your-name.workers.dev`. Copy-paste it into your browser — your app is live! The subdomain is your Cloudflare account name (set during signup or auto-assigned).

Your app is now live on Cloudflare's edge network (300+ locations worldwide).

## 7. Add Services

```bash
# Create a D1 database
wrangler d1 create my-database
# Output: database_id = "xxxx-xxxx-xxxx"  <-- add this to wrangler.toml

# Create an R2 bucket
wrangler r2 bucket create my-files

# Create a KV namespace
wrangler kv namespace create MY_CACHE
# Output: id = "xxxx"  <-- add this to wrangler.toml
```

Then add bindings to `wrangler.toml`:
```toml
[[d1_databases]]
binding = "DB"
database_name = "my-database"
database_id = "xxxx-xxxx-xxxx"

[[r2_buckets]]
binding = "BUCKET"
bucket_name = "my-files"

[[kv_namespaces]]
binding = "CACHE"
id = "xxxx"
```

Access in code via `c.env.DB`, `c.env.BUCKET`, `c.env.CACHE` (where `c` is the Hono request context — see `references/js-patterns.md` for full examples).

## 8. Custom Domain (Optional)

```bash
# Add custom domain via dashboard or:
wrangler deploy  # deploys to *.workers.dev
# Then in Cloudflare dashboard: Workers > your-worker > Settings > Domains & Routes
# Add your custom domain (must be on Cloudflare DNS)
```

## Project Structure (Recommended)

### Simple API
```
my-app/
├── src/
│   ├── index.ts          # Entry point (Hono app)
│   └── domains/
│       └── users/
│           ├── users.routes.ts
│           └── users.schema.ts
├── migrations/           # D1 SQL migrations
├── wrangler.toml
├── package.json
├── tsconfig.json
└── .dev.vars             # Local secrets (gitignored!)
```

### Fullstack (API + SPA)
```
my-app/
├── api/
│   ├── src/index.ts
│   ├── wrangler.toml
│   └── package.json
├── client/
│   ├── src/
│   ├── vite.config.ts
│   ├── wrangler.toml     # Pages config
│   └── package.json
├── package.json          # Workspace root
└── .env                  # Shared env vars
```

### Python API
```
my-python-app/
├── src/
│   └── entry.py          # FastAPI app
├── wrangler.toml
├── pyproject.toml        # Python deps (uv)
└── .dev.vars
```

## Free Tier Limits (What You Get for $0)

| Service | Free Allowance |
|---------|---------------|
| Workers | 100K requests/day, 10ms CPU/request |
| Static Assets | Unlimited bandwidth |
| D1 | 5M reads/day, 100K writes/day, 5GB storage |
| R2 | 10GB storage, 1M writes, 10M reads/month |
| KV | 100K reads/day, 1K writes/day, 1GB |
| Queues | 10K operations/day |
| Workers AI | 10K neurons/day |
| Turnstile | 1M requests/month |
| Hyperdrive | Included (100K queries/day free) |

The **$5/month paid plan** dramatically increases all limits (e.g., 10M Worker requests/month).
