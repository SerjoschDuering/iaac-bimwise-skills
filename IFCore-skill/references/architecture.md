# Architecture & Conventions

## Project Structure

```
your-team-repo/
├── src/
│   ├── ifc_checker.py     # check functions live here
│   └── fire_safety.py     # additional files are fine — same conventions
├── requirements.txt        # team dependencies
└── README.md
```

Only `src/*.py` matters to the platform. Everything else (local test scripts, notebooks,
Gradio apps, CLI tools) is your choice — the platform ignores it.

**Platform auto-discovery:** the platform scans all `src/*.py` files and collects every
`check_*` function. You don't register or export anything — just follow the naming convention.

## Code Conventions

- **Max 300 lines per file.** Split into modules when approaching the limit.
- **One function per check.** Don't combine multiple regulation checks.
- **Function names:** `check_<what>` — e.g. `check_door_width`, `check_room_area`.
- **First arg is always `model`** — an `ifcopenshell.file` object.
- **Return `list[str]`** — each string prefixed `[PASS]`, `[FAIL]`, or `[???]`.
- **No bare try/except.** Only catch specific known errors.

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
- check_* functions: (model, ...) -> list[str] with [PASS]/[FAIL]/[???] prefix

## Issue Reporting
When you encounter a contract mismatch, skill gap, or integration problem:
gh issue create --repo SerjoschDuering/iaac-bimwise-skills --label "<label>" --title "<title>"
Labels: contract-gap, skill-drift, learning, schema-change, integration-bug

## Learnings
<!-- Add here after every debugging session that reveals a recurring issue -->
```

**Keep it updated.** After any session where you hit a recurring error, add it to Learnings.
The AI gets smarter with every fix — only if you write it down.
