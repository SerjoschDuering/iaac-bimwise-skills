# IFC Compliance Checker -- Agent Skill

Company engineering standards for our IFC compliance checking startup.

This is an [Agent Skill](https://code.visualstudio.com/docs/copilot/customization/agent-skills) --
a set of instructions your AI coding assistant loads automatically when relevant.

## Install

Copy this repository URL and tell your AI agent:

> Please install this as an agent skill: `<repo-url>`

Works with GitHub Copilot, Claude Code, Cursor, and other agents that support the Agent Skills standard.

## What's Inside

| File | What it covers |
|------|---------------|
| `SKILL.md` | Entry point -- overview and links to references |
| `references/validation-schema.md` | Shared output format for all check functions |
| `references/architecture.md` | Platform architecture (frontend, orchestrator, team agents) |
| `references/pydantic-ai.md` | PydanticAI agent setup, tools, structured output |
