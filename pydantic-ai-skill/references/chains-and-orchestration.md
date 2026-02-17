# Chains and Orchestration

## Pattern 1: Chain (Agent A Output -> Agent B Input)

The simplest multi-agent pattern. Run agents in sequence, pass results forward.

```python
from pydantic import BaseModel
from pydantic_ai import Agent

class Extract(BaseModel):
    rule: str
    actual: str

extractor = Agent('google-gla:gemini-2.0-flash', output_type=Extract)
checker = Agent('google-gla:gemini-2.0-flash', output_type=bool)

# Step 1: extract facts
a = extractor.run_sync('Door A12 width is 780mm; min required is 800mm.')

# Step 2: feed into next agent
b = checker.run_sync(f'Does it pass? rule={a.output.rule} actual={a.output.actual}')
print(f'Passed: {b.output}')
```

## Pattern 2: Agent Delegation (Agent Calls Agent as Tool)

One agent's tool internally calls another agent. The orchestrator decides when to delegate.

```python
from pydantic_ai import Agent, RunContext, UsageLimits

# Worker agent
worker = Agent('google-gla:gemini-2.0-flash', output_type=list[str])

# Orchestrator agent
orchestrator = Agent(
    'google-gla:gemini-2.0-flash',
    instructions='Use the worker_tool to get data, then summarize.',
)

@orchestrator.tool
async def worker_tool(ctx: RunContext[None], query: str) -> list[str]:
    """Delegate work to the specialist worker agent."""
    r = await worker.run(
        query,
        usage=ctx.usage,    # CRITICAL: rolls up token usage to parent
    )
    return r.output

result = orchestrator.run_sync(
    'Analyze this building.',
    usage_limits=UsageLimits(request_limit=10),
)
```

**Always pass `usage=ctx.usage`** to the delegate. Without it, the parent's `UsageLimits`
won't apply to the delegate's requests.

### Delegation with Shared Dependencies

```python
from dataclasses import dataclass
from pydantic_ai import Agent, RunContext

@dataclass
class SharedDeps:
    api_key: str
    db_url: str

orchestrator = Agent('google-gla:gemini-2.0-flash', deps_type=SharedDeps)
worker = Agent('google-gla:gemini-2.0-flash', deps_type=SharedDeps, output_type=str)

@orchestrator.tool
async def delegate_check(ctx: RunContext[SharedDeps], data: str) -> str:
    """Run a specialized check."""
    r = await worker.run(
        data,
        deps=ctx.deps,      # pass parent's deps
        usage=ctx.usage,     # pass usage tracking
    )
    return r.output
```

## Pattern 3: Programmatic Hand-off (Pipeline)

App code controls which agent runs next. Agents don't call each other.

```python
from pydantic import BaseModel
from pydantic_ai import Agent, RunUsage, UsageLimits

class FlightDetails(BaseModel):
    flight_number: str

class Failed(BaseModel):
    """Unable to find a satisfactory choice."""

search_agent = Agent(
    'google-gla:gemini-2.0-flash',
    output_type=FlightDetails | Failed,
)

seat_agent = Agent(
    'google-gla:gemini-2.0-flash',
    output_type=str,
)

async def main():
    usage = RunUsage()  # shared usage tracker across all agents

    # Step 1: search
    r1 = await search_agent.run('Find a flight to Paris', usage=usage)
    if isinstance(r1.output, Failed):
        print('No flight found')
        return

    # Step 2: seat selection (carry conversation forward)
    r2 = await seat_agent.run(
        f'Pick a window seat for {r1.output.flight_number}',
        usage=usage,
    )
    print(r2.output)
```

**Key difference from delegation:** In hand-off, `usage` is a `RunUsage()` object you
create once. In delegation, it's `ctx.usage` from the tool's `RunContext`.

## Pattern 4: Parallel Execution

### Multiple Independent Agents via asyncio.gather

```python
import asyncio
from pydantic_ai import Agent

agent_a = Agent('google-gla:gemini-2.0-flash', output_type=str)
agent_b = Agent('google-gla:gemini-2.0-flash', output_type=str)

async def analyze(data: str) -> dict:
    result_a, result_b = await asyncio.gather(
        agent_a.run(f'Perspective A: {data}'),
        agent_b.run(f'Perspective B: {data}'),
    )
    return {'a': result_a.output, 'b': result_b.output}
```

### Orchestrator-Workers (Dynamic Fan-out)

```python
from pydantic import BaseModel
from pydantic_ai import Agent

class Plan(BaseModel):
    sections: list[str]

planner = Agent('google-gla:gemini-2.0-flash', output_type=Plan)
writer = Agent('google-gla:gemini-2.0-flash', output_type=str)

async def write_article(topic: str) -> str:
    plan = await planner.run(f'Plan sections for: {topic}')
    tasks = [writer.run(f'Write: {s}') for s in plan.output.sections]
    results = await asyncio.gather(*tasks)
    return '\n\n'.join(r.output for r in results)
```

### Parallel Tool Calls Within an Agent

When the LLM returns multiple tool calls in one response, PydanticAI runs them
concurrently via `asyncio.create_task`. This is automatic.

Force sequential execution when tools have side effects:

```python
@agent.tool(sequential=True)
async def write_to_db(ctx: RunContext, data: str) -> str:
    """This tool runs sequentially even if called in parallel."""
    ...
```

## Best Practices

1. **Always pass `usage=ctx.usage`** in delegation tools.
2. **Use `output_type=Success | Failed`** for explicit failure handling.
3. **Keep agents focused** — one agent per concern. Separation makes testing easy.
4. **Set `UsageLimits`** in production to prevent infinite loops.
5. **Use `message_history=result.new_messages()`** to carry context across turns.

## Anti-patterns

- **Circular delegation** (A calls B calls A) — no cycle detection, loops until limits hit.
- **Forgetting `async def`** on tools that `await` delegate agents.
- **Shared mutable state** in parallel tools — use `sequential=True` or locks.
- **Skipping `UsageLimits`** — an agent can make hundreds of requests without limits.

## Official Docs

- [Multi-Agent Patterns](https://ai.pydantic.dev/multi-agent-applications/)
- [Agents](https://ai.pydantic.dev/agent/)
- [Advanced Tool Features](https://ai.pydantic.dev/tools-advanced/)
