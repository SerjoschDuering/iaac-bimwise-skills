# Deployment & CI/CD

How code goes from your laptop to a running platform.

## The Big Picture

```
Your team repo (GitHub)           ifcore-platform repo (GitHub)
  tools/checker_*.py                  backend/ + frontend/
       |                                   |
       | (git submodule)                   | (push to staging or main branch)
       v                                   v
  backend/teams/your-team/         GitHub Actions (automatic)
                                      |              |
                                      v              v
                                 HuggingFace     Cloudflare
                                  Space           Worker
                                (Python API)    (React + API)
```

**Two deployments, one repo:**
- **Backend** (Python, FastAPI) deploys to a HuggingFace Docker Space — this is where your `check_*` functions actually run
- **Frontend** (React app + API layer) deploys to a Cloudflare Worker — this is what users see in the browser

## Staging vs Production

There are **two copies** of the platform running at all times:

| | Staging | Production |
|---|---|---|
| **Purpose** | Test changes safely | What users see |
| **Branch** | `staging` | `main` (protected — requires PR + review) |
| **HF Space** | `serjd-ifcore-platform-staging.hf.space` | `serjd-ifcore-platform.hf.space` |
| **CF Worker** | `ifcore-platform-staging.tralala798.workers.dev` | `ifcore-platform.tralala798.workers.dev` |
| **Database** | `ifcore-db-staging` (separate SQLite DB on Cloudflare) | `ifcore-db` (separate) |
| **File storage** | `ifcore-files-staging` (separate bucket on Cloudflare) | `ifcore-files` (separate) |

Staging and production have **completely separate databases and file storage**.
Nothing you do on staging affects production.

**Note:** HuggingFace lowercases your username in Space URLs. `serJD` becomes `serjd` in the URL.

## How Deployment Triggers

```
feature branch
    |
    | merge PR into staging
    v
staging branch --push--> GitHub Actions --> deploys to staging HF + CF
    |
    | test on staging, verify it works
    | create PR from staging to main (requires 1 approving review)
    v
main branch ---push--> GitHub Actions --> deploys to production HF + CF
```

Under normal circumstances, **you don't deploy manually** — push to a branch, and
GitHub Actions does the rest. See "Manual Deploys" at the bottom for emergencies.

### What GitHub Actions Does

When you push to `staging` or `main`, the workflow (`.github/workflows/deploy.yml`)
runs two parallel jobs:

1. **Backend job:**
   - Checks out the repo
   - Fetches all team submodules at their latest commit (`git submodule update --init --recursive --remote`)
   - Runs `deploy.sh staging` or `deploy.sh prod` depending on the branch
   - `deploy.sh` copies everything into a clean temp directory and force-pushes to the HF Space
     (force-push is safe here — the HF Space is not a collaboration repo, it is just a deploy target)
   - HF rebuilds the Docker container (~2 min)

2. **Frontend job:**
   - Checks out the repo
   - Installs Node.js 20 and npm dependencies
   - Runs database migrations (creates/updates tables in Cloudflare D1, an edge SQLite database)
   - Builds the React app with Vite
   - Deploys the Worker to Cloudflare

**How to check if deployment succeeded:** Go to the **Actions tab** in the GitHub repo.
A green checkmark means it worked. Click a failed run to see the error logs.

## How Your Team Code Reaches Production

Your team writes `check_*` functions and pushes to your own repo. Here is how those
functions end up running on the live platform:

1. **You push** `tools/checker_doors.py` to your team repo on GitHub
2. **A captain** updates the submodule reference in `ifcore-platform`
3. **Captain pushes** to the `staging` branch
4. **GitHub Actions** runs `deploy.sh staging`:
   - `git submodule update --init --recursive --remote` pulls your latest code
   - `rsync` copies everything to a clean temp directory (stripping binary files)
   - Force-pushes to the staging HF Space
5. **The HF Space rebuilds** with Docker (~2 min)
6. **The orchestrator** (`orchestrator.py`) scans `teams/*/tools/checker_*.py`
   and discovers your `check_*` functions automatically
7. **When someone uploads an IFC file**, your function runs against it
8. After verifying on staging, captain creates a PR from `staging` to `main`
9. PR gets 1 review, merges, GitHub Actions deploys to production

**Teams never touch the platform repo directly.** You only push to your own team repo.

## Data Flow (How Checks Run)

