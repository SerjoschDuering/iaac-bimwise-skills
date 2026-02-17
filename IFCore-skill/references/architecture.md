# Architecture & Conventions

## Gradio App Structure

```
app/
├── app_simple.py      # text-only interface (Day 1)
├── app.py             # full interface with 3D viewer (Day 2+)
└── src/
    ├── ifc_checker.py     # YOUR ONLY EDIT TARGET — check functions live here
    └── ifc_visualizer.py  # 3D export (do not touch)
```

## Code Conventions

- **Max 300 lines per file.** Split into modules when approaching the limit.
- **One function per check.** Don't combine multiple regulation checks.
- **Function names:** `check_<what>` — e.g. `check_door_width`, `check_room_area`.
- **No bare try/except.** Only catch specific known errors.
- **All endpoint functions** return the validation schema format.

## AGENTS.md / CLAUDE.md

Every team MUST have this file in their repo root. Your AI assistant reads it automatically.

**Required line:**
```
Always read the IFCore skill before developing on this project.
```

**Template:**
```markdown
# <Project Name>

Always read the IFCore skill before developing on this project.

## Structure
<paste your app/ directory tree here>

## Conventions
- Max 300 lines per file
- One function per regulation check
- All endpoint functions return the IFCore validation schema

## Learnings
<!-- Add here after every debugging session that reveals a recurring issue -->
```

**Keep it updated.** After any session where you hit a recurring error, add it to Learnings.
The AI gets smarter with every fix — only if you write it down.
