# Frontend Architecture

Modular web app. Each feature (upload, results, 3D viewer, dashboard) is a
**module** — a self-contained folder. Backend operations are **async jobs**.

> **New to web development?** A frontend is the part users see and click on
> (the website). The backend is the server doing the heavy work (running checks).
> They talk to each other through an **API** — a set of URLs the frontend
> calls to send or receive data.

## Structure

```
src/
├── app.js          ← shell: nav bar + router
├── api.js          ← shared API client → CF Worker
├── store.js        ← shared state (Zustand)
├── poller.js       ← polls active jobs, updates store
├── modules/
│   ├── upload/     ← file upload
│   ├── results/    ← results table
│   ├── viewer-3d/  ← IFC 3D viewer
│   └── dashboard/  ← analytics
└── shared/         ← reusable components
```

## Shell + Router

Nav bar at top, content area below. Router swaps modules like tabs.

> **What's a router?** It watches the URL in the browser. When the URL says
> `/results`, it shows the results module. When it says `/dashboard`, it shows
> the dashboard. Each "page" is a module, but it's all one app — no page reloads.

```
┌──────────────────────────────────────┐
│  Nav:  Upload | Results | 3D | Dash  │
├──────────────────────────────────────┤
│   ← active module renders here →     │
└──────────────────────────────────────┘
```

Each module exports `mount(container)`. That's the only contract.

## Async Job Pattern

Backend tasks (IFC checks, AI agents) take 10-60 seconds. Too slow for
a normal request-response. Everything uses **async jobs**.

> **What's async?** Normally when you click a button, the browser waits for
> the server to respond — that's synchronous ("sync"). But if the server needs
> 30 seconds, the browser would freeze. **Async** means: "start the work,
> come back and check later." The server returns a job ID immediately,
> and the frontend keeps checking ("polling") until the job is done.

```
Frontend           Worker (proxy)      HF Space (FastAPI)
────────           ──────────────      ──────────────────
POST /check  ───>  proxy         ───>  start background task
             <───  {jobId}       <───  return {jobId} immediately

poll GET /jobs/id  read D1              ...working...
             <───  {status:"running"}

poll GET /jobs/id  read D1        ───>  POST /jobs/id/complete (callback)
             <───  {status:"done", data:[...]}
```

**Why this pattern?** The CF Worker has a 10ms CPU limit on the free tier —
it physically can't wait 30 seconds. So it just reads/writes to the database
and returns. The heavy work happens on the HF Space.

> **What's a callback?** When the HF Space finishes, it calls the Worker
> back ("hey, job X is done, here are the results"). The Worker writes
> the results to the database. Next time the frontend polls, it gets them.

### Recipe: Adding a New Async Endpoint

Three files. Always the same pattern.

| File | What to add |
|------|-------------|
| **HF Space** `main.py` | `POST /your-thing` → starts `BackgroundTasks`, returns `{jobId}`. When done, POSTs results back to Worker. |
| **Worker** | Proxy route for `POST /api/your-thing`. Job tracking routes (`GET /api/jobs/:id`, `POST /api/jobs/:id/complete`) are shared — built once. |
| **Frontend** `api.js` | `startYourThing(fileUrl)` → returns `{jobId}`. Call `store.trackJob(jobId)` — poller handles the rest. |

> **What's an endpoint?** A specific URL the server listens on. `POST /check`
> is an endpoint. `GET /jobs/123` is another. Think of them as doors into the
> backend — each door does one thing.

## Shared State (Zustand)

**Zustand** is a tiny state library (~1KB). Think of it as a shared whiteboard —
any module can read or write to it. The poller updates it when jobs complete.

> **What's state?** The data your app is currently showing. "Which file is
> selected? Are checks running? What were the results?" That's all state.
> Without a shared store, each module would have its own copy and they'd
> get out of sync. Zustand keeps one source of truth.

**How modules use it:**
- **Upload** sets `currentFile`, starts a job → `trackJob(jobId)`
- **Results** reads `getActiveResults()` → renders a table
- **3D Viewer** reads results → highlights failing elements in red
- **Dashboard** reads results → shows charts and stats

They all see the same data. When a job completes, everything re-renders.

```javascript
// store.js — schematic
{
  currentFile: null,                    // { url, name }
  jobs: {},                             // { [jobId]: { status, data, startedAt } }
  activeJobId: null,
  trackJob(jobId),                      // start tracking
  completeJob(jobId, data),             // poller calls this
  getActiveResults(),                   // results for active job
}
```

