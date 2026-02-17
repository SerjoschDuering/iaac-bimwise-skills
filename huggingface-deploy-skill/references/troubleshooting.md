# Troubleshooting

## Build Failures

### "No app file found"
Missing or malformed `README.md` frontmatter. Must have `sdk: docker` and be at repo root.

### Dockerfile build error
Check logs on the HF dashboard (Space page → Logs tab). Common causes:
- Missing system deps for ifcopenshell (`libgomp1 libgl1`) — note: `libgl1-mesa-glx` was renamed to `libgl1` in Debian Bookworm (python:3.11-slim)
- Wrong Python version (use 3.11 — ifcopenshell wheels may not exist for 3.12+)
- `pip install` fails — check package names and version pins
- Network timeout during `git clone` inside Dockerfile

### Build hangs or times out
HF build infra has a ~20 minute timeout. Common culprits:
- `git clone` hitting rate limits or private repos without auth
- Large dependencies (PyTorch, etc.) — consider if you actually need them
- Missing `--no-cache-dir` on pip install (caching to ephemeral disk is pointless)

### "Permission denied" during build
Files not owned by uid 1000. Use `COPY --chown=user` and `USER user` in Dockerfile.

## Runtime Issues

### Space shows "ERROR" status
The container crashed at startup. Check logs on dashboard. Most common causes:
- Import error (missing dependency, syntax error in your code)
- Port mismatch: app must listen on `0.0.0.0:7860`
- Missing secret: `os.environ["GEMINI_API_KEY"]` throws KeyError if secret not set
- UID mismatch: must run as user 1000

### Cold start takes 60+ seconds
Normal for free tier after 48h inactivity. Mitigation:

```python
# Wake all spaces before a demo
from huggingface_hub import HfApi
api = HfApi()
for team in ["team-a", "team-b", "team-c", "team-d", "team-e"]:
    api.restart_space(f"ifcore-org/ifcore-{team}")
```

### "Connection refused" when calling the Space
- Space might be sleeping — check status with `api.get_space_runtime()`
- Space might still be building — wait for RUNNING stage
- URL format wrong — check hyphen doubling rule in sdk-operations.md

## Network Issues

### Can't connect to external database
Outbound connections on non-standard ports may be blocked. HTTP/HTTPS (80/443)
always works. For databases, use HTTP APIs or a proxy Worker instead of direct connections.

### `*.workers.dev` URLs don't resolve from inside HF Space
Known DNS issue. Cloudflare `*.workers.dev` three-dot subdomains fail from HF infra.
**Use a custom domain** on the Cloudflare Worker instead (e.g. `api.ifcore.dev`).

### CORS errors calling the Space from a frontend
Add CORS middleware to your FastAPI app:
```python
from fastapi.middleware.cors import CORSMiddleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://your-frontend.pages.dev"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## Git / Auth Issues

### `git push` returns 401 or asks for password
Run `hf auth login --add-to-git-credential` to store your HF token as a git credential.

### `git clone` fails with "repository not found"
- Check the URL format: `https://huggingface.co/spaces/USERNAME/SPACE-NAME`
- Verify the Space exists: `hf spaces ls --search "my-app"`

## Secrets Issues

### Secret not available at runtime
- Set via SDK (`api.add_space_secret()`), not git
- Read with `os.environ.get("KEY")` (not `os.environ["KEY"]` — use `.get()` for safety)
- Secrets are only available at **runtime**, not during `docker build`
- Restart after changing: `api.restart_space("your-org/my-app")`

### Secret needed during Docker build (e.g. private git clone)
Use Docker build secrets (advanced):
```dockerfile
RUN --mount=type=secret,id=GITHUB_TOKEN,mode=0444,required=true \
    git clone https://$(cat /run/secrets/GITHUB_TOKEN)@github.com/org/repo.git
```

## Disk and Memory

### "No space left on device" during build
Docker image too large. Use `--no-cache-dir` on pip, multi-stage builds,
or download large files at runtime instead of bundling.

### Out of memory at runtime
Free tier: 16 GB RAM. If loading IFC models concurrently, memory adds up.
Process one file at a time if possible. Clean up with `del model` after each check.
