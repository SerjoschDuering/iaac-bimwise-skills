---
name: IFCore
description: Use when developing on the BIMwise IFC compliance checker. Covers company context, current state, validation schema, app structure, and feature development patterns.
---

# IFCore — Company Skill

> **Living document.** Sections marked [TBD] are decided in board meetings.
> When a [TBD] is resolved, update this skill and tell your agent to adapt.

## Company Context

BIMwise is building an AI-powered building compliance checker. Five teams each own a Gradio app with IFC check functions. Teams currently work independently. The goal is a unified platform where a main orchestrator calls each team's Gradio app as a sub-agent.

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
