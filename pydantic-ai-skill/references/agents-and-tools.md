# Agents and Tools

## Creating an Agent

```python
from pydantic_ai import Agent

agent = Agent(
    'google-gla:gemini-2.0-flash',  # model string
    output_type=str,                  # default: plain text
    instructions='You are a helpful assistant.',  # NOT in message_history
    deps_type=None,                   # type for dependency injection
    retries=1,                        # retries on validation errors
)
```

Key constructor params: `model`, `output_type`, `instructions`, `system_prompt`,
`deps_type`, `tools`, `model_settings`, `retries`, `end_strategy`.

**`instructions` vs `system_prompt`:** Use `instructions` (default). It is NOT stored
in `message_history` between runs. Use `system_prompt` only when the prompt must
survive being passed as `message_history` to another agent.

## Running an Agent

```python
# Synchronous (scripts, notebooks, Gradio)
result = agent.run_sync('What is 2 + 2?')

# Async (FastAPI, async code)
result = await agent.run('What is 2 + 2?')

# Streaming (chat UIs)
async with agent.run_stream('Tell me about Paris.') as response:
    async for text in response.stream_text():
        print(text, end='')
```

`run_sync()` and `run()` return `AgentRunResult` with: `result.output`, `result.usage()`,
`result.all_messages()`, `result.new_messages()`.
`run_stream()` returns a context manager yielding `StreamedRunResult`.

## Tool Registration

Three ways to register tools:

```python
# 1. @agent.tool — receives RunContext (access deps, usage, retry count)
@agent.tool
def get_data(ctx: RunContext[MyDeps], query: str) -> str:
    """Fetch data from the database.

    Args:
        query: The search query string.
    """
    return ctx.deps.db.query(query)

# 2. @agent.tool_plain — no context, simpler
@agent.tool_plain
def calculate_area(width: float, height: float) -> float:
    """Calculate area in square meters.

    Args:
        width: Width in millimeters.
        height: Height in millimeters.
    """
    return (width * height) / 1_000_000

# 3. Via constructor
agent = Agent('google-gla:gemini-2.0-flash', tools=[get_data, calculate_area])
```

**Docstrings matter.** PydanticAI parses them (Google/NumPy/Sphinx styles) to generate
tool descriptions and parameter docs for the LLM. Write clear docstrings.

### Tool Decorator Options

```python
@agent.tool(
    retries=3,                         # per-tool retry count
    require_parameter_descriptions=True, # error if params undocumented
    sequential=True,                    # serialize execution (no parallel)
    timeout=30.0,                       # seconds
)
def my_tool(ctx: RunContext[MyDeps], param: str) -> str:
    ...
```

### ModelRetry — Ask the LLM to Try Again

```python
from pydantic_ai import ModelRetry

@agent.tool_plain
def validate_input(value: str) -> str:
    """Validate the input value."""
    if not value.isalpha():
        raise ModelRetry('Value must contain only letters. Please try again.')
    return value
```

`ModelRetry` sends the error message back to the LLM, which then retries with corrected args.

## Structured Output

Force the LLM to return a specific shape:

```python
from pydantic import BaseModel
from pydantic_ai import Agent

class CheckResult(BaseModel):
    check_name: str
    passed: bool
    message: str

agent = Agent('google-gla:gemini-2.0-flash', output_type=CheckResult)
result = agent.run_sync('Check: door is 780mm, minimum is 800mm.')
print(result.output.passed)    # False
print(result.output.message)   # "780mm is below minimum 800mm"
```

### Union Types (Multiple Valid Outputs)

```python
class Success(BaseModel):
    data: str

class Failure(BaseModel):
    reason: str

agent = Agent('google-gla:gemini-2.0-flash', output_type=[Success, Failure])
result = agent.run_sync('...')
if isinstance(result.output, Success):
    print(result.output.data)
```

### Output Validators

```python
@agent.output_validator
async def validate(ctx: RunContext, output: CheckResult) -> CheckResult:
    if output.check_name == '':
        raise ModelRetry('check_name must not be empty.')
    return output
```

## Dependencies (RunContext)

Inject shared state into tools without global variables:

```python
from dataclasses import dataclass
from pydantic_ai import Agent, RunContext

@dataclass
class MyDeps:
    api_key: str
    db_connection: object

agent = Agent('google-gla:gemini-2.0-flash', deps_type=MyDeps)

@agent.tool
async def query_db(ctx: RunContext[MyDeps], sql: str) -> list[dict]:
    """Execute a SQL query."""
    return await ctx.deps.db_connection.execute(sql)

# Pass instance at runtime
deps = MyDeps(api_key='abc', db_connection=my_db)
result = agent.run_sync('Find all doors', deps=deps)
```

`RunContext` attributes: `ctx.deps`, `ctx.retry` (current retry count),
`ctx.usage` (token tracking), `ctx.messages` (history so far).

## Gotchas

- `@agent.tool` requires `RunContext` as first param. Forget it? Use `@agent.tool_plain`.
- `deps_type` takes the **class** (`MyDeps`), not an instance (`MyDeps()`).
- Tool params must be JSON-serializable: `str`, `int`, `float`, `bool`, `list`, `dict`, Pydantic models.
- Default `retries=1`. Set higher for complex structured output.
- `run_sync()` creates an event loop — don't call it from inside `async` code. Use `await agent.run()`.

## Official Docs

- [Agents](https://ai.pydantic.dev/agent/)
- [Function Tools](https://ai.pydantic.dev/tools/)
- [Output](https://ai.pydantic.dev/output/)
- [Dependencies](https://ai.pydantic.dev/dependencies/)
