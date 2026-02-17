# R2 File Storage (IFC Models)

IFC files live in Cloudflare R2 (S3-compatible, zero egress fees).
Spaces download them at check time — never bundle IFC files in the Docker image.

```
User uploads IFC → R2 bucket → Space downloads at /check time → processes → deletes temp file
```

## Setup

Secrets (set before first push — see [Secrets](./secrets-best-practices.md)):
```python
api.add_space_secret("org/app", key="S3_ACCESS_KEY", value="...")
api.add_space_secret("org/app", key="S3_SECRET_KEY", value="...")
api.add_space_variable("org/app", key="S3_BUCKET_URL",
                        value="https://your-bucket.r2.cloudflarestorage.com")
```

## Download Pattern

```python
import boto3, os, tempfile

s3 = boto3.client("s3",
    endpoint_url=os.environ.get("S3_BUCKET_URL"),
    aws_access_key_id=os.environ.get("S3_ACCESS_KEY"),
    aws_secret_access_key=os.environ.get("S3_SECRET_KEY"),
)

def download_ifc(bucket: str, key: str) -> str:
    """Download IFC from R2, return local temp path."""
    tmp = tempfile.NamedTemporaryFile(suffix=".ifc", delete=False)
    s3.download_file(bucket, key, tmp.name)
    return tmp.name
```

Add `boto3` to `requirements.txt`.

## Caching Parsed Models

Same IFC file gets checked repeatedly (students iterate on their check functions).
Don't re-download and re-parse every time. Cache the parsed model in memory, evict oldest.

```python
import ifcopenshell, hashlib, gc
from collections import OrderedDict

MAX_CACHED_MODELS = 8       # evict oldest when this many models are cached
MAX_CACHE_MB = 8_000        # hard cap ~8GB (leaves 8GB for app + OS)
_cache_mb_used = 0

_model_cache: OrderedDict[str, tuple[ifcopenshell.file, float]] = OrderedDict()

def _estimate_mb(path: str) -> float:
    """IFC models use ~10x file size in RAM. Rough but safe."""
    return os.path.getsize(path) / 1_000_000 * 10

def get_model(ifc_url: str) -> ifcopenshell.file:
    """Download + parse once, return cached model on repeat calls."""
    global _cache_mb_used
    cache_key = hashlib.md5(ifc_url.encode()).hexdigest()

    if cache_key in _model_cache:
        _model_cache.move_to_end(cache_key)
        return _model_cache[cache_key][0]

    # Download and parse
    path = download_ifc_from_url(ifc_url)
    mb = _estimate_mb(path)
    model = ifcopenshell.open(path)
    os.unlink(path)

    # Evict oldest until within budget
    while (_cache_mb_used + mb > MAX_CACHE_MB or
           len(_model_cache) >= MAX_CACHED_MODELS) and _model_cache:
        _, (old_model, old_mb) = _model_cache.popitem(last=False)
        _cache_mb_used -= old_mb
        del old_model
        gc.collect()

    _model_cache[cache_key] = (model, mb)
    _cache_mb_used += mb
    return model
```

The `/check` endpoint just calls `get_model()` — first request downloads,
subsequent requests for the same URL return the cached model instantly.

```python
@app.post("/check")
async def check(ifc_url: str):
    model = get_model(ifc_url)  # cached on repeat calls
    return run_all_checks(model)
```

**Sizing:** Duplex model ≈ 2MB file → ~20MB in RAM (10x rule of thumb).
Free tier has 16GB — cache budget is 8GB (`MAX_CACHE_MB`), up to 8 models.
Leaves 8GB for FastAPI, Python, OS. Evicts oldest when either limit is hit.

**Cache clears on restart** — HF Spaces are ephemeral, so no stale data risk.