Once deployed, this is what happens when a user runs checks:

```
1. User uploads IFC file
   Browser --> CF Worker POST /api/upload --> stored in R2 (Cloudflare's file storage)

2. User clicks "Run Checks" (must be logged in)
   Browser --> CF Worker POST /api/checks/run {project_id}
                --> auth check + project ownership check
                --> looks up file_url from D1 (never trusts the client)
                --> reads IFC from R2, encodes as base64
                --> POST {ifc_b64, project_id} to HF Space /check
                --> HF returns {job_id} immediately

3. HF runs all check_* functions in background (~5-30s depending on model size)

4. Browser polls every 2 seconds (auth + ownership checked each poll)
   Browser --> CF Worker GET /api/checks/jobs/:id
                --> CF Worker polls HF GET /jobs/{hf_job_id}
                --> when done: stores results in D1 (Cloudflare's SQLite database)
                --> returns results to browser
```

**Why polling instead of instant response?** HuggingFace Spaces cannot reach
Cloudflare Worker URLs (a DNS routing limitation). So the browser asks "are the
results ready yet?" every 2 seconds until they are.

**Why base64?** Same DNS reason. The HF backend cannot download files from
the CF Worker, so the Worker reads the IFC file and sends the raw data
inline in the request body.

## Environment Variables & Secrets

Variables the platform needs to run. There are two kinds:

**Plain variables** (not sensitive, stored in config files):
- `HF_SPACE_URL` — which HF Space to talk to. Set in `wrangler.jsonc` per environment.

**Secrets** (sensitive, stored encrypted, NEVER in code):

| Secret | Where to set it | What it is |
|---|---|---|
| `GEMINI_API_KEY` | HF Space Settings > Secrets | Google AI key for the chat feature |
| `BETTER_AUTH_SECRET` | `wrangler secret put BETTER_AUTH_SECRET` | Random string for auth sessions |
| `BETTER_AUTH_URL` | `wrangler secret put BETTER_AUTH_URL` | The Worker's own URL |
| `HF_TOKEN` | GitHub repo Settings > Secrets > Actions | HuggingFace access token for deploys |
| `CLOUDFLARE_API_TOKEN` | GitHub repo Settings > Secrets > Actions | Cloudflare API token for deploys |
| `CLOUDFLARE_ACCOUNT_ID` | GitHub repo Settings > Secrets > Actions | Your Cloudflare account ID |

## Local Development

You don't need staging or production to develop. Run everything locally:

```bash
# Terminal 1 — backend
cd backend && pip install -r requirements.txt
uvicorn main:app --reload --port 7860

# Terminal 2 — frontend
cd frontend && npm install && npm run db:migrate
echo "HF_SPACE_URL=http://localhost:7860" > .dev.vars
npm run dev    # --> http://localhost:5173
```

The Cloudflare Vite plugin emulates the database (D1) and file storage (R2)
locally. No cloud accounts needed for development.

Note: `npm run db:migrate` creates a local SQLite file for development.
This is different from `npm run db:migrate:staging` which updates the
actual cloud database.

## Manual Deploys (Emergency Only)

If CI is broken or you need a hotfix, you can deploy manually.
**Remember:** manual deploys skip database migrations. Run them separately if needed.

```bash
# Backend — staging
bash backend/deploy.sh staging

# Backend — production
bash backend/deploy.sh prod

# Frontend — staging (includes build + deploy to staging Worker)
cd frontend && npm run deploy:staging

# Frontend — production (includes build + deploy to production Worker)
cd frontend && npm run deploy

# Database migrations (if skipped)
cd frontend && npm run db:migrate:staging        # staging
cd frontend && npm run db:migrate:remote         # production
```

## Verify What Is Deployed

```bash
# Staging backend
curl https://serjd-ifcore-platform-staging.hf.space/health

# Production backend
curl https://serjd-ifcore-platform.hf.space/health
```

Both should return JSON listing discovered checks. If the staging Space is sleeping
(it goes to sleep after 48 hours of inactivity), the first request can take up to 60 seconds to wake it up.

## Related Skills

These skills provide deeper guidance for specific parts of the stack:

- **Cloudflare Skill** — Workers, D1 database, R2 storage, deployment patterns
- **HuggingFace Deploy Skill** — Docker Spaces, secrets, deploy scripts
- **PydanticAI Skill** — AI agents, tools, structured output, chat patterns
