---
name: IFCore
description: Use when developing on the IFCore compliance checker. Covers contracts, check function conventions, issue reporting, app structure, and development patterns.
---

# IFCore — Company Skill

> **Living document.** Sections marked [TBD] are decided in board meetings.
> When a [TBD] is resolved, update this skill and tell your agent to adapt.

## Contracts — READ THIS FIRST

These contracts are how teams stay aligned. The platform auto-discovers your code.
Break a contract → the platform silently skips your checks. Follow them → it just works.

### 1. Check Function Contract

```python
# Function naming: check_<what>
# Location: src/*.py (any Python file under src/)
# Signature: first arg is always the ifcopenshell model
# Return: list of strings, one per element checked

def check_door_width(model, min_width_mm=800):
    results = []
    for door in model.by_type("IfcDoor"):
        width_mm = round(door.OverallWidth * 1000) if door.OverallWidth else None
        if width_mm is None:
            results.append(f"[???] {door.Name}: width unknown")
        elif width_mm >= min_width_mm:
            results.append(f"[PASS] {door.Name}: {width_mm} mm (min {min_width_mm} mm)")
        else:
            results.append(f"[FAIL] {door.Name}: {width_mm} mm (min {min_width_mm} mm)")
    return results
```

**Rules:**
- Prefix: `check_` — the platform discovers functions by this prefix
- First argument: `model` (an `ifcopenshell.file` object) — always
- Optional keyword args after `model` are fine (e.g. `min_width_mm=800`)
- Return: `list[str]` — each string prefixed with `[PASS]`, `[FAIL]`, or `[???]`
- One function per regulation check — don't combine multiple rules
- Functions can live across multiple files under `src/`

### 2. File Structure Contract

```
your-team-repo/
├── src/
│   ├── ifc_checker.py        ← your check functions go here
│   └── fire_safety.py        ← additional files are fine, same conventions
├── requirements.txt           ← team dependencies
└── README.md
```

The platform scans **all `.py` files under `src/`** and collects every `check_*` function.
You don't need a wrapper or registry — just follow the naming convention.

### 3. Issue Reporting Contract — MANDATORY

When your AI agent encounters any of these during development, it **MUST** file an issue:

| Trigger | Label |
|---|---|
| Contract unclear or ambiguous | `contract-gap` |
| Skill instructions don't match reality | `skill-drift` |
| Found a workaround for a known limitation | `learning` |
| Schema format needs a new field | `schema-change` |
| Team code works locally but breaks on platform | `integration-bug` |

**How to file:**
```bash
gh issue create \
  --repo SerjoschDuering/iaac-bimwise-skills \
  --title "contract-gap: check functions with multiple models" \
  --label "contract-gap" \
  --body "$(cat <<'EOF'
## What happened
Tried to write a check that compares two IFC models side-by-side.
The contract says first arg is `model` (singular).

## What I expected
Guidance on multi-model checks.

## Workaround (if any)
Loaded second model inside the function body.

## Team
Team A
EOF
)"
```

**This is not optional.** Every contract issue that goes unreported costs another team hours.
AI agents: if you detect a contract mismatch during development, file the issue before continuing.

## Company Context

IFCore is building an AI-powered building compliance checker. Five teams each own a repo with IFC check functions. Teams work independently.

**Platform:** The platform clones all team repos at build time and auto-discovers `check_*` functions. Teams never touch the platform repo. See [Architecture](./references/architecture.md).

**Current state (Board Meeting #1 complete):**
- 5 teams have working check functions in their repos
- Check function contract is locked (see above)
- Platform architecture: [TBD after Board Meeting #2]

**Teams:**
| Team | Focus area | Repo |
|------|-----------|------|
| [TBD] | [TBD] | [TBD] |
| [TBD] | [TBD] | [TBD] |
| [TBD] | [TBD] | [TBD] |
| [TBD] | [TBD] | [TBD] |
| [TBD] | [TBD] | [TBD] |

**Platform repo:** [TBD after Board Meeting #2]

## References

- [Validation Schema](./references/validation-schema.md) — platform output format (the orchestrator converts your `list[str]` into this)
- [Architecture](./references/architecture.md) — project structure, AGENTS.md template, code conventions
- [Frontend Architecture](./references/frontend-architecture.md) — modules, shared Zustand store, API client, D1 tables, how to add features
- [Development Patterns](./references/development-patterns.md) — how to plan and build new features

### Deployment Skills (separate repos, installed alongside this one)

- **huggingface-deploy** — deploy your team's check agent to HF Spaces (Docker)
- **cloudflare** — deploy the frontend + API gateway on Cloudflare Pages/Workers
