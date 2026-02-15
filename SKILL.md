---
name: ifc-compliance-checker
description: Use when writing IFC compliance checks, building checking agents, or working with the project structure. Covers the shared data schema, project conventions, AGENTS.md patterns, and PydanticAI agent setup.
---

# IFC Compliance Checker

Company standards for building compliance checking against Spanish CTE regulations.

## Data Schema

All check functions must return results in the shared format.
See [references/validation-schema.md](./references/validation-schema.md).

```python
{"element_id": str, "element_type": str, "element_name": str,
 "rule": str, "requirement": str, "actual_value": str, "passed": bool | None}
```

## Project Structure & Conventions

Local app structure, code conventions (max 300 lines), and how to maintain
an AGENTS.md / CLAUDE.md file with learnings.
See [references/architecture.md](./references/architecture.md).

## PydanticAI

Agent setup, structured output, tools, and chains.
See [references/pydantic-ai.md](./references/pydantic-ai.md).
