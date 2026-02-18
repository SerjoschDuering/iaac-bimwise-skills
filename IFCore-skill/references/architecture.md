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
submodules and flattens them into `teams/<your-repo>/tools/` before building the Docker image.
Your repo structure (`tools/checker_*.py` with `check_*` functions) must match this layout
exactly for auto-discovery to work. Captains handle the pull and flatten via `deploy.sh` —
teams never push to the platform repo directly.

## Code Conventions

- **Max 300 lines per file.** Split into modules when approaching the limit.
- **One function per check.** Don't combine multiple regulation checks.
- **File names:** `checker_<topic>.py` — e.g. `checker_doors.py`, `checker_fire_safety.py`.
- **Function names:** `check_<what>` — e.g. `check_door_width`, `check_room_area`.
- **First arg is always `model`** — an `ifcopenshell.file` object.
- **Return `list[str]`** — each string prefixed `[PASS]`, `[FAIL]`, or `[???]`.
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
<paste your app/ directory tree here>

## Conventions
- Max 300 lines per file
- One function per regulation check
- Files: tools/checker_<topic>.py — only checker_*.py files are scanned
- Functions: check_*(model, ...) -> list[str] with [PASS]/[FAIL]/[???] prefix

## Issue Reporting
When you encounter a contract mismatch, skill gap, or integration problem:
gh issue create --repo SerjoschDuering/iaac-bimwise-skills --label "<label>" --title "<title>"
Labels: contract-gap, skill-drift, learning, schema-change, integration-bug

## Learnings
<!-- Add here after every debugging session that reveals a recurring issue -->
```

**Keep it updated.** After any session where you hit a recurring error, add it to Learnings.
The AI gets smarter with every fix — only if you write it down.
