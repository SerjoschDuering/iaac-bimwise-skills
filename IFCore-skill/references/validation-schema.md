# Validation Schema

Check functions **exposed as Gradio API endpoints** (called by the orchestrator) MUST return this format.
Internal helper functions are not required to follow it.

```python
{
    "element_id":    str,       # IFC GlobalId
    "element_type":  str,       # e.g. "IfcDoor", "IfcSpace"
    "element_name":  str,       # human-readable; fallback: f"{el.is_a()} [{el.GlobalId[:8]}]"
    "rule":          str,       # e.g. "Door Width (Accessibility)"
    "requirement":   str,       # e.g. ">= 800 mm"
    "actual_value":  str,       # what was found — always include units
    "passed":        bool|None  # True = pass, False = fail, None = data missing
}
```

## Example

```python
{
    "element_id":   "3xF4d2kLnE8fQcR9",
    "element_type": "IfcDoor",
    "element_name": "Door #42",
    "rule":         "Door Width (Accessibility)",
    "requirement":  ">= 800 mm",
    "actual_value": "750 mm",
    "passed":       False
}
```

**Platform API contract extensions:** [TBD after Board Meeting #2]
