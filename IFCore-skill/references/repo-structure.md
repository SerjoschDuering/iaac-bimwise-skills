# Repository Structure

## Team repos — `Mastodonte`, `lux-ai`, `team-d`, `structures`, `team-e`
One repo per team. Students only ever touch their own.
```
<team-repo>/
├── tools/
│   ├── checker_doors.py       # check_door_width, check_door_clearance
│   ├── checker_fire_safety.py # check_fire_rating, check_exit_count
│   └── checker_rooms.py       # check_room_area, check_ceiling_height
├── requirements.txt
├── AGENTS.md
└── README.md
```

## Platform monorepo — `ifcore-platform`
**One repo. Two folders. Two deployments.**
```
ifcore-platform/
│
├── backend/                    → deploys to HuggingFace Space (Docker)
│   ├── README.md                   ← HF frontmatter (sdk: docker, app_port: 7860)
│   ├── Dockerfile                  ← python:3.11-slim, uvicorn --workers 1
│   ├── requirements.txt
│   ├── main.py                     ← FastAPI: /health, /check, /jobs/:id, /chat
│   ├── orchestrator.py             ← discovers check_* from teams/*/tools/
│   ├── deploy.sh                   ← submodule update → rsync → force-push to HF
│   └── teams/                      ← git submodules, populated by deploy.sh
│       ├── demo/tools/checker_demo.py
│       ├── Mastodonte/tools/       ← habitability checks
│       ├── lux-ai/tools/           ← energy/solar checks
│       ├── team-d/tools/           ← fire compliance checks
│       ├── structures/tools/       ← structural checks
│       └── team-e/tools/           ← lighting/facade checks
│
├── frontend/                   → deploys to Cloudflare Workers + Static Assets
│   ├── index.html
│   ├── package.json
│   ├── vite.config.ts              ← Cloudflare + TanStack Router + React plugins
│   ├── wrangler.jsonc              ← D1 + R2 bindings, SPA routing
│   ├── tsconfig.json
│   │
│   ├── worker/                     ← Hono API gateway (Cloudflare Worker)
│   │   ├── index.ts                    entry: CORS + route mounting
│   │   ├── types.ts                    Bindings type (DB, STORAGE, etc.)
│   │   ├── routes/
│   │   │   ├── health.ts
│   │   │   ├── projects.ts            CRUD projects
│   │   │   ├── checks.ts              proxy to HF + job tracking + lazy-polling
│   │   │   ├── upload.ts              multipart → R2
│   │   │   ├── files.ts               serve R2 objects
│   │   │   └── chat.ts                proxy to HF /chat
│   │   └── lib/
│   │       ├── db.ts                   D1 query helpers
│   │       ├── auth.ts                 Better Auth setup (D1-backed)
│   │       └── schema.ts              Drizzle schema for auth tables
│   │
│   ├── migrations/
│   │   └── 0001_init.sql              5 tables: users, projects, jobs, check_results, element_results
│   │
│   └── src/                        ← React 19 + TypeScript SPA
│       ├── main.tsx                    React entry + TanStack Router
│       ├── routeTree.gen.ts            auto-generated
│       │
│       ├── routes/                     file-based routes (auto code-split)
│       │   ├── __root.tsx                  layout: WorkspaceToolbar + 3-column grid
│       │   ├── index.tsx                   / → redirect to /projects
│       │   ├── login.tsx                   /login page
│       │   ├── profile.tsx                 /profile page
│       │   ├── projects.tsx                /projects layout wrapper
│       │   ├── projects.index.tsx          project list + upload form
│       │   ├── projects.$id.tsx            project detail + checks
│       │   ├── dashboard.tsx               /dashboard
│       │   ├── checks.tsx                  /checks results view
│       │   ├── report.tsx                  /report team report
│       │   └── chat.tsx                    /chat standalone chat
│       │
│       ├── features/                   feature modules (colocated)
│       │   ├── auth/
│       │   │   ├── LoginPage.tsx
│       │   │   ├── ProfilePage.tsx
│       │   │   └── UserSettingsModal.tsx
│       │   ├── upload/
│       │   │   ├── UploadForm.tsx
│       │   │   └── useUpload.ts
│       │   ├── checks/
│       │   │   ├── CheckRunner.tsx
│       │   │   └── ResultsTable.tsx
│       │   ├── viewer/
│       │   │   ├── BIMViewer.tsx            ThatOpen Components IFC viewer
│       │   │   ├── ViewerPanel.tsx          wrapper with controls
│       │   │   ├── useViewer.ts             syncs check results → colorMap (5 statuses)
│       │   │   ├── viewerActions.ts         highlight/hide/isolate GUID helpers
│       │   │   ├── viewerDiagnostics.ts     phase tracking + error classification
│       │   │   └── ElementTooltip.tsx       shows check results on element click
│       │   ├── categories/
│       │   │   ├── CategoryCards.tsx         category status cards
│       │   │   ├── CategorySidebar.tsx       left sidebar wrapper
│       │   │   └── useCategoryColors.ts     category → highlightColorMap
│       │   ├── dashboard/
│       │   │   ├── TechnicalDashboard.tsx   pass/fail KPIs + charts
│       │   │   ├── StatusChart.tsx          donut + category bar charts (Recharts)
│       │   │   ├── KpiGauge.tsx             animated SVG gauge
│       │   │   └── ElementTable.tsx         element-level results table
│       │   ├── report/
│       │   │   └── TeamReportPanel.tsx      hierarchical team→check→element report
│       │   └── chat/
│       │       └── ChatPanel.tsx            AI compliance assistant
│       │
│       ├── stores/
│       │   ├── store.ts                combined Zustand store (5 slices)
│       │   ├── types.ts                AppStore type
│       │   └── slices/
│       │       ├── projectsSlice.ts
│       │       ├── checksSlice.ts
│       │       ├── jobsSlice.ts
│       │       ├── viewerSlice.ts      ifcUrl, colorMap, highlightColorMap, selectedIds, hiddenIds, isReady
│       │       └── filterSlice.ts      selectedCategory, selectedCheckId
│       │
│       ├── lib/
│       │   ├── api.ts                  typed fetch wrapper for /api/*
│       │   ├── poller.ts               polls running jobs every 2s (batched state updates)
│       │   ├── types.ts                shared TS types (Project, Job, CheckResult, ElementResult)
│       │   ├── constants.ts            STATUS_COLORS, CATEGORIES, getCategory, statusToHex
│       │   ├── auth-client.ts          Better Auth React client
│       │   └── web-ifc-shim.ts         re-exports globalThis.WebIFC for bundler bypass
│       │
│       ├── components/                 shared UI
│       │   └── Navbar.tsx
│       │
│       └── styles/
│           └── globals.css             CSS variables, glass panels, animations
│
└── feature-plans/              ← PRD documents (Thursday)
    └── TEMPLATE.md
```
