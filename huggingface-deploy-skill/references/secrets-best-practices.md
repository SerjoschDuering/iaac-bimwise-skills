# Secrets Best Practices

LLM API keys (Gemini, OpenAI, etc.) must never be committed to git.
HuggingFace Spaces injects secrets as environment variables at runtime.

## The Rule

```
git repo   → public, no secrets
HF Secrets → private, injected at runtime only
```

## Set Secrets Before First Push

If the app starts before secrets are set, it crashes with `KeyError`.
Set secrets first, then push code.

```python
from huggingface_hub import HfApi
api = HfApi()

api.add_space_secret("serJD/my-app", key="GEMINI_API_KEY", value="AIza...")
api.add_space_secret("serJD/my-app", key="OPENAI_API_KEY", value="sk-...")
```

## Read Secrets Safely in Code

```python
import os

# Use .get() with a fallback — avoids KeyError if secret missing
GEMINI_KEY = os.environ.get("GEMINI_API_KEY", "")

if not GEMINI_KEY:
    raise RuntimeError("GEMINI_API_KEY secret not set — add it in Space settings")
```

Fail loudly at startup rather than silently at request time.

## Multiple Team Spaces — Batch Set

```python
from huggingface_hub import HfApi
import os

api = HfApi()
ORG = "ifcore-org"
GEMINI_KEY = os.environ["GEMINI_API_KEY"]  # load from YOUR env, never hardcode

teams = ["team-a", "team-b", "team-c", "team-d", "team-e"]
for team in teams:
    api.add_space_secret(f"{ORG}/ifcore-{team}", key="GEMINI_API_KEY", value=GEMINI_KEY)
    print(f"Set secret for {team}")
```

Run this from your local machine where the key is in your env — never from inside a Space.

## Update a Secret

```python
# Same call — add_space_secret overwrites if key already exists
api.add_space_secret("serJD/my-app", key="GEMINI_API_KEY", value="AIza_new...")
# Then restart to pick up the new value
api.restart_space("serJD/my-app")
```

## Delete a Secret

```python
api.delete_space_secret("serJD/my-app", key="GEMINI_API_KEY")
```

## What Secrets Are NOT

- Not available during `docker build` — only at runtime
- Not visible in git history, logs, or HF Space files tab
- Not shared between Spaces — set per Space

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Hardcoded key in `main.py` | Use `os.environ.get("KEY")` |
| Key in `requirements.txt` or Dockerfile | Never put secrets in files |
| Using `os.environ["KEY"]` without fallback | Use `.get()` + startup check |
| Forgetting to restart after update | `api.restart_space("org/name")` |
| Pushing before setting secrets | Set secrets first, then `git push` |

## Secrets You'll Need

| Secret | Purpose | Who sets it |
|--------|---------|-------------|
| `GEMINI_API_KEY` | LLM-powered checks (PydanticAI) | Instructor (shared key) |
| `S3_ACCESS_KEY` | R2 file download (see [R2 File Storage](./r2-file-storage.md)) | Captain |
| `S3_SECRET_KEY` | R2 file download | Captain |
| `HF_TOKEN` | Only if cloning private repos in Dockerfile | Captain |
