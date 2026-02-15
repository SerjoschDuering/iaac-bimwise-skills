# Validation Schema

All check functions MUST return this format.

| Field | Type | Description |
|-------|------|-------------|
| `element_id` | str | IFC GlobalId |
| `element_type` | str | e.g. `"IfcDoor"`, `"IfcSpace"` |
| `element_name` | str | Human-readable. Fallback: `f"{element.is_a()} [{element.GlobalId[:8]}]"` |
| `rule` | str | Check name, e.g. `"Door Width (Accessibility)"` |
| `requirement` | str | What the regulation demands, e.g. `">= 800 mm"` |
| `actual_value` | str | What was found, always include units |
| `passed` | bool \| None | `True` = pass, `False` = fail, `None` = data missing |

## Example

```python
{
    "element_id": "3xF4d2kLnE8fQcR9...",
    "element_type": "IfcDoor",
    "element_name": "Door #42",
    "rule": "Door Width (Accessibility)",
    "requirement": ">= 800 mm",
    "actual_value": "750 mm",
    "passed": False
}
```
