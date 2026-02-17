# Deployment, CI/CD & Secrets

## CLI Deployment

### Single Worker
```bash
wrangler deploy                     # Production
wrangler deploy --env preview       # Preview environment
```

### Python Worker
```bash
uv run pywrangler deploy
```

### Fullstack (API Worker + Frontend)
```bash
# 1. Migrate database
wrangler d1 migrations apply my-db --remote

# 2. Deploy API
wrangler deploy
# Output: https://my-api.username.workers.dev

# 3. Build frontend (NO env var needed if using Pages Functions proxy — see below)
npm run build

# 4. Deploy frontend (wrangler auto-detects functions/ directory)
wrangler pages deploy ./dist --project-name=my-client --commit-dirty=true
```

### Pages Functions Proxy (RECOMMENDED for API routing)

When frontend (Pages) and API (Worker) are separate, the frontend uses relative
paths (`/api/...`). You need a proxy. **DO NOT use `_redirects` with status 200 —
Cloudflare Pages does NOT support proxy mode unlike Netlify.** Use Pages Functions instead:

```typescript
// client/functions/api/[[path]].ts — proxies /api/* to Workers API
const API_ORIGIN = 'https://my-api.username.workers.dev'

export const onRequest: PagesFunction = async ({ request }) => {
  const url = new URL(request.url)
  const target = new URL(url.pathname + url.search, API_ORIGIN)
  const headers = new Headers(request.headers)
  headers.set('Host', new URL(API_ORIGIN).host)
  return fetch(target.toString(), {
    method: request.method,
    headers,
    body: request.method !== 'GET' && request.method !== 'HEAD' ? request.body : undefined,
  })
}
```

Create matching files for each route prefix that needs proxying:
- `functions/api/[[path]].ts` — proxies `/api/*`
- `functions/auth/[[path]].ts` — proxies `/auth/*`

The `functions/` directory must be at the same level as `dist/` (in the project root,
not inside dist). `wrangler pages deploy` auto-detects and bundles them.

### Provision All Resources from CLI
```bash
wrangler d1 create my-db                              # Database
wrangler r2 bucket create my-files                    # Storage
wrangler kv namespace create MY_CACHE                 # Key-value
wrangler secret put API_KEY                           # Secret (prompts for value)
wrangler secret put API_KEY --env preview             # Per-environment secret
wrangler secret list                                  # List secrets (names only)
wrangler secret delete OLD_KEY                        # Delete secret
```

---

## Secrets Management

### Three Levels of Configuration

| Type | Where | Visible? | Use For |
|------|-------|----------|---------|
| `[vars]` in wrangler.toml | In git | Yes, in dashboard + code | Non-sensitive (ENVIRONMENT, APP_NAME) |
| `wrangler secret put` | CF encrypted store | No (write-only after set) | API keys, DB passwords, JWT secrets |
| `.dev.vars` | Local file, gitignored | Local only | Development secrets |

**Docs:** https://developers.cloudflare.com/workers/configuration/secrets/
**Env vars docs:** https://developers.cloudflare.com/workers/configuration/environment-variables/

### Production Secrets

```bash
# Set (interactive, prompts for value - value never in shell history)
wrangler secret put DATABASE_URL

# Set per-environment
wrangler secret put DATABASE_URL --env preview

# Bulk set from JSON
echo '{"KEY1":"val1","KEY2":"val2"}' | wrangler secret bulk

# List (names only, values always hidden)
wrangler secret list

# Delete
wrangler secret delete MY_SECRET
```

### Local Development Secrets

Create `.dev.vars` in your project root (next to wrangler.toml):

```env
# .dev.vars  ← ADD TO .gitignore!
DATABASE_URL=postgres://localhost:5432/dev
API_KEY=dev-key-12345
JWT_SECRET=local-dev-secret
TURNSTILE_SECRET=1x0000000000000000000000000000000AA
```

Per-environment files: `.dev.vars.preview`, `.dev.vars.staging`

**Important:** Use either `.dev.vars` or `.env` but not both. Wrangler reads both formats.

### Accessing Secrets in Code

Secrets and vars are accessed identically - no code changes needed:

```typescript
// JS/TS
app.get('/', (c) => {
  const secret = c.env.API_KEY           // from wrangler secret put
  const config = c.env.ENVIRONMENT       // from [vars] in wrangler.toml
})
```

```python
# Python
class Default(WorkerEntrypoint):
    async def fetch(self, request):
        secret = self.env.API_KEY
        config = self.env.ENVIRONMENT
```

---

## Environments (Preview / Staging / Production)

### wrangler.toml Pattern

```toml
name = "my-api"
main = "src/index.ts"
compatibility_date = "2025-01-01"

# --- Production (default) ---
[[d1_databases]]
binding = "DB"
database_name = "my-db"
database_id = "prod-db-id-here"

[vars]
ENVIRONMENT = "production"

# --- Preview ---
[env.preview]
name = "my-api-preview"         # Deploys as separate Worker

[[env.preview.d1_databases]]
binding = "DB"
database_name = "my-db-preview"
database_id = "preview-db-id-here"

[env.preview.vars]
ENVIRONMENT = "preview"

# --- Staging (optional) ---
[env.staging]
name = "my-api-staging"

[[env.staging.d1_databases]]
binding = "DB"
database_name = "my-db-staging"
database_id = "staging-db-id-here"

[env.staging.vars]
ENVIRONMENT = "staging"
```

