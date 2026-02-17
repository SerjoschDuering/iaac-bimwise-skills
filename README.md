# IFCore Company Skills

This repository contains **Agent Skills** for the IFCore project.

## What is an Agent Skill?

An Agent Skill is an open standard for giving AI coding assistants specialized knowledge. It works across tools — GitHub Copilot, Claude Code, Cursor, and others.

The key concept is **progressive disclosure**: the AI doesn't load everything at once.

| Level | What loads | When |
|-------|-----------|------|
| 1 — Discovery | `name` + `description` from frontmatter only | Always — just lightweight metadata |
| 2 — Instructions | Full `SKILL.md` body | When your request matches the skill description |
| 3 — Resources | Files in `references/` | Only when the AI needs them |

This means you can have many skills installed without bloating your context. The AI knows what's available and pulls in detail only when relevant.

## Skills in this repo

| Skill | Description |
|-------|-------------|
| [IFCore-skill](./IFCore-skill/) | Use when developing on the IFCore compliance checker — company context, validation schema, architecture, feature patterns |

## Install

Copy the prompt below and give it to your AI coding assistant:

```
Please install the following Agent Skills repository into my current project so I can use it with my AI coding assistant:

Repository: https://github.com/SerjoschDuering/iaac-bimwise-skills

Install the IFCore-skill (found in the IFCore-skill/ folder) by placing it in the correct skills directory for my tool (.github/skills/, .claude/skills/, or equivalent). Each skill is a folder with a SKILL.md file — keep that structure intact.
```

Your AI will handle the rest.
