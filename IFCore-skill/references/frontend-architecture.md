# Frontend Architecture

The IFCore frontend is a modular web app on Cloudflare Pages. Each feature
(upload, results table, 3D viewer, dashboard) is a **module** — a self-contained
folder that plugs into the shell. Students add modules via PRDs and branches.

## How the App is Organized

```
src/
├── app.js             ← the shell: navigation + router
├── api.js             ← shared API client (talks to CF Worker)
├── store.js           ← shared state (Zustand) — the "brain"
├── modules/
│   ├── upload/        ← file upload module
│   │   └── index.js
│   ├── results/       ← results table module
│   │   └── index.js
│   ├── viewer-3d/     ← 3D IFC viewer module
│   │   └── index.js
│   └── dashboard/     ← analytics dashboard module
│       └── index.js
└── shared/            ← reusable pieces (buttons, cards, layout)
```

## The Shell

The shell is the frame around everything — a nav bar at the top, a content area
below. When you click "Results" in the nav, the router swaps in the results module.
Think of it like tabs in an app.

```
┌──────────────────────────────────────┐
│  Nav:  Upload | Results | 3D | Dash  │
├──────────────────────────────────────┤
│                                      │
│   ← active module renders here →     │
│                                      │
└──────────────────────────────────────┘
```

Each module exports a `mount(container)` function. The router calls it when
the user navigates to that module. That's the only contract between shell and module.

## Shared State (Zustand Store)

Modules don't talk to each other directly. They read from and write to a
**shared store** — a central place that holds the current app state.

**Zustand** is a tiny state management library (~1KB). Think of it as a shared
whiteboard that any module can read or write to.

The store holds:

```javascript
// store.js
import { create } from 'zustand'

export const useStore = create((set) => ({
  // Current IFC file being worked on
  currentFile: null,       // { url, name, size }
  setCurrentFile: (file) => set({ currentFile: file }),

  // Current job (a check run)
  currentJob: null,        // { jobId, status, createdAt }
  setCurrentJob: (job) => set({ currentJob: job }),

  // Check results for the current job
  results: [],             // list of check results from all teams
  setResults: (results) => set({ results }),

  // Loading / error states
  loading: false,
  setLoading: (v) => set({ loading: v }),
  error: null,
  setError: (e) => set({ error: e }),
}))
```

**How modules use it:**
- Upload module: sets `currentFile` after upload, triggers a check, sets `currentJob`
- Results module: reads `results` and displays a table
- 3D Viewer: reads `results` and highlights failing elements in the model
- Dashboard: reads `results` and shows charts/stats

They all see the same data. When Upload sets new results, the table, viewer,
and dashboard all update automatically.

## API Client

All modules call the backend through one shared API client. This keeps
the API URL and error handling in one place.

```javascript
// api.js
const API_BASE = 'https://api.ifcore.dev'  // CF Worker

export async function uploadFile(file) { ... }
export async function runCheck(fileUrl) { ... }
export async function getResults(jobId) { ... }
export async function getStats() { ... }
```

Modules never call `fetch()` directly — they use `api.js`.

## How to Add a New Module

1. Create a folder: `src/modules/your-feature/`
2. Create `index.js` that exports `mount(container)`
3. Register the route in `app.js`
4. Read from `useStore` for shared data
5. Call `api.js` for backend requests

Example — a simple "summary" module:

```javascript
// src/modules/summary/index.js
import { useStore } from '../../store.js'

export function mount(container) {
  const { results } = useStore.getState()
  const passed = results.filter(r => r.message.startsWith('[PASS]')).length
  const failed = results.filter(r => r.message.startsWith('[FAIL]')).length

  container.innerHTML = `
    <h2>Summary</h2>
    <p>${passed} passed, ${failed} failed</p>
  `
}
```

Then add the route in `app.js`:
```javascript
import { mount as mountSummary } from './modules/summary/index.js'
// ... in the router:
'/summary': mountSummary,
```

That's it. The module is isolated — it doesn't import from other modules, only
from `store.js`, `api.js`, and `shared/`.

## How to Add New State

If your module needs new shared data (e.g., a list of saved reports):

1. Add the state + setter to `store.js`:
```javascript
savedReports: [],
addReport: (report) => set((state) => ({
  savedReports: [...state.savedReports, report]
})),
```

2. Use it in your module:
```javascript
const { savedReports, addReport } = useStore.getState()
```

Rule: if only YOUR module needs the data, keep it local (don't add to store).
If multiple modules need it, add to store.

## Database (D1)

All persistent data lives in Cloudflare D1 (SQLite at the edge). The frontend
never talks to D1 directly — it goes through the CF Worker API.

### Current Tables

```sql
-- Check results (one row per check run)
CREATE TABLE results (
  job_id TEXT PRIMARY KEY,
  file_url TEXT,
  data TEXT,          -- JSON array of check results
  created_at INTEGER
);
```

### Adding a New Table

If your feature needs to store new data (e.g., saved reports, annotations):

1. **Define the table** in a migration file
2. **Add a Worker endpoint** that reads/writes it
3. **Add an API function** in `api.js`
4. **Use it from your module**

Example — adding a "reports" table:

```sql
-- migrations/002_reports.sql
CREATE TABLE reports (
  id TEXT PRIMARY KEY,
  job_id TEXT REFERENCES results(job_id),
  title TEXT,
  notes TEXT,
  created_at INTEGER
);
```

Run: `wrangler d1 execute ifcore-results --file migrations/002_reports.sql`

Then add the Worker endpoint and `api.js` function to match. The pattern is
always: **D1 table → Worker endpoint → api.js function → module uses it.**

## Module Rules

- Modules don't import from other modules
- Modules read shared data from `store.js`
- Modules call the backend through `api.js`
- Modules can use anything from `shared/` (buttons, cards, layout helpers)
- Keep modules small — one folder, one concern
- If a module grows past 300 lines, split into sub-files within the folder

## PRD Review Process

Before building, each team writes a short PRD for their module. All PRDs are
reviewed together in one session so everyone sees:
- What each team is building
- What API endpoints are needed (surfaces gaps early)
- What shared state each module expects
- Whether any modules overlap

This happens at Board Meeting #2 (Wednesday). See the week plan for timing.
