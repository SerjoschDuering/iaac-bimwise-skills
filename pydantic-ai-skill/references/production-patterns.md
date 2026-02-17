# Production Patterns

## Error Handling and Retries

### ModelRetry (ask the LLM to correct itself)

```python
from pydantic_ai import Agent, RunContext, ModelRetry

agent = Agent('google-gla:gemini-2.0-flash')

@agent.tool(retries=3)
def get_user(ctx: RunContext, name: str) -> int:
    """Get a user's ID from their full name."""
    user_id = ctx.deps.users.get(name=name)
    if user_id is None:
        raise ModelRetry(f'No user named {name!r}. Use the exact full name.')
    return user_id
```

When `ModelRetry` is raised, the error message goes back to the LLM as context.
`ctx.retry` gives you the current retry count (0-based).

Pydantic `ValidationError` on tool args also triggers the same retry path automatically.

### Retry Configuration

```python
# Agent-level default (all tools + output validation)
agent = Agent('google-gla:gemini-2.0-flash', retries=3)

# Tool-level override
@agent.tool(retries=5)
def flaky_tool(ctx: RunContext, x: str) -> str: ...

# Separate output retries
agent = Agent('google-gla:gemini-2.0-flash', retries=2, output_retries=4)
```

### UsageLimits (prevent infinite loops)

```python
from pydantic_ai import Agent, UsageLimits
from pydantic_ai.exceptions import UsageLimitExceeded

agent = Agent('google-gla:gemini-2.0-flash')

try:
    result = agent.run_sync(
        'Do something complex.',
        usage_limits=UsageLimits(
            request_limit=5,         # max LLM API calls
            tool_calls_limit=10,     # max tool executions
            total_tokens_limit=4000, # max tokens
        ),
    )
except UsageLimitExceeded as e:
    print(f'Limit exceeded: {e}')
```

Default `request_limit` is 50. Always set limits in production.

### Exception Hierarchy

```
AgentRunError
  ├── UnexpectedModelBehavior   # model returned garbage
  └── UsageLimitExceeded        # token/request cap hit
ModelRetry                       # tool-internal, not usually caught by app
UserError                        # developer mistake (wrong API usage)
```

## Logging / Observability (Logfire)

### Setup

```bash
pip install "pydantic-ai-slim[logfire]"
logfire auth
logfire projects new
```

### Basic Integration

```python
import logfire
from pydantic_ai import Agent

logfire.configure()
logfire.instrument_pydantic_ai()  # instruments ALL agents globally

agent = Agent('google-gla:gemini-2.0-flash', instructions='Be concise.')
result = agent.run_sync('Hello')
```

### What Gets Logged Automatically

| What | Detail |
|---|---|
| Agent run spans | Complete trace per `agent.run()` |
| Model request spans | Each LLM API call |
| Tool execution spans | Every tool call and result |
| Token metrics | `gen_ai.usage.input_tokens`, `output_tokens` |
| Messages | System instructions, input/output messages |

### Non-Logfire OpenTelemetry Backend

```python
import os
os.environ['OTEL_EXPORTER_OTLP_ENDPOINT'] = 'http://localhost:4318'
logfire.configure(send_to_logfire=False)
logfire.instrument_pydantic_ai()
```

Works with Langfuse, W&B Weave, Arize, SigNoz, mlflow, etc.

### DIY Logging (Without Logfire)

```python
result = agent.run_sync('Check compliance.')
u = result.usage()
print(f'Requests: {u.requests}, Tokens: {u.total_tokens}')
```

## Streaming

### Text Streaming

```python
async with agent.run_stream('Tell me about Paris.') as response:
    async for text in response.stream_text():
        print(text, end='', flush=True)
    print(response.usage())
```

`stream_text(delta=True)` for incremental chunks only.

### Structured Output Streaming

```python
async with agent.run_stream('Describe Paris.') as response:
    async for partial in response.stream_output():
        print(partial)  # partial Pydantic model, grows as tokens arrive
```

### Sync Streaming

```python
with agent.run_stream_sync('Hello') as response:
    for text in response.stream_text():
        print(text, end='')
```

## Testing

### Strategy

Use `TestModel` or `FunctionModel` instead of real LLMs. Use `agent.override()` to
inject the test model. Block real LLM calls globally.

### Global Safety Guard

```python
# conftest.py
from pydantic_ai import models
models.ALLOW_MODEL_REQUESTS = False  # any real LLM call raises RuntimeError
```

### TestModel

```python
from pydantic_ai.models.test import TestModel

def test_my_agent():
    with my_agent.override(model=TestModel()):
        result = my_agent.run_sync('Hello')
        assert result.output is not None
```

`TestModel` calls ALL tools by default. Restrict with `TestModel(call_tools=['specific_tool'])`.

### FunctionModel (Full Control)

```python
from pydantic_ai.models.function import FunctionModel, AgentInfo
from pydantic_ai.messages import ModelMessage, ModelResponse, TextPart, ToolCallPart

def fake_model(messages: list[ModelMessage], info: AgentInfo) -> ModelResponse:
    if len(messages) == 1:
        # First call: invoke a tool
        return ModelResponse(parts=[
            ToolCallPart('check_door', {'door_id': 'D1', 'width': 780})
        ])
    else:
        # After tool result: return final answer
        return ModelResponse(parts=[TextPart('Door D1 fails.')])

with agent.override(model=FunctionModel(fake_model)):
    result = agent.run_sync('Check doors.')
    assert 'fails' in result.output
```

### Pytest Fixture Pattern

```python
import pytest
from pydantic_ai import models
from pydantic_ai.models.test import TestModel

@pytest.fixture(autouse=True)
def block_real_llm():
    original = models.ALLOW_MODEL_REQUESTS
    models.ALLOW_MODEL_REQUESTS = False
    yield
    models.ALLOW_MODEL_REQUESTS = original

@pytest.fixture
def test_agent():
    with my_agent.override(model=TestModel()):
        yield
```

### Message Inspection

```python
from pydantic_ai import capture_run_messages

with capture_run_messages() as messages:
    with agent.override(model=TestModel()):
        result = agent.run_sync('Check compliance.')

# messages contains the full ModelRequest/ModelResponse exchange
assert len(messages) >= 2
```

## Result Inspection

```python
result = agent.run_sync('Check compliance.')

result.output              # typed output
result.usage()             # RunUsage(requests=1, input_tokens=62, ...)
result.usage().total_tokens  # input_tokens + output_tokens
result.all_messages()      # complete conversation
result.new_messages()      # only this run's messages
result.timestamp           # when the run completed
result.run_id              # unique identifier
```

### Aggregating Usage Across Runs

```python
from pydantic_ai.usage import RunUsage

total = RunUsage()
for prompt in prompts:
    result = agent.run_sync(prompt)
    total += result.usage()
print(f'Total tokens: {total.total_tokens}')
```

## Gotchas

- `result.usage()` is a method call (parentheses), not a property.
- `TestModel` calls ALL tools by default — restrict with `call_tools=`.
- `run_stream()` is an async context manager — use `async with`, not `await`.
- `ALLOW_MODEL_REQUESTS = False` is global — reset in fixtures.
- Default `retries=1`, not 0. Set `retries=0` to disable.

## Official Docs

- [Testing](https://ai.pydantic.dev/testing/)
- [Logfire Integration](https://ai.pydantic.dev/logfire/)
- [Error Handling / Retries](https://ai.pydantic.dev/retries/)
- [API: Usage](https://ai.pydantic.dev/api/usage/)
