# Architecture & Conventions

## Project Structure

```
your-team-repo/
├── tools/
│   ├── checker_doors.py       # check_door_width, check_door_clearance
│   ├── checker_fire_safety.py # check_fire_rating, check_exit_count
│   └── checker_rooms.py       # check_room_area, check_ceiling_height
├── requirements.txt            # team dependencies
└── README.md
```

**File naming:** `checker_<topic>.py` — group related checks by topic.
Only `tools/checker_*.py` matters to the platform. Everything else (local test scripts,
notebooks, Gradio apps, CLI tools) is your choice — the platform ignores it.

**Platform auto-discovery:** the orchestrator scans `teams/*/tools/checker_*.py` and collects
every `check_*` function. No subdirectories — files must be directly inside `tools/`.
Helper files (e.g. `tools/utils.py`) are fine for shared code but won't be scanned.

**Platform integration:** the platform (`ifcore-platform`) pulls all 5 team repos via git
submodules under `backend/teams/`. `deploy.sh` runs `git submodule update --init --recursive --remote`,
then rsync copies the entire `backend/` directory (with real team files, no symlinks) to a temp dir
and force-pushes to HuggingFace. Your `tools/checker_*.py` files end up at
`teams/<your-repo>/tools/checker_*.py` on the HF Space.
Captains run `deploy.sh` — teams never push to the platform repo directly.

## Request Flow

All frontend requests go through the CF Worker (`/api/*`). The Worker proxies to HF where needed:

```
Browser → CF Worker /api/upload           → stores IFC in R2, creates project in D1
Browser → CF Worker /api/projects         → list own + shared projects from D1
Browser → CF Worker /api/projects/:id     → AUTH + ownership → single project + jobs
Browser → CF Worker /api/checks/run       → AUTH + ownership → reads IFC from R2 as base64 → POST to HF /check
Browser → CF Worker /api/checks/jobs/:id  → AUTH + ownership → lazy-polls HF → remaps job_id → updates D1
Browser → CF Worker /api/chat             → AUTH required → proxies to HF /chat (PydanticAI + Gemini)
Browser → CF Worker /api/stats            → AUTH required → user's aggregated stats
Browser → CF Worker /api/auth/*           → Better Auth (D1-backed sessions via Drizzle)
Browser → CF Worker /api/files/:key       → ownership check via R2 key → serves objects from R2
Browser → CF Worker /api/health           → health check (no auth)
```

**Never call HF directly from the browser.** HF Spaces cannot resolve `*.workers.dev` DNS, and
CORS issues make direct calls unreliable. The Worker is the single gateway.

**job_id remapping:** HF generates its own job UUIDs. The Worker must remap `check_result.job_id`
from the HF UUID to the CF job UUID before inserting into D1 (foreign key constraint).

### Access Control Model

Two kinds of projects:
- **Shared** (`user_id = null`) — accessible to everyone, used for demo models on the landing page
- **Private** (`user_id = <id>`) — only accessible to the owner

The helper `canAccessProject(project, userId)` in `worker/lib/db.ts` enforces this:
```ts
if (!project.user_id) return true;   // shared → anyone
return project.user_id === userId;    // private → owner only
```

Routes that modify or read private data call `getSessionUser()` + `canAccessProject()`.
Upload (`POST /api/upload`) is intentionally unauthenticated — it's a student exercise to add auth.

**Trust boundary:** The Worker never trusts client-supplied `file_url`. When running checks,
it looks up `file_url` from D1 by `project_id` and reads R2 directly.

## Concurrency

The HF Space runs with `--workers 1` (single uvicorn worker — ifcopenshell is not fork-safe). Each check job:
1. Receives base64-encoded IFC from the CF Worker (avoids HF DNS issues)
2. Runs all discovered `check_*` functions against the model
3. Stores results in-memory (`_jobs` dict); CF Worker polls and writes to D1

The frontend renders IFC directly in the browser (no GLB conversion needed).
See [3D Viewer](./3d-viewer.md) for details.

This uses `BackgroundTasks` (FastAPI), NOT `asyncio.get_event_loop().create_task()`.
The CPU-heavy IFC processing runs in a background thread automatically.

## Environment Variables

**Backend (HF Space):** `GEMINI_API_KEY` — required for the `/chat` endpoint (PydanticAI + Gemini).
Without it, chat returns `UserError`. Set as a HF Space secret for production.

**Frontend (CF Worker):** Secrets in `frontend/.dev.vars` (local) or `wrangler secret put` (prod):
- `BETTER_AUTH_SECRET` — at least 32 random chars for auth sessions
- `BETTER_AUTH_URL` — worker's own URL (e.g. `http://localhost:5173` locally)
- `HF_SPACE_URL` — where the HF backend lives

## Code Conventions

- **Max 300 lines per file.** Split into modules when approaching the limit.
- **One function per check.** Don't combine multiple regulation checks.
- **File names:** `checker_<topic>.py` — e.g. `checker_doors.py`, `checker_fire_safety.py`.
- **Function names:** `check_<what>` — e.g. `check_door_width`, `check_room_area`.
- **First arg is always `model`** — an `ifcopenshell.file` object.
- **Return `list[dict]`** — each dict has `element_id`, `element_type`, `element_name`, `element_name_long`, `check_status`, `actual_value`, `required_value`, `comment`, `log` (see [Validation Schema](./validation-schema.md)).
- **No bare try/except.** Only catch specific known errors.

**What is `model`?** It's an `ifcopenshell.file` object — a parsed IFC file loaded into memory.
You query it with `model.by_type("IfcDoor")` to get all doors, `model.by_type("IfcWall")` for
walls, etc. Each element has properties like `.Name`, `.GlobalId`, and type-specific attributes.

## AGENTS.md / CLAUDE.md

Every team MUST have this file in their repo root. Your AI assistant reads it automatically.
If it does not exist, create it before starting any work.

**Template:**
```markdown
# <Project Name>

Always read the IFCore skill before developing on this project.

## Structure
<paste your tools/ directory tree here>

## Conventions
- Max 300 lines per file
- One function per regulation check
- Files: tools/checker_<topic>.py — only checker_*.py files are scanned
- Functions: check_*(model, ...) -> list[dict] per validation-schema.md (check_status, actual_value, etc.)

## Issue Reporting
When you encounter a contract mismatch, skill gap, or integration problem:
gh issue create --repo SerjoschDuering/iaac-bimwise-skills --label "<label>" --title "<title>"
Labels: contract-gap, skill-drift, learning, schema-change, integration-bug

## Learnings
<!-- Add here after every debugging session that reveals a recurring issue -->
```

**Keep it updated.** After any session where you hit a recurring error, add it to Learnings.
The AI gets smarter with every fix — only if you write it down.
