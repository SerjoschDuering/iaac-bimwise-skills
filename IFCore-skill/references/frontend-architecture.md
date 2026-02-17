# Frontend Architecture

Modular web app. Each feature (upload, results, 3D viewer, dashboard) is a
**module** — a self-contained folder. Backend operations are **async jobs**.

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
request-response. Everything uses **async jobs**:

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

**Why?** Worker has 10ms CPU limit — can't wait. It just reads/writes D1.

### Recipe: Adding a New Async Endpoint

Three files. Always the same.

| File | What to add |
|------|-------------|
| **HF Space** `main.py` | `POST /your-thing` → starts `BackgroundTasks`, returns `{jobId}`. When done, POSTs results back to Worker. |
| **Worker** | Proxy route for `POST /api/your-thing`. Job tracking routes (`GET /api/jobs/:id`, `POST /api/jobs/:id/complete`) are shared — built once. |
| **Frontend** `api.js` | `startYourThing(fileUrl)` → returns `{jobId}`. Call `store.trackJob(jobId)` — poller handles the rest. |

## Shared State (Zustand)

**Zustand** is a tiny state library (~1KB). Think of it as a shared whiteboard —
any module can read or write to it. The poller updates it when jobs complete.

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

All persistent data in D1 (SQLite). Frontend never talks to D1 directly.

```sql
-- Core table (built once)
CREATE TABLE jobs (
  job_id TEXT PRIMARY KEY,
  status TEXT DEFAULT 'running',   -- running | done | error
  file_url TEXT,
  data TEXT,                       -- JSON results (null while running)
  created_at INTEGER
);
```

**Adding a new table:** migration file → `wrangler d1 execute` → Worker endpoint → `api.js` function → module uses it.

## PRD Review (Wednesday)

Before building, each team writes a PRD. All reviewed together:
- What each team builds
- What async endpoints are needed
- What shared state each module expects
- Whether modules overlap
