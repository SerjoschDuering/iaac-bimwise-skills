---
name: pydantic-ai
description: >
  Use when building AI agents with PydanticAI. Covers agent setup, tool registration,
  structured output, multi-agent orchestration, chat/conversation patterns, system prompts,
  model configuration, streaming, error handling, logging, and testing.
  Triggers: "create an agent", "add a tool", "pydantic ai", "structured output",
  "multi-agent", "orchestrator", "agent delegation", "chat interface",
  "system prompt", "model retry", "logfire", "test agent".
version: 0.1.0
---

# PydanticAI Agent Framework

Build AI agents with type-safe structured output, tool use, and multi-agent orchestration.
PydanticAI v1.0.x (stable). Python 3.10+. Docs: [ai.pydantic.dev](https://ai.pydantic.dev/)

## Install

```bash
pip install "pydantic-ai-slim[google]"
# Set your API key
export GOOGLE_API_KEY=your_key_from_aistudio.google.com
```

## Quick Start

```python
from pydantic_ai import Agent

agent = Agent('google-gla:gemini-2.0-flash')
result = agent.run_sync('What is IFC? One sentence.')
print(result.output)
```

## Core Concepts

| Concept | What it is | One-liner |
|---|---|---|
| Agent | Wrapper around an LLM | `Agent('google-gla:gemini-2.0-flash')` |
| Tool | Function the agent can call | `@agent.tool` decorator |
| Structured output | Force LLM to return a Pydantic model | `output_type=MyModel` |
| Dependencies | Inject shared state into tools | `deps_type=MyDeps`, `RunContext[MyDeps]` |
| Chain | Output of agent A feeds agent B | Call two agents in sequence |
| Delegation | Agent A calls agent B as a tool | `await worker.run(...)` inside a tool |

## When to Use Which Pattern

```
Single agent + tools     <- Most cases. One agent, multiple tools.
Agent delegation         <- Agent A's tool calls Agent B (nested agents)
Programmatic hand-off    <- App code sequences agents (pipeline)
```

## Reference Docs

| Read when... | File |
|---|---|
| Setting up agents, registering tools, structured output, deps | [agents-and-tools.md](./references/agents-and-tools.md) |
| Multi-agent orchestration, delegation, parallel execution | [chains-and-orchestration.md](./references/chains-and-orchestration.md) |
| Multi-turn chat, message history, Gradio integration | [chat-and-conversation.md](./references/chat-and-conversation.md) |
| System prompts, dynamic prompts, model config, switching | [prompts-and-models.md](./references/prompts-and-models.md) |
| Logging, error handling, retries, streaming, testing | [production-patterns.md](./references/production-patterns.md) |

## Self-Correcting Skill

This skill is actively evolving. If any reference contains outdated APIs or patterns
that don't work, fix it immediately or ask the user to update.
