# Python Backend Patterns on Cloudflare Workers

**Status:** Open beta since April 2024. Actively developed. Not yet GA.

## How It Works

Python Workers run CPython compiled to WebAssembly (via Pyodide) inside V8 isolates.
At deploy time, Cloudflare snapshots your imports into memory - so cold starts are ~1 second
(2.4x faster than AWS Lambda).

## Setup

```bash
# Install pywrangler (wraps wrangler + uses uv for Python deps)
uv tool install workers-py

# New project
uv run pywrangler init my-python-api
cd my-python-api

# Local dev
uv run pywrangler dev

# Deploy
uv run pywrangler deploy

# Generate types for IDE
uv run pywrangler types
```

## wrangler.toml for Python

> **Note:** `pywrangler init` creates `wrangler.jsonc` (not `.toml`). Both formats work.

```toml
name = "my-python-api"
main = "src/entry.py"                          # .py file, not .ts!
compatibility_flags = ["python_workers"]       # REQUIRED (beta flag)
compatibility_date = "2025-09-01"              # Must be >= 2025-08-14 for class-based handlers

[[d1_databases]]
binding = "DB"
database_name = "my-db"
database_id = "xxxxx"

[[r2_buckets]]
binding = "BUCKET"
bucket_name = "my-files"

[[kv_namespaces]]
binding = "CACHE"
id = "xxxxx"
```

## pyproject.toml (Dependencies)

```toml
[project]
name = "my-python-api"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi",
    "httpx",
]

[dependency-groups]
dev = ["workers-py"]
```

**No pip, no requirements.txt.** Use `uv` + `pyproject.toml`.

---

## Handler Pattern (compatibility_date matters!)

> **CRITICAL:** The `compatibility_date` determines which handler pattern the runtime expects.
> - `>= 2025-08-14`: Class-based `WorkerEntrypoint.fetch()` (current, shown below)
> - `< 2025-08-14`: Old `on_fetch` top-level function (deprecated)
>
> If you get `TypeError: Method on_fetch does not exist`, your compat date is too old.
> Set it to `2025-09-01` or later.

## Basic Python Worker

```python
# src/entry.py
from workers import WorkerEntrypoint, Response

class Default(WorkerEntrypoint):
    async def fetch(self, request):
        return Response("Hello from Python Workers!")
```

## FastAPI on Workers

```python
# src/entry.py
from workers import WorkerEntrypoint
from fastapi import FastAPI, Request
from pydantic import BaseModel
import asgi

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello from FastAPI on Cloudflare!"}

@app.get("/items")
async def list_items(req: Request):
    env = req.scope["env"]  # Access CF bindings via request scope
    results = await env.DB.prepare("SELECT * FROM items").run()
    return {"data": results.results}

class ItemCreate(BaseModel):
    title: str
    price: float

@app.post("/items")
async def create_item(item: ItemCreate, req: Request):
    env = req.scope["env"]
    result = await env.DB.prepare(
        "INSERT INTO items (title, price) VALUES (?, ?)"
    ).bind(item.title, item.price).run()
    return {"data": result}

@app.get("/items/{item_id}")
async def get_item(item_id: int, req: Request):
    env = req.scope["env"]
    result = await env.DB.prepare(
        "SELECT * FROM items WHERE id = ?"
    ).bind(item_id).first()
    if not result:
        return {"error": "Not found"}, 404
    return {"data": result}

# Bridge FastAPI to Worker fetch handler
class Default(WorkerEntrypoint):
    async def fetch(self, request):
        return await asgi.fetch(app, request, self.env)
```

## D1 from Python

```python
# Read
results = await self.env.DB.prepare("SELECT * FROM users WHERE email = ?").bind("user@example.com").run()

# Write
await self.env.DB.prepare("INSERT INTO users (name, email) VALUES (?, ?)").bind("Alice", "alice@example.com").run()

# First row only
user = await self.env.DB.prepare("SELECT * FROM users WHERE id = ?").bind(1).first()
```

## KV from Python

```python
await self.env.CACHE.put("key", "value")
value = await self.env.CACHE.get("key")
await self.env.CACHE.delete("key")
```

## R2 from Python

```python
# Upload
await self.env.BUCKET.put("files/doc.pdf", file_bytes)

# Download
obj = await self.env.BUCKET.get("files/doc.pdf")

# List
listing = await self.env.BUCKET.list(prefix="files/")
```

---

## What Works / What Doesn't

### Supported
- Pure Python packages from PyPI
- Pyodide-bundled packages (NumPy, Pandas, Pillow, scipy, scikit-learn, matplotlib)
- Async HTTP clients (httpx, aiohttp)
- All CF bindings (D1, KV, R2, Durable Objects, AI, Queues, Vectorize)

### NOT Supported
- Flask, Django (synchronous frameworks)
- `requests` library (synchronous HTTP)
- Packages with native C extensions NOT in Pyodide
- threading, multiprocessing, subprocess
- File system operations (WASM sandbox)

### Limits
- 128MB memory per isolate (includes Python runtime + your code + data)
- Free: 10ms CPU/request | Paid: 5 min CPU/request
- Script size: 3MB compressed (free), 10MB (paid)
- Python-in-WASM is slower than native JS for CPU-intensive work

---

## Python vs JS/TS Decision

| Factor | JS/TS (Hono) | Python (FastAPI) |
|--------|-------------|-----------------|
| Maturity on CF | GA, production-ready | Open beta |
| Cold start | ~5ms (effectively zero with pre-warming) | ~1 second (WASM) |
| Package ecosystem | Full npm | PyPI + Pyodide subset |
| ORM | Drizzle (excellent D1 support) | Raw SQL via bindings |
| Type safety | End-to-end with Zod + TS | Pydantic models |
| Framework | Hono (CF-native) | FastAPI (adapted via ASGI) |
| Best for | Production apps, fullstack | Data/ML APIs, Python teams |
