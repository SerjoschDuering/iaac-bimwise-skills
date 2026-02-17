---
name: IFCore
description: Use when developing on the IFCore compliance checker. Covers company context, current state, validation schema, app structure, and feature development patterns.
---

# IFCore — Company Skill

> **Living document.** Sections marked [TBD] are decided in board meetings.
> When a [TBD] is resolved, update this skill and tell your agent to adapt.

## Company Context

IFCore is building an AI-powered building compliance checker. Five teams each own a Gradio app with IFC check functions. Teams currently work independently.

**Deployment goal:** Each team deploys their Gradio app as a HuggingFace Space. Teams live in isolation — they own their dependencies and can use their own library versions. The main platform accesses them via the Gradio API. The shared validation schema is the only contract that must hold across this boundary.

**Current state (Board Meeting #1 complete):**
- 5 teams have working Gradio apps with check functions
- Shared validation schema is locked (see below)
- Platform architecture: [TBD after Board Meeting #2]

**Teams:**
| Team | Focus area | HF Space URL |
|------|-----------|--------------|
| [TBD] | [TBD] | [TBD] |
| [TBD] | [TBD] | [TBD] |
| [TBD] | [TBD] | [TBD] |
| [TBD] | [TBD] | [TBD] |
| [TBD] | [TBD] | [TBD] |

**Orchestrator:** [TBD after Board Meeting #2]

## References

- [Validation Schema](./references/validation-schema.md) — output contract for check functions exposed as API
- [Architecture](./references/architecture.md) — app structure, AGENTS.md template, code conventions
- [Development Patterns](./references/development-patterns.md) — how to plan and build new features
