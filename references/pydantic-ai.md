# PydanticAI

Model: `google-gla:gemini-2.0-flash`

## Agent with structured output

```python
from pydantic import BaseModel
from pydantic_ai import Agent

class CheckResult(BaseModel):
    element_id: str
    element_type: str
    element_name: str
    rule: str
    requirement: str
    actual_value: str
    passed: bool | None

class ComplianceReport(BaseModel):
    results: list[CheckResult]
    summary: str

agent = Agent(
    "google-gla:gemini-2.0-flash",
    system_prompt="You are a building compliance checker.",
    output_type=ComplianceReport,
)
```

## Registering tools

```python
from pydantic_ai import RunContext

@agent.tool
def check_doors(ctx: RunContext[None], ifc_path: str) -> str:
    """Check all doors against accessibility requirements."""
    model = ifcopenshell.open(ifc_path)
    results = [check_door_width(door) for door in model.by_type("IfcDoor")]
    return str(results)
```

## Chain (agent A feeds into agent B)

```python
extractor = Agent("google-gla:gemini-2.0-flash", output_type=Extract)
checker   = Agent("google-gla:gemini-2.0-flash", output_type=bool)

a = extractor.run_sync("Door A12 width is 780mm; min is 800mm.")
b = checker.run_sync(f"Does it pass? rule={a.output.rule} actual={a.output.actual}")
```
