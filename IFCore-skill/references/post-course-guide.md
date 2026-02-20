# After the Course: Fork & Own the Platform

This guide is for students who want to keep developing after the IAAC AI Week.
You will fork the platform, deploy it under your own accounts, and have a fully
independent IFC compliance checker.

## What You Need (All Free)

| Service | Sign up | What it does |
|---|---|---|
| [GitHub](https://github.com) | Already have it | Hosts your code |
| [HuggingFace](https://huggingface.co) | Free account | Runs the Python backend |
| [Cloudflare](https://dash.cloudflare.com/sign-up) | Free account | Hosts frontend + database + file storage |
| [Google AI Studio](https://aistudio.google.com/apikey) | Free API key | Powers the AI chat feature |
| [Node.js 18+](https://nodejs.org) | Install locally | Required to build the frontend |
| Python 3.10+ | Install locally | Required to run the backend |

## Step 1: Fork the Repos

Fork these on GitHub (click "Fork" button on each repo page):

1. **Platform:** `SerjoschDuering/ifcore-platform` → your account
2. **Your team repo:** already yours
3. **Skills (optional):** `SerjoschDuering/iaac-bimwise-skills` → your account

```bash
git clone https://github.com/YOUR_USERNAME/ifcore-platform.git
cd ifcore-platform
```

## Step 2: Create Your HuggingFace Space

1. Go to [huggingface.co/new-space](https://huggingface.co/new-space)
2. Name: `ifcore-platform` (or anything you want)
3. SDK: **Docker**
4. Visibility: **Public** (free tier requires public)
5. Note your username — your URL will be:
   `https://YOURUSERNAME-ifcore-platform.hf.space`
   **Important:** HF lowercases your username in URLs. If your username is `MyName`,
   the URL becomes `https://myname-ifcore-platform.hf.space`.

**Get an access token** (you will need this for deploys):
1. Go to [huggingface.co/settings/tokens](https://huggingface.co/settings/tokens)
2. Create a new token with **Write** access
3. Save it somewhere safe — you will use it in Steps 5 and 6

**Set the Gemini API key** (for the AI chat feature):
- Go to your Space page > Settings > Secrets > New Secret
- Key: `GEMINI_API_KEY`, Value: your key from [Google AI Studio](https://aistudio.google.com/apikey)

## Step 3: Create Cloudflare Resources

```bash
cd frontend
npm install
npx wrangler login              # opens browser, authenticate once

# Create database and storage
npx wrangler d1 create ifcore-db
# ^^^ COPY the database_id UUID from the output — you need it in the next step

npx wrangler r2 bucket create ifcore-files
```

## Step 4: Update Config Files

**Only 2 files need your values.** Everything else works as-is.

### File 1: `backend/deploy.sh`

Open `backend/deploy.sh`. Find the `case` block near the top (around lines 8-11)
and replace `serJD` with your HuggingFace username:

```bash
# Before (instructor's values):
case "$TARGET" in
    staging) HF_REPO="${HF_REPO:-serJD/ifcore-platform-staging}" ;;
    prod)    HF_REPO="${HF_REPO:-serJD/ifcore-platform}" ;;

# After (your values):
case "$TARGET" in
    staging) HF_REPO="${HF_REPO:-YOURHFUSERNAME/ifcore-platform-staging}" ;;
    prod)    HF_REPO="${HF_REPO:-YOURHFUSERNAME/ifcore-platform}" ;;
```

### File 2: `frontend/wrangler.jsonc`

Open `frontend/wrangler.jsonc`. Update **only** these two values in the top-level
section (leave the rest unchanged — especially `binding` and `database_name`):

```jsonc
// Find these lines and update the values:
"database_id": "PASTE_YOUR_D1_ID_HERE"    // from Step 3
// ...
"HF_SPACE_URL": "https://YOURUSERNAME-ifcore-platform.hf.space"  // lowercase username!
```

**Do NOT delete the `binding` or `database_name` fields** — the app needs them.

## Step 5: Deploy

```bash
# Backend — authenticate with HuggingFace first
pip install huggingface_hub
huggingface-cli login           # paste the access token from Step 2

# Deploy to your HF Space (~2 min build)
bash backend/deploy.sh prod

# Frontend — create database tables and deploy
cd frontend
npm run db:migrate:remote       # creates tables in your D1 database
npm run deploy                  # builds + deploys to Cloudflare
```

**Verify it works:**
```bash
curl https://YOURUSERNAME-ifcore-platform.hf.space/health
```
Should return JSON like: `{"status": "ok", "checks_discovered": 5, ...}`

If the Space shows "Building" on the HF page, wait 2-3 minutes and try again.

## Step 6: Set Up CI/CD (Optional but Recommended)

This makes deployments automatic — push code, it deploys by itself.

1. Go to your fork on GitHub > Settings > Secrets and Variables > Actions
2. Add these three secrets:

| Secret name | Where to get it |
|---|---|
| `HF_TOKEN` | Your HuggingFace access token from Step 2 |
| `CLOUDFLARE_API_TOKEN` | [Cloudflare dashboard](https://dash.cloudflare.com/profile/api-tokens) > Create Token > Edit Cloudflare Workers |
| `CLOUDFLARE_ACCOUNT_ID` | Visible on your [Cloudflare dashboard](https://dash.cloudflare.com) home page |

**Important:** Make sure you already updated `deploy.sh` (Step 4) to point to YOUR
HuggingFace username before enabling CI/CD. Otherwise, deploys will try to push to
the instructor's Space and fail.

Now pushing to `main` auto-deploys to production. Pushing to `staging` auto-deploys
to staging (if you set that up in Step 7).

## Step 7: Add a Staging Environment (Optional)

If you want a safe testing environment before production:

```bash
# 1. Create a staging HF Space (same process as Step 2, name it ifcore-platform-staging)

# 2. Create staging Cloudflare resources
cd frontend
npx wrangler d1 create ifcore-db-staging
# ^^^ Copy this new database_id

npx wrangler r2 bucket create ifcore-files-staging
```

Open `frontend/wrangler.jsonc` and find the `env.staging` section at the bottom.
Replace `STAGING_DB_ID_HERE` with the real UUID from the command above.

Then create a staging branch:
```bash
git checkout -b staging
git push origin staging
```

Now pushing to `staging` deploys to your staging environment.

## Step 8: Add Your Team's Checks

Your team repo is already a submodule. To add or update teams:

```bash
# Update your existing submodule to the latest commit
git submodule update --remote backend/teams/your-team

# Or add a new team repo
git submodule add -f https://github.com/SOMEONE/their-repo backend/teams/new-team

# Deploy so the orchestrator discovers the new checks
bash backend/deploy.sh prod
```

## Adding New Features

See [Development Patterns](./development-patterns.md) for the full workflow,
and [Feature Development Workflow](./feature-development.md) for a step-by-step
guide with sample prompts you can give your AI assistant.

**Quick version:**
1. Describe what you want to build (a "spec" or feature request)
2. Ask your AI assistant to plan the implementation
3. Build on a feature branch, test locally
4. Push to staging, verify it works
5. Create a PR to main

## Troubleshooting

**HF Space shows "Building":** Wait 2-3 minutes. Check build logs on the Space page.

**HF Space is sleeping:** Free-tier Spaces sleep after 48h inactivity. First request
takes 10-60s to wake up. This is normal — just wait.

**Checks not discovered:** Two things to check:
1. Your `tools/checker_*.py` files must be directly in `tools/` (not in subdirectories)
2. Your team repo must be added as a submodule (Step 8) and deployed

**Frontend deploys but shows old code:** Always use `npm run deploy` (not `npx wrangler deploy`).
The npm script includes the Vite build step.

**"database_id not found":** You forgot to update the D1 ID in `wrangler.jsonc` (Step 4).

**CI deploys to wrong HF Space:** You forgot to update `deploy.sh` (Step 4) before
enabling CI/CD (Step 6).

**Need help?** Ask your AI assistant with the IFCore skill installed — it knows
all the contracts, patterns, and gotchas.
