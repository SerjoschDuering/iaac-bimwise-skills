# Repository Structure

## Team repos — `ifcore-team-a` … `ifcore-team-e`
One repo per team. Students only ever touch their own.
```
ifcore-team-a/
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
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── main.py                     ← FastAPI: /health, POST /check, POST /convert
│   ├── orchestrator.py             ← discovers check_* from teams/*/tools/
│   ├── ifc_converter.py            ← IFC→GLB conversion (trimesh + ifcopenshell)
│   ├── deploy.sh                   ← pull submodules → flatten → push to HF
│   └── teams/                      ← gitignored except demo/, populated by deploy.sh
│       └── demo/tools/checker_demo.py
│
├── frontend/                   → deploys to Cloudflare Workers + Static Assets
│   ├── index.html
│   ├── package.json
│   ├── vite.config.ts              ← Cloudflare + TanStack Router + React plugins
│   ├── wrangler.jsonc              ← D1 + R2 bindings, SPA routing
│   ├── tsconfig.json
│   ├── tsconfig.worker.json
│   │
│   ├── worker/                     ← Hono API gateway (Cloudflare Worker)
│   │   ├── index.ts                    entry: CORS + route mounting
│   │   ├── types.ts                    Bindings type (DB, STORAGE, etc.)
│   │   ├── routes/
│   │   │   ├── health.ts
│   │   │   ├── projects.ts            CRUD projects
│   │   │   ├── checks.ts              proxy to HF + job tracking + callback
│   │   │   ├── upload.ts              multipart → R2
│   │   │   └── files.ts               serve R2 objects
│   │   └── lib/
│   │       └── db.ts                   D1 query helpers
│   │
│   ├── migrations/
│   │   └── 0001_init.sql              5 tables: users, projects, jobs, check_results, element_results
│   │
│   └── src/                        ← React 19 + TypeScript SPA
│       ├── main.tsx                    React entry + TanStack Router
│       ├── routeTree.gen.ts            auto-generated
│       ├── routes/                     file-based routes (auto code-split)
│       │   ├── __root.tsx                  layout: Navbar + <Outlet>
│       │   ├── index.tsx                   / → redirect to /projects
│       │   ├── projects.tsx                /projects layout
│       │   ├── projects.index.tsx          project list + upload
│       │   ├── projects.$id.tsx            project detail + checks
│       │   ├── checks.tsx                  check results table
│       │   └── viewer.tsx                  3D viewer (lazy R3F)
│       ├── features/                   feature modules (colocated)
│       │   ├── upload/
│       │   ├── checks/
│       │   └── viewer/
│       ├── stores/                     Zustand (slices pattern)
│       │   ├── store.ts
│       │   └── slices/
│       ├── lib/                        api.ts, poller.ts, types.ts
│       ├── components/                 Navbar, StatusBadge, LoadingSpinner
│       └── styles/globals.css
│
└── feature-plans/              ← PRD documents (Thursday)
    └── TEMPLATE.md
```