**Poller:** every 2s, calls `GET /api/jobs/:id` for running jobs.
When status flips to `"done"`, calls `store.completeJob()`.

**API client** (`api.js`): all modules go through this — never call `fetch()` directly.
Key functions: `uploadFile()`, `startCheck()`, `getJob()`, `getStats()`.

> **What's `fetch()`?** The browser's built-in way to call an API. `api.js`
> wraps it so you don't repeat the base URL and error handling everywhere.

## Module Pattern

Modules render once, then **subscribe** to re-render on state changes:

```javascript
// modules/summary/index.js — schematic
export function mount(container) {
  function render() {
    const results = useStore.getState().getActiveResults()
    container.innerHTML = `${passed} passed, ${failed} failed`
  }
  render()                           // initial
  useStore.subscribe(render)         // re-render on change
}
```

> **What's subscribe?** "Notify me when something changes." Without it, your
> module renders once and never updates. With `subscribe(render)`, every time
> the store changes (new results, job completed), your module re-renders
> automatically.

**Rules:**
- Don't import from other modules
- Read state from `store.js` via `subscribe()`
- Call backend through `api.js`
- One folder, one concern

## Adding a Module (Checklist)

1. Create `src/modules/<name>/index.js` with `mount()` + `subscribe()`
2. Register route in `app.js`
3. If it needs a new backend endpoint → follow async recipe above

## Adding Shared State

Only if multiple modules need it. Otherwise keep local.

1. Add to `store.js`: state field + setter
2. Read from modules via `getState()` + `subscribe()`

## Database (D1)

> **What's D1?** Cloudflare's database service. It runs SQLite (a simple
> database that stores data in tables, like a spreadsheet). The frontend
> never talks to D1 directly — it goes through the Worker API.

```sql
CREATE TABLE users (
  id TEXT PRIMARY KEY,
  name TEXT,
  team TEXT,                     -- e.g. "ifcore-team-a", nullable
  created_at INTEGER
);

CREATE TABLE projects (
  id TEXT PRIMARY KEY,
  user_id TEXT REFERENCES users(id),
  name TEXT,
  file_url TEXT,
  ifc_schema TEXT,               -- e.g. "IFC4", null if unknown
  region TEXT,                   -- e.g. "CH", null if unknown
  building_type TEXT,            -- e.g. "residential", null
  metadata TEXT,                 -- JSON blob, nullable
  created_at INTEGER
);

CREATE TABLE check_results (
  id TEXT PRIMARY KEY,
  project_id TEXT REFERENCES projects(id),
  job_id TEXT,                   -- groups results from one run
  check_name TEXT,               -- e.g. "check_door_width"
  team TEXT,                     -- e.g. "ifcore-team-a"
  status TEXT DEFAULT 'running', -- pass | fail | unknown | error | running
  summary TEXT,                  -- "14 doors: 12 pass, 2 fail"
  has_elements INTEGER DEFAULT 0,
  created_at INTEGER
);

CREATE TABLE element_results (
  id TEXT PRIMARY KEY,
  check_result_id TEXT REFERENCES check_results(id),
  element_id TEXT,               -- IFC GlobalId (nullable)
  element_type TEXT,             -- e.g. "IfcDoor" (nullable)
  element_name TEXT,             -- e.g. "Door #42" (nullable)
  element_name_long TEXT,        -- e.g. "Door #42 (Level 1, Zone A)" (nullable)
  check_status TEXT,             -- pass | fail | warning | blocked | log
  actual_value TEXT,             -- e.g. "750 mm"
  required_value TEXT,           -- e.g. "800 mm"
  comment TEXT,                  -- human-readable explanation (nullable)
  log TEXT                       -- debug/trace info (nullable)
);
```

See [Validation Schema](./validation-schema.md) for how team `list[dict]` output maps to these rows.

> **What's a migration?** A file that changes the database structure
> (adds a table, adds a column). You run it once with `wrangler d1 execute`.
> It's like a recipe for the database — run it, and the new table exists.

**Adding a new table:** migration file → `wrangler d1 execute` → Worker endpoint → `api.js` function → module uses it.

## PRD Review (Wednesday)

> **What's a PRD?** Product Requirements Document — a short plan describing
> what you're building, why, and what "done" looks like. Doesn't need to be
> fancy. Half a page is fine.

Before building, each team writes a PRD for their module. All reviewed together:
- What each team builds
- What async endpoints are needed
- What shared state each module expects
- Whether modules overlap
