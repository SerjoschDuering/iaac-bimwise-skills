# Docker Space Setup

A Docker-based HF Space needs these files at the repo root.
You can have additional files (src/*.py, data/, etc.) — these are the required ones.

## 1. README.md (frontmatter — required)

```yaml
---
title: My App Name
emoji: 🏗️
colorFrom: blue
colorTo: green
sdk: docker
app_port: 7860
pinned: false
---
```

`sdk: docker` tells HF to build from your Dockerfile (not Gradio/Streamlit).
`app_port: 7860` is the port your app listens on inside the container.

## 2. Dockerfile

```dockerfile
FROM python:3.11-slim

# ifcopenshell needs these system libraries
RUN apt-get update && apt-get install -y \
    git libgomp1 libgl1 \
    && rm -rf /var/lib/apt/lists/*

# HF Spaces runs containers as uid 1000
RUN useradd -m -u 1000 user
WORKDIR /app

COPY --chown=user requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY --chown=user . .
USER user

# Must bind to 0.0.0.0:7860 — HF routes traffic here
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "7860"]
```

**Critical rules:**
- Container runs as `uid 1000` — create the user and `chown` files to it
- Bind `0.0.0.0:7860` — HF routes external traffic to this port
- `--no-cache-dir` keeps image small
- Use `python:3.11-slim` — ifcopenshell wheels may not exist for 3.12+
- Include `git` in apt-get if your Dockerfile clones external repos at build time

**CMD must match your file:** if your FastAPI app is in `server.py` with variable
`application`, use `CMD ["uvicorn", "server:application", ...]`.

## 3. requirements.txt

```
fastapi>=0.110.0
uvicorn[standard]>=0.29.0
ifcopenshell
python-multipart
httpx
pydantic-ai
```

Add team-specific deps as needed. Pin versions to avoid conflicts.

## 4. main.py (FastAPI entry point)

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
def health():
    return {"status": "ok"}

@app.post("/check")
async def check(ifc_url: str):
    # Your check logic here
    return []
```

## Local Testing

```bash
# Option 1: Docker (matches production exactly)
docker build -t my-app .
docker run -p 7860:7860 -e GEMINI_API_KEY=your_key my-app
# open http://localhost:7860/docs

# Option 2: uvicorn directly (faster iteration, no Docker needed)
pip install -r requirements.txt
GEMINI_API_KEY=your_key uvicorn main:app --host 0.0.0.0 --port 7860 --reload
```

## Environment Variables vs Secrets

- **Variables** (public): set in README frontmatter or via SDK. Visible to anyone.
- **Secrets** (private): set via SDK only. Injected as env vars at runtime. Never in git.

```python
import os
GEMINI_KEY = os.environ.get("GEMINI_API_KEY")
```

Set secrets BEFORE your first push — see [SDK Operations](./sdk-operations.md).

## File Size Limits

- Max single file: 10 GB (requires Git LFS for files > 10 MB)
- Use `.gitattributes` for large files: `*.ifc filter=lfs diff=lfs merge=lfs`
- Prefer downloading large files at runtime (from URLs/R2) over bundling in the image
