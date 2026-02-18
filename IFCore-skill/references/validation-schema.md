# Validation Schema

Two layers: what teams produce, and how the platform stores it.

## Team Output — [TBD: Board Meeting #1]

The exact return format for `check_*` functions will be decided in Board Meeting #1.
Teams will likely return **structured JSON** (list of dicts) that maps directly to the
database schema below — no string parsing needed.

**Probable format** (pending board decision):

```python
def check_door_width(model, min_width_mm=800):
    results = []
    for door in model.by_type("IfcDoor"):
        width_mm = round(door.OverallWidth * 1000) if door.OverallWidth else None
        results.append({
            "element_id":     door.GlobalId,
            "element_type":   "IfcDoor",
            "element_name":   door.Name or f"Door #{door.id()}",
            "status":         "unknown" if width_mm is None
                              else "pass" if width_mm >= min_width_mm
                              else "fail",
            "actual_value":   f"{width_mm} mm" if width_mm else None,
            "required_value": f"{min_width_mm} mm",
        })
    return results
```

Each dict maps directly to one `element_results` row (see below). The orchestrator
writes these to the database with minimal transformation.

**Status values:**
- `"pass"` — element meets the requirement
- `"fail"` — element violates the requirement
- `"unknown"` — data missing, cannot determine

> **Until the board locks this:** the exact fields and naming may change.
> The database schema below is the target — team output should align to it.

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
  "id":              "string",
  "check_result_id": "string",
  "element_id":      "string | null",
  "element_type":    "string | null",
  "element_name":    "string | null",
  "status":          "string (pass | fail | unknown)",
  "actual_value":    "string | null",
  "required_value":  "string | null",
  "raw":             "string | null"
}
```

- `element_id`: IFC GlobalId (if available)
- `raw`: preserved original output (for debugging / fallback display)

## How It Fits Together

```
Team function returns:
  [
    {"element_id": "2O2Fr$t4X7Z", "element_type": "IfcDoor", "element_name": "Door #42",
     "status": "pass", "actual_value": "850 mm", "required_value": "800 mm"},
    {"element_id": "1B3Rs$u5Y8A", "element_type": "IfcDoor", "element_name": "Door #17",
     "status": "fail", "actual_value": "750 mm", "required_value": "800 mm"}
  ]

Orchestrator creates:

  check_results row:
    check_name  = "check_door_width"
    team        = "ifcore-team-a"
    status      = "fail"              ← any fail → whole check fails
    summary     = "2 doors: 1 pass, 1 fail"
    has_elements = 1

  element_results rows:  (one per dict in the list)
    { element_name: "Door #42", status: "pass", actual_value: "850 mm", ... }
    { element_name: "Door #17", status: "fail", actual_value: "750 mm", ... }
```
