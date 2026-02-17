# Chat and Conversation

## Multi-Turn Conversations

Pass `message_history` to continue a conversation across multiple runs:

```python
from pydantic_ai import Agent

agent = Agent('google-gla:gemini-2.0-flash', instructions='You are a helpful assistant.')

result1 = agent.run_sync('Tell me a joke.')
print(result1.output)

# Continue the conversation
result2 = agent.run_sync(
    'Explain why that is funny.',
    message_history=result1.new_messages(),
)
print(result2.output)
```

**Use `new_messages()` (not `all_messages()`)** when chaining — avoids duplicating
system prompts.

**`instructions` vs `system_prompt` behavior with history:**
- `instructions`: Always included fresh, regardless of `message_history`. **Recommended.**
- `system_prompt`: NOT regenerated if `message_history` is non-empty (history is assumed to include one).

## Accessing Messages

```python
result = agent.run_sync('Hello')

result.output              # the typed output
result.all_messages()      # all messages including prior history
result.new_messages()      # only messages from this run
result.all_messages_json() # serialized to JSON bytes
result.usage()             # RunUsage(requests=1, input_tokens=62, ...)
```

## Serializing Message History (for Storage)

```python
from pydantic_ai import ModelMessagesTypeAdapter

# Serialize
history = result.all_messages()
as_json = result.all_messages_json()

# Deserialize
restored = ModelMessagesTypeAdapter.validate_json(as_json)

# Resume from stored history
result = agent.run_sync('Follow-up', message_history=restored)
```

## History Processors (Token Budget)

Trim message history before sending to the model:

```python
from pydantic_ai import Agent
from pydantic_ai.messages import ModelMessage

def keep_recent(messages: list[ModelMessage]) -> list[ModelMessage]:
    """Keep only the last 10 messages."""
    return messages[-10:]

agent = Agent(
    'google-gla:gemini-2.0-flash',
    history_processors=[keep_recent],
)
```

## Gradio Chat Integration

### Pattern A: Simplest (Single-User, Course/Prototyping)

Keep a mutable list as the message store. Works well in single-user contexts:

```python
import gradio as gr
from pydantic_ai import Agent
from pydantic_ai.messages import ModelMessage

agent = Agent(
    'google-gla:gemini-2.0-flash',
    instructions='You are a building code compliance assistant.',
)

conversation: list[ModelMessage] = []

async def respond(message: str, history: list[dict]) -> str:
    global conversation
    result = await agent.run(message, message_history=conversation)
    conversation = result.all_messages()
    return result.output

def reset():
    global conversation
    conversation = []

with gr.Blocks(title='Compliance Checker') as demo:
    gr.ChatInterface(fn=respond, type='messages')
    gr.Button('Clear conversation').click(fn=reset)

demo.launch()
```

### Pattern B: Streaming Chat

```python
import gradio as gr
from pydantic_ai import Agent
from pydantic_ai.messages import ModelMessage

agent = Agent('google-gla:gemini-2.0-flash', instructions='Be helpful.')
conversation: list[ModelMessage] = []

async def respond(message: str, history: list[dict]):
    global conversation
    async with agent.run_stream(message, message_history=conversation) as result:
        async for text in result.stream_text():
            yield text
        conversation = result.all_messages()

gr.ChatInterface(fn=respond, type='messages').launch()
```

### Pattern C: With Tools (Full Agent Chat)

```python
import gradio as gr
from pydantic_ai import Agent, RunContext
from pydantic_ai.messages import ModelMessage

agent = Agent(
    'google-gla:gemini-2.0-flash',
    instructions='You check buildings. Use check_door_width when asked about doors.',
)

@agent.tool_plain
def check_door_width(door_name: str, width_mm: float, min_mm: float) -> str:
    """Check if a door meets the minimum width requirement.

    Args:
        door_name: Name of the door element.
        width_mm: Actual door width in millimeters.
        min_mm: Minimum required width in millimeters.
    """
    passed = width_mm >= min_mm
    return f'{"PASS" if passed else "FAIL"}: {door_name} is {width_mm}mm (min {min_mm}mm)'

conversation: list[ModelMessage] = []

async def respond(message: str, history: list[dict]) -> str:
    global conversation
    result = await agent.run(message, message_history=conversation)
    conversation = result.all_messages()
    return result.output

gr.ChatInterface(fn=respond, type='messages', title='BIMwise').launch()
```

## Gotchas

| Issue | Solution |
|---|---|
| System prompt doubled on multi-turn | Use `instructions` (not `system_prompt`), or use `new_messages()` |
| Agent loses context between turns | Store `result.all_messages()` and pass back on next run |
| Token budget exceeded | Add `history_processors=[keep_recent]` to Agent |
| Gradio history != PydanticAI history | Maintain a separate PydanticAI message list |
| `stream_text(delta=True)` drops final message | Use `stream_text()` (no delta) if you need `all_messages()` after |

## Official Docs

- [Messages and Chat History](https://ai.pydantic.dev/message-history/)
- [Chat App Example](https://ai.pydantic.dev/examples/chat-app/)
- [Gradio ChatInterface](https://www.gradio.app/docs/gradio/chatinterface)
