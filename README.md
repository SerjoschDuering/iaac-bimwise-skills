# IFCore Company Skills

This repository contains **Agent Skills** for the IFCore project.

## What is an Agent Skill?

An Agent Skill is a knowledge file your AI coding assistant reads automatically. It tells the AI about your project's standards, conventions, and patterns — so it follows them without you having to explain every time.

When a skill is installed, your AI assistant knows:
- The company context and current state of the project
- The shared data schema all team apps must follow
- How to structure new features
- What conventions to apply

## Skills in this repo

| Skill | Description |
|-------|-------------|
| [IFCore-skill](./IFCore-skill/) | Core skill for all IFCore development — company context, validation schema, architecture, feature patterns |

## Install

Copy the prompt below and give it to your AI coding assistant (Cursor, Claude, Copilot, or any other):

```
I want to install the following repository as an agent skill that I can use across all my projects.
Skill repository: https://github.com/SerjoschDuering/iaac-bimwise-skills
Please install the IFCore-skill from this repository as an agent skill and make it available in my current project.
```

Your AI will handle the installation for your specific tool.
