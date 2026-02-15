# Project Structure & Conventions

## Gradio App Structure

```
app/
├── app_simple.py          # Simple text-only interface
├── app.py                 # Full interface with 3D viewer
└── src/
    ├── ifc_checker.py     # YOUR CHECK FUNCTIONS GO HERE
    └── ifc_visualizer.py  # 3D export (don't touch)
```

You only edit `src/ifc_checker.py`. The Gradio apps import from it.

## AGENTS.md / CLAUDE.md

Every team MUST maintain an instructions file in their repo root.
Use `AGENTS.md` (Copilot) or `CLAUDE.md` (Claude Code) -- your AI assistant reads it automatically.

This file should contain:
- What the project does (one sentence)
- File structure overview
- Conventions (see below)
- **Learnings** -- when you hit a recurring problem, add it here so the AI doesn't repeat the mistake

### Example

```markdown
# <Project Name>

## Structure
- Put the `app/` directory tree here so the AI knows the project layout
- <file A> -- <what it does>
- <file B> -- <what it does>

## Conventions
- <convention A>
- <convention B>

## Learnings
- <learning A>
- <learning B>
```

### Keep It Updated

After every debugging session where you learn something new, add it to the Learnings section.
This is how our company's knowledge compounds -- the AI gets smarter with every fix.

## Code Conventions

- **Max 300 lines per file.** Split into modules when approaching the limit.
- **No try/except blocks** unless you're handling a specific known error.
- **All check functions** return the validation schema format (see validation-schema.md).
- **One function per regulation check.** Don't combine multiple checks into one function.
- **Function names:** `check_<what>` -- e.g. `check_door_width`, `check_room_area`.