**Docs:** https://developers.cloudflare.com/workers/wrangler/environments/

```bash
wrangler dev --env preview          # Local dev with preview config
wrangler deploy --env preview       # Deploy preview
wrangler deploy --env staging       # Deploy staging
wrangler deploy                     # Deploy production
```

---

## GitHub Actions

### Prerequisites

1. Create API token: https://dash.cloudflare.com/profile/api-tokens
   - Use template **"Edit Cloudflare Workers"** (covers Workers, Pages, D1, R2, KV)
2. Add to GitHub repo: Settings → Secrets and variables → Actions:
   - `CLOUDFLARE_API_TOKEN`
   - `CLOUDFLARE_ACCOUNT_ID`

### Simple Worker Deploy

```yaml
# .github/workflows/deploy.yml
name: Deploy Worker
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
```

**Action docs:** https://github.com/cloudflare/wrangler-action

### Fullstack Deploy (API + Client)

```yaml
# .github/workflows/deploy-fullstack.yml
name: Deploy Fullstack
on:
  push:
    branches: [main]

jobs:
  deploy-api:
    runs-on: ubuntu-latest
    outputs:
      api-url: ${{ steps.deploy.outputs.deployment-url }}
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1
      - run: bun install

      - name: Run D1 migrations
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          workingDirectory: api
          command: d1 migrations apply my-db --remote

      - name: Deploy API
        id: deploy
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          workingDirectory: api

  deploy-client:
    runs-on: ubuntu-latest
    needs: deploy-api
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v1
      - run: bun install

      - name: Build with API URL
        working-directory: client
        run: bun run build
        env:
          VITE_API_URL: ${{ needs.deploy-api.outputs.api-url }}

      - name: Deploy Client
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          workingDirectory: client
          command: pages deploy dist --project-name=my-client
```

### Preview on PRs

```yaml
# .github/workflows/preview.yml
name: Preview Deploy
on:
  pull_request:

jobs:
  preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          command: deploy --env preview
```

### Python Worker Deploy

```yaml
# .github/workflows/deploy-python.yml
name: Deploy Python Worker
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v7
      - run: uv sync
      - run: uv run pywrangler deploy
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
```

---

## Pages Auto-Deploy (Zero-Config CI/CD)

For static sites, you can skip GitHub Actions entirely:

1. Cloudflare Dashboard → Pages → Create Project
2. Connect GitHub repository
3. Set build command (`npm run build`) and output directory (`dist`)
4. Done. Every push to main = production deploy. Every PR = preview deploy with unique URL.

Skip deploy with commit message prefix: `[CI Skip]` or `[CF-Pages-Skip]`

**Docs:** https://developers.cloudflare.com/pages/configuration/git-integration/

---

## Best Practices

1. **Scope API tokens** - create one token per project with minimal permissions
2. **Never commit secrets** - `.dev.vars` for local, `wrangler secret put` for prod
3. **.gitignore** must include: `.dev.vars*`, `.env*`, `.wrangler/`
4. **Separate databases per environment** - production and preview should use different D1 databases
5. **Preview deploys on PRs** - catch bugs before they hit production
6. **Deploy API before client** - client needs a working API endpoint to proxy to

## Common Gotchas

1. **Pages `_redirects` does NOT support proxy (200 status)** — unlike Netlify. You MUST
   use Pages Functions (`functions/` dir) to proxy API calls to a separate Worker. See above.
2. **Better Auth "Invalid origin"** — when Pages (frontend) and Workers (API) are on
   different domains, add the Pages URL to `trustedOrigins` in your Better Auth config.
3. **Duplicate route files** — TanStack Router's file-based routing will error if both
   `.tsx` and `.js` versions of the same route exist. Remove duplicates before building.
4. **`wrangler pages deploy` picks up functions/** — the `functions/` directory must be
   relative to your working directory (not inside `dist/`). Wrangler auto-detects and bundles them.

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Secrets | https://developers.cloudflare.com/workers/configuration/secrets/ |
| Environment variables | https://developers.cloudflare.com/workers/configuration/environment-variables/ |
| Environments | https://developers.cloudflare.com/workers/wrangler/environments/ |
| GitHub Actions | https://developers.cloudflare.com/workers/ci-cd/external-cicd/github-actions/ |
| wrangler-action | https://github.com/cloudflare/wrangler-action |
| Workers Builds (native CI) | https://developers.cloudflare.com/workers/ci-cd/builds/ |
| Pages Git integration | https://developers.cloudflare.com/pages/configuration/git-integration/ |
| API tokens | https://developers.cloudflare.com/fundamentals/api/get-started/create-token/ |
