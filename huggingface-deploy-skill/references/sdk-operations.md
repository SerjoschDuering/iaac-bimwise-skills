# SDK Operations

The `huggingface_hub` Python SDK handles secrets, batch provisioning, and status checks.
The `hf` CLI covers repo creation and auth. The SDK covers everything else.

```python
from huggingface_hub import HfApi
api = HfApi()  # uses token from `hf auth login`
```

## Create a Space

```python
api.create_repo(
    repo_id="your-org/my-app",
    repo_type="space",
    space_sdk="docker",
    private=False,
)
```

Or via CLI:
```bash
hf repo create my-app --repo-type space --space-sdk docker
```

## Upload Files to a Space

```python
# Upload an entire folder
api.upload_folder(
    repo_id="your-org/my-app",
    repo_type="space",
    folder_path="./my-app/",
)

# Upload a single file
api.upload_file(
    repo_id="your-org/my-app",
    repo_type="space",
    path_or_fileobj="main.py",
    path_in_repo="main.py",
)
```

## Secrets Management

Secrets are injected as environment variables at runtime. Never commit them to git.
Set secrets BEFORE pushing code — otherwise the app crashes on first startup.

```python
# Set a secret (private — only visible at runtime)
api.add_space_secret(
    repo_id="your-org/my-app",
    key="GEMINI_API_KEY",
    value="AIza...",
)

# Set a public variable (visible to anyone)
api.add_space_variable(
    repo_id="your-org/my-app",
    key="TEAM_NAME",
    value="team-a",
)
```

## Check Space Status

```python
runtime = api.get_space_runtime("your-org/my-app")
print(runtime.stage)
# RUNNING — app is live
# BUILDING — Docker image building (wait 2-5 min)
# RUNNING_BUILDING — old version running, new build in progress
# SLEEPING — inactive >48h, will cold-start on next request
# PAUSED — manually paused
# ERROR — build or runtime failure (check logs on dashboard)
# NO_APP_FILE — missing README.md frontmatter
```

## Restart / Wake a Sleeping Space

```python
api.restart_space("your-org/my-app")
```

Useful before demos to avoid cold start delays:
```python
# Wake all team spaces before a presentation
for team in ["team-a", "team-b", "team-c", "team-d", "team-e"]:
    api.restart_space(f"ifcore-org/ifcore-{team}")
    print(f"Waking ifcore-{team}...")
```

## Duplicate a Template Space (Batch Provisioning)

Create multiple Spaces from one template in a loop:

```python
import os

GEMINI_KEY = os.environ["GEMINI_API_KEY"]
ORG = "ifcore-org"
TEMPLATE = f"{ORG}/ifcore-template"

teams = ["team-a", "team-b", "team-c", "team-d", "team-e"]

for team in teams:
    api.duplicate_space(
        from_id=TEMPLATE,
        to_id=f"{ORG}/ifcore-{team}",
        private=False,
        exist_ok=True,
        secrets=[{"key": "GEMINI_API_KEY", "value": GEMINI_KEY}],
        variables=[{"key": "TEAM_NAME", "value": team}],
    )
    print(f"Created: {ORG}/ifcore-{team}")
```

## Delete a Space

```python
api.delete_repo("your-org/my-app", repo_type="space")
```

## Space URLs

```
Public URL:  https://{owner}-{space-name}.hf.space
API docs:    https://{owner}-{space-name}.hf.space/docs
Health:      https://{owner}-{space-name}.hf.space/health
```

If the owner name contains hyphens, the separator between owner and space name
becomes `--`. Example: org `ifcore-org`, space `platform` →
`https://ifcore--org-platform.hf.space`.

When in doubt, check the Space Settings page for the exact URL.
