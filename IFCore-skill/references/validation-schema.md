# Validation Schema

Two formats exist: what teams produce, and what the platform serves.

## Team Output (what your check functions return)

Teams return `list[str]`. Each string is one element checked, prefixed with status:

```
[PASS] Door #42: 850 mm (min 800 mm)
[FAIL] Door #17: 750 mm (min 800 mm)
[???]  Door #99: width unknown
```

Prefix meanings:
- `[PASS]` — element meets the requirement
- `[FAIL]` — element violates the requirement
- `[???]` — data missing, cannot determine

**This is the only format teams need to produce.** The platform handles the rest.

## Platform Schema (what the orchestrator converts to)

The platform parses team strings and serves structured JSON to the frontend:

```python
{
    "check_id":      str,       # auto-generated: team_name + function_name + index
    "team":          str,       # which team produced this result
    "element_id":    str,       # IFC GlobalId (extracted if available)
    "element_type":  str,       # e.g. "IfcDoor", "IfcSpace"
    "element_name":  str,       # human-readable name from the string
    "rule":          str,       # from the check function name (check_door_width → "door width")
    "requirement":   str,       # extracted from string if pattern matches
    "actual_value":  str,       # extracted from string if pattern matches
    "passed":        bool|None, # True=[PASS], False=[FAIL], None=[???]
    "raw":           str        # the original string, preserved as-is
}
```

Teams don't produce this. The platform orchestrator does the conversion.
If the string doesn't match expected patterns, `raw` is always available as fallback.

**Platform API contract extensions:** [TBD after Board Meeting #2]
