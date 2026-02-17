# Prompts and Models

## System Prompts

### Static (at construction)

```python
from pydantic_ai import Agent

agent = Agent(
    'google-gla:gemini-2.0-flash',
    instructions='You are a building compliance checker. Units are millimeters.',
)
```

### Dynamic (evaluated at run time)

Access dependencies, dates, or runtime state:

```python
from datetime import date
from pydantic_ai import Agent, RunContext

agent = Agent('google-gla:gemini-2.0-flash', deps_type=str)

@agent.instructions
def add_context(ctx: RunContext[str]) -> str:
    return f'Today is {date.today()}. Project: {ctx.deps}.'

result = agent.run_sync('Check compliance.', deps='Duplex Apartment')
```

Multiple decorators allowed — they append in definition order.
Returning empty string skips that part.

### instructions vs system_prompt

| | `instructions` | `system_prompt` |
|---|---|---|
| In `message_history` | No | Yes |
| Regenerated on new runs | Yes (always fresh) | No (if history present) |
| Best for | Single-agent (default choice) | Cross-agent context sharing |

**Use `instructions` as your default.** Only use `system_prompt` when the instruction
must survive being passed as `message_history` to another agent.

## Prompt Best Practices for Tool-Using Agents

**1. Be explicit about when to use tools:**
```python
agent = Agent(
    'google-gla:gemini-2.0-flash',
    instructions="""You check IFC building models against regulations.

ALWAYS use the check tools to verify dimensions — never estimate from memory.
ALWAYS use lookup_regulation before citing any regulation number.
When all checks are complete, summarize results as a structured list.""",
)
```

**2. Describe expected output in the prompt** (especially with structured output):
```python
agent = Agent(
    'google-gla:gemini-2.0-flash',
    output_type=ComplianceReport,
    instructions="""For each element, run the appropriate check tool.
Return results with status (PASS/FAIL), regulation clause, and measured vs required values.""",
)
```

**3. Separate static facts from dynamic context:**
```python
agent = Agent(
    'google-gla:gemini-2.0-flash',
    instructions='You are a building compliance checker. Units are always millimeters.',
)

@agent.instructions
def add_project(ctx: RunContext[ProjectDeps]) -> str:
    return f'Project: {ctx.deps.project_name}. Regulation set: {ctx.deps.regulation_version}.'
```

**4. Write clear tool docstrings** — they become the tool schema description:
```python
@agent.tool_plain
def get_wall_thickness(wall_id: str) -> float:
    """Get the thickness of a wall in millimeters from the IFC model.

    Args:
        wall_id: The IFC GlobalId of the wall element.
    """
    ...
```

## Model Configuration

### Model Strings

```
"<provider>:<model-name>"
```

| Provider | Format | Auth |
|---|---|---|
| Google GLA | `google-gla:gemini-2.0-flash` | `GOOGLE_API_KEY` env var |
| Google Vertex | `google-vertex:gemini-2.0-flash` | `gcloud` ADC |
| OpenAI | `openai:gpt-4o` | `OPENAI_API_KEY` env var |
| Anthropic | `anthropic:claude-sonnet-4-5` | `ANTHROPIC_API_KEY` env var |

### Gemini Models (used in this course)

```python
agent = Agent('google-gla:gemini-2.0-flash')  # fast, cheap, good enough
```

Install: `pip install "pydantic-ai-slim[google]"`
Key from: [aistudio.google.com/apikey](https://aistudio.google.com/apikey)

### Model Settings

```python
from pydantic_ai import Agent
from pydantic_ai.settings import ModelSettings

agent = Agent(
    'google-gla:gemini-2.0-flash',
    model_settings=ModelSettings(
        temperature=0.0,     # deterministic for compliance checking
        max_tokens=2048,
    ),
)

# Override per-run
result = agent.run_sync(
    'Check this.',
    model_settings=ModelSettings(temperature=0.7),
)
```

### Google-Specific Settings

```python
from pydantic_ai.models.google import GoogleModelSettings

settings = GoogleModelSettings(
    temperature=0.2,
    max_tokens=4096,
    google_thinking_config={'thinking_level': 'low'},
)
```

### Model Switching at Runtime

```python
# Different model for a specific run
result = agent.run_sync('Complex analysis.', model='google-gla:gemini-2.5-flash')
```

### Fallback Model

```python
from pydantic_ai.models.fallback import FallbackModel

fallback = FallbackModel(
    'google-gla:gemini-2.0-flash',     # try first
    'google-gla:gemini-2.5-flash',     # fallback
)
agent = Agent(fallback)
```

## Official Docs

- [Agents](https://ai.pydantic.dev/agent/) — system prompts, instructions
- [Google Models](https://ai.pydantic.dev/models/google/) — Gemini config
- [Models Overview](https://ai.pydantic.dev/models/overview/) — all providers
