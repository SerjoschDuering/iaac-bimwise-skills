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
ifcore-platform/           ← ONE git repo
│
├── backend/               → deploys to HuggingFace Space
│   ├── README.md              ← HF frontmatter (sdk: docker, app_port: 7860)
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── main.py
│   ├── orchestrator.py
│   ├── deploy.sh
│   └── teams/                 ← gitignored, populated by deploy.sh
│       ├── ifcore-team-a/tools/checker_*.py
│       ├── ifcore-team-b/tools/checker_*.py
│       └── ...                ← one folder per team, flattened from submodules
│
└── frontend/              → deploys to Cloudflare (Pages + Worker)
    ├── public/
    │   └── index.html
    ├── src/                   ← static app (CF Pages)
    │   ├── app.js
    │   ├── api.js
    │   ├── store.js
    │   ├── poller.js
    │   └── modules/
    │       ├── upload/index.js
    │       ├── results/index.js
    │       ├── viewer-3d/index.js
    │       └── dashboard/index.js
    ├── functions/             ← API gateway (CF Pages Functions = Worker)
    │   └── api/
    │       └── [[route]].js
    ├── migrations/
    │   └── 0001_create_jobs.sql
    ├── package.json
    └── wrangler.toml          ← D1 + R2 bindings, custom domain
```
