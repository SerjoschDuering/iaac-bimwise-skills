---
name: huggingface-deploy
description: Use when deploying Python apps to HuggingFace Spaces. Covers preflight checks, Docker Space setup, CLI deployment, secrets, and monitoring.
---

# HuggingFace Spaces Deployment

Deploy Python applications as Docker-based HuggingFace Spaces via the `hf` CLI.
Docker SDK is required — ifcopenshell needs system libraries (`apt-get`).

## Preflight Checklist

Run these checks IN ORDER. Stop at the first failure.

### 1. CLI installed

```bash
pip install -U huggingface_hub
hf version
```

Requires `huggingface_hub >= 1.0.0`. The old `huggingface-cli` is deprecated — use `hf`.

### 2. Logged in

```bash
hf auth whoami
```

If this fails:
- Create account: https://huggingface.co/join
- Create token: https://huggingface.co/settings/tokens → New token → **Write** scope (must include Repos: Write and Spaces permissions)
- Login: `hf auth login` (paste token when prompted)
- Token is stored at `~/.cache/huggingface/token`

### 3. Git credential helper for HuggingFace

During `hf auth login` you are prompted "Add token as git credential? (Y/n)" — press Enter to accept. If you skipped it, run:

```bash
hf auth login --add-to-git-credential
```

Without this, `git clone` and `git push` to `huggingface.co` will fail with 401.

### 4. Git LFS

```bash
git lfs --version
```

If missing: `brew install git-lfs && git lfs install` (macOS) or `apt install git-lfs` (Linux).
Any version is fine — just needs to be installed.

### 5. Docker (optional — for local testing only)

```bash
docker --version
```

Not required for deployment. HF builds Docker images on their infra.

## Deployment Flow

```
1. Resolve your username    → hf auth whoami
2. Create Space repo        → hf repo create
3. Clone it locally         → git clone
4. Set secrets FIRST        → Python SDK (before pushing code!)
5. Add files + push         → git push (triggers build on HF)
6. Check status             → Python SDK: api.get_space_runtime()
```

### Step-by-step

```bash
# 1. Get your username (needed for URLs below)
# Note: output format is "user:  serJD" — extract last word
HF_USER=$(hf auth whoami | head -1 | awk '{print $NF}')
echo "Deploying as: $HF_USER"

# 2. Create the Space (Docker SDK)
hf repo create my-app --repo-type space --space-sdk docker

# 3. Clone
git clone https://huggingface.co/spaces/$HF_USER/my-app
cd my-app

# 4. Set secrets BEFORE first push (otherwise app crashes on startup)
python3 -c "
from huggingface_hub import HfApi
api = HfApi()
api.add_space_secret('$HF_USER/my-app', key='GEMINI_API_KEY', value='YOUR_KEY')
"

# 5. Add your files (see references/docker-space.md for required files)
# Required: README.md (with frontmatter), Dockerfile, main.py, requirements.txt
# Additional files (src/*.py, data/, etc.) are fine — include whatever you need.

# 6. Push — triggers Docker build on HF (takes 2-5 minutes)
git add -A && git commit -m "initial deploy" && git push

# 7. Check build status (Python SDK — no CLI equivalent)
python3 -c "
from huggingface_hub import HfApi
rt = HfApi().get_space_runtime('$HF_USER/my-app')
print(f'Status: {rt.stage}')
"
# BUILDING → wait. RUNNING → done. ERROR → check logs on HF dashboard.
```

Your Space URL:
```
https://$HF_USER-my-app.hf.space
https://$HF_USER-my-app.hf.space/docs   ← FastAPI auto-docs
```

Note: if your username/org contains hyphens, the URL separator becomes `--`.
Example: org `ifcore-org`, space `platform` → `https://ifcore--org-platform.hf.space`.
Check the Space Settings page for the exact URL if unsure.

## Build Logs

There is no CLI command for build logs. Two options:
- **Dashboard**: visit your Space page → "Logs" tab (fastest for debugging)
- **SDK status polling**: `api.get_space_runtime()` returns stage (BUILDING/RUNNING/ERROR) but not log content

## Free Tier Limits

| Resource | Limit |
|----------|-------|
| vCPU | 2 |
| RAM | 16 GB |
| Disk | 50 GB (ephemeral — resets on restart) |
| Sleep after inactivity | 48 hours |
| Cold start after sleep | 10–60 seconds |

**Network constraint:** outbound connections on non-standard ports may be blocked.
HTTP/HTTPS (80/443) always works. Direct database connections (PostgreSQL 5432,
MySQL 3306, Redis 6379) are unreliable — use HTTP APIs for external services.

## When Something Goes Wrong

File an issue on the skills repo:
```bash
gh issue create \
  --repo SerjoschDuering/iaac-bimwise-skills \
  --label "integration-bug" \
  --title "HF deploy: <what broke>"
```

## References

- [Docker Space Setup](./references/docker-space.md) — Dockerfile, README frontmatter, requirements, local testing
- [SDK Operations](./references/sdk-operations.md) — programmatic deployment, secrets, batch provisioning, status checks
- [Secrets Best Practices](./references/secrets-best-practices.md) — LLM API keys, safe env var patterns, batch provisioning secrets
- [R2 File Storage](./references/r2-file-storage.md) — IFC file transfer via S3/R2, download pattern, RAM cleanup
- [Troubleshooting](./references/troubleshooting.md) — build failures, cold starts, port issues, DNS gotchas
