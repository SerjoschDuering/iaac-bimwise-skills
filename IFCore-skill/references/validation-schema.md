# Validation Schema

Two layers: what teams produce, and how the platform stores it.

## Team Output (locked — Board Meeting #1)

Each `check_*` function returns `list[dict]`. Each dict is one element checked,
mapping directly to one `element_results` row in the database.

```python
def check_door_width(model, min_width_mm=800):
    results = []
    for door in model.by_type("IfcDoor"):
        width_mm = round(door.OverallWidth * 1000) if door.OverallWidth else None
        results.append({
            "element_id":       door.GlobalId,
            "element_type":     "IfcDoor",
            "element_name":     door.Name or f"Door #{door.id()}",
            "element_name_long": f"{door.Name} (Level 1, Zone A)",
            "check_status":     "blocked" if width_mm is None
                                else "pass" if width_mm >= min_width_mm
                                else "fail",
            "actual_value":     f"{width_mm} mm" if width_mm else None,
            "required_value":   f"{min_width_mm} mm",
            "comment":          None if width_mm and width_mm >= min_width_mm
                                else f"Door is {min_width_mm - width_mm} mm too narrow"
                                if width_mm else "Width property missing",
            "log":              None,
        })
    return results
```

**Required dict fields** (teams produce these — `id` and `check_result_id` are added by the orchestrator):

| Field | Type | Description |
|-------|------|-------------|
| `element_id` | string \| null | IFC GlobalId |
| `element_type` | string \| null | e.g. `"IfcDoor"`, `"IfcWall"` |
| `element_name` | string \| null | Short name, e.g. `"Door #42"` |
| `element_name_long` | string \| null | Detailed name with context, e.g. `"Door #42 (Level 1, Zone A)"` |
| `check_status` | string | **`pass`** \| **`fail`** \| **`warning`** \| **`blocked`** \| **`log`** |
| `actual_value` | string \| null | What was found, e.g. `"750 mm"` |
| `required_value` | string \| null | What the regulation requires, e.g. `"800 mm"` |
| `comment` | string \| null | Human-readable explanation (why it failed, what's wrong) |
| `log` | string \| null | Debug/trace info (optional, for troubleshooting) |

**`check_status` values:**
- `"pass"` — element meets the requirement
- `"fail"` — element violates the requirement
- `"warning"` — element is borderline or needs manual review
- `"blocked"` — data missing, check cannot run (e.g. property not found)
- `"log"` — informational output, not a pass/fail judgment

## Platform Database Schema (D1)

Four tables. The frontend reads from these via the CF Worker API.

```
┌─────────┐       ┌─────────────┐       ┌────────────────┐       ┌──────────────────┐
│  users  │ 1───* │  projects   │ 1───* │  check_results │ 1───* │  element_results │
└─────────┘       └─────────────┘       └────────────────┘       └──────────────────┘
```

### `users` — one row per person

```json
{
  "id":         "string",
  "name":       "string",
  "team":       "string | null",
  "created_at": "integer"
}
```

### `projects` — one row per uploaded IFC file

```json
{
  "id":            "string",
  "user_id":       "string",
  "name":          "string",
  "file_url":      "string",
  "ifc_schema":    "string | null",
  "region":        "string | null",
  "building_type": "string | null",
  "metadata":      "string | null (JSON)",
  "created_at":    "integer"
}
```

### `check_results` — one row per `check_*` function run

```json
{
  "id":            "string",
  "project_id":    "string",
  "job_id":        "string",
  "check_name":    "string",
  "team":          "string",
  "status":        "string (running | pass | fail | unknown | error)",
  "summary":       "string",
  "has_elements":  "integer (0 | 1)",
  "created_at":    "integer"
}
```

- `check_name`: the function name, e.g. `check_door_width`
- `team`: derived from the repo folder name, e.g. `ifcore-team-a`
- `status`: `running` while job is in progress; then aggregate — `pass` if all elements pass, `fail` if any fail, `error` if the function threw
- `summary`: human-readable, e.g. "14 doors checked: 12 pass, 2 fail"
- `has_elements`: `1` if the check produced element-level results, `0` otherwise

### `element_results` — one row per element checked

```json
{
  "id":               "string",
  "check_result_id":  "string",
  "element_id":       "string | null",
  "element_type":     "string | null",
  "element_name":     "string | null",
  "element_name_long":"string | null",
  "check_status":     "string (pass | fail | warning | blocked | log)",
  "actual_value":     "string | null",
  "required_value":   "string | null",
  "comment":          "string | null",
  "log":              "string | null"
}
```

- `id`, `check_result_id`: added by the orchestrator (teams don't produce these)
- `element_id`: IFC GlobalId (if available)
- `check_status`: matches what the team function returns — not aggregated
- `comment`: human-readable explanation of the result
- `log`: debug/trace info for troubleshooting

## How It Fits Together

```
Team function returns:
  [
    {"element_id": "2O2Fr$t4X7Z", "element_type": "IfcDoor",
     "element_name": "Door #42", "element_name_long": "Door #42 (Level 1)",
     "check_status": "pass", "actual_value": "850 mm", "required_value": "800 mm",
     "comment": null, "log": null},
    {"element_id": "1B3Rs$u5Y8A", "element_type": "IfcDoor",
     "element_name": "Door #17", "element_name_long": "Door #17 (Level 2)",
     "check_status": "fail", "actual_value": "750 mm", "required_value": "800 mm",
     "comment": "Door is 50 mm too narrow", "log": null}
  ]

Orchestrator creates:

  check_results row:
    check_name   = "check_door_width"
    team         = "ifcore-team-a"
    status       = "fail"              ← any fail → whole check fails
    summary      = "2 doors: 1 pass, 1 fail"
    has_elements = 1

  element_results rows:  (one per dict, id + check_result_id added)
    { element_name: "Door #42", check_status: "pass", actual_value: "850 mm", ... }
    { element_name: "Door #17", check_status: "fail", actual_value: "750 mm", comment: "Door is 50 mm too narrow", ... }
```
