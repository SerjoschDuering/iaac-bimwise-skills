# Development Patterns

## How to Work With the User

You're helping someone who may be new to coding, AI, or both. Before doing anything:

- **Ask what they want to build.** Don't assume — let them explain in their own words.
- **Suggest an approach and explain why.** Offer to explain any term they might not know.
- **Break things into small steps.** One thing at a time. Check in after each step.
- **Use plain language.** Say "a function that checks door widths" not "a callable that
  validates dimensional properties."

## When Someone Wants to Add a New Check

1. **Understand what they want.** Ask: "What rule should this check enforce?"
   Get a concrete example — "doors must be at least 900mm wide" is better than "door check."
2. **Talk through the approach.** Explain what the check function will do, what IFC data it needs.
3. **Suggest a plan.** Offer a short plan (see template below). If they prefer to skip it, go to code.
4. **Build it together.** Write the check function, test it, walk through the result.
5. **Review when done.** Suggest a quick review to catch things you both might have missed.

## Feature Plans (Optional but Helpful)

```
feature-plans/
└── F1-door-width-check/
    ├── plan.md          # what we're building and why
    └── learnings.md     # what went wrong, what to do differently
```

## Plan Template

```markdown
# F<N>: <Feature Name>

## What Are We Building?
<!-- Goal in plain language. What problem does this solve? -->

## What Does "Done" Look Like?
- [ ] ...

## How It Works
<!-- Brief approach. What IFC data? What does the check function do? -->
```

## Adding a Frontend Feature

For platform features (not check functions), follow the React module pattern:

1. Create `src/features/<name>/` with component(s) + hook(s)
2. Create `src/routes/<route>.tsx` using `createFileRoute`
3. If it needs new state → add a slice to `stores/slices/` + register in `store.ts` + `types.ts`
4. If it needs a new API call → add to `lib/api.ts`
5. If it needs a new backend endpoint → follow the async job recipe in frontend-architecture.md

**Rules:** Don't import from other feature modules. Read/write state through
`useStore`. Call backend through `lib/api.ts`. One folder, one concern.

**Auth pattern:** If the route accesses private data, the Worker route must call
`getSessionUser()` and `canAccessProject()` from `worker/lib/db.ts`.
Shared projects (`user_id=null`) are accessible to everyone; private projects only to the owner.
Never trust client-supplied `file_url` — always look it up from D1 by `project_id`.

**Shared constants:** Team categories, status colors, and utility functions live in
`lib/constants.ts` (single source of truth). Never hardcode team names or status colors.

## Deployment

### Backend (HF Space)
```bash
cd backend/
bash deploy.sh     # pulls submodules, rsync to temp dir, force-push to HF
```
Requires `HF_TOKEN` env var (or HF CLI auth). Space: `serJD/ifcore-platform`.

### Frontend (Cloudflare)
```bash
cd frontend/
npm run deploy     # = npm run build && wrangler deploy
```
**ALWAYS use `npm run deploy`** — never `npx wrangler deploy` alone. The Worker is compiled
by Vite via `@cloudflare/vite-plugin`, not wrangler's esbuild. Running wrangler alone deploys
the last cached build and ignores code changes.

### D1 Migrations
```bash
cd frontend/
npm run db:migrate              # local dev
npx wrangler d1 execute ifcore-db --remote --file=migrations/0001_init.sql  # production
```

## Local Dev: Common Failures & How to Fix

Before debugging anything, check these first:

### "Can't login / create account"
1. **Wrong port** — If Vite starts on a port other than 5173 (zombie processes), auth rejects requests.
   Fix: `lsof -i :5173` → `kill <PID>` → restart.
2. **Missing drizzle schema** — Better Auth + Drizzle needs `worker/lib/schema.ts` with tables named
   `user` (singular), `session`, `account`, `verification`. Without it: `"model 'user' not found"`.
3. **Date vs integer** — D1 rejects JS Date objects. Schema must use `integer("col", { mode: "timestamp" })`.
4. **Missing DB migrations** — Run `npm run db:migrate` in `frontend/` after every schema change.
5. **bcrypt CPU** — `better-auth` defaults to bcrypt which is CPU-heavy on Workers. If login is slow,
   check whether the auth config uses a lighter hash.

### "Chat doesn't work"
1. **Locally:** `GEMINI_API_KEY` must be set in the terminal running the backend:
   `export GEMINI_API_KEY=your-key`
2. **Production:** Set as a HF Space secret (already done for `serJD/ifcore-platform`).
3. The chat goes: browser → CF Worker `POST /api/chat` → HF `POST /chat` (PydanticAI + Gemini).
   Requires `worker/routes/chat.ts` route.

### "Upload / checks fail"
1. Check both servers are running: backend on `:7860`, frontend on `:5173`
2. Check `.dev.vars` has `HF_SPACE_URL=http://localhost:7860`
3. Check R2 is working locally: `.wrangler/state/v3/r2/` should exist after first upload

### "HF Space seems stuck"
1. **Cold start** — After 48h inactivity, Space sleeps. First request takes 10-60s.
2. **In-memory jobs** — `_jobs` dict resets on Space restart. Stuck "running" jobs in D1 must be re-submitted.
3. **DNS** — HF cannot resolve `*.workers.dev`. Never send Workers URLs to HF as callbacks.

### "403 Forbidden on checks or projects"
1. **Not logged in** — most routes require auth. Check the session cookie exists.
2. **Wrong project owner** — private projects (`user_id != null`) are only accessible to the owner.
   Shared demo projects (`user_id = null`) are accessible to everyone.
3. **Stale session** — try logging out and back in.

### General debugging approach
1. **Check the API directly** with curl before blaming the frontend
2. **Read the terminal output** — wrangler dev logs all errors
3. **Check DB schema** — `npx wrangler d1 execute ifcore-db --local --command "PRAGMA table_info(tablename);"`
4. **Kill and restart** — the Worker caches module-level singletons; code changes sometimes need a full restart

## After You Finish

Suggest a review:

> "That check is working. Want me to do a quick review to catch anything we missed?
> I can do it right now, or you can start a fresh chat and paste this:"

```
I just finished implementing [feature name].
Codebase is at [path]. Key files changed: [list].
Please review against the plan at feature-plans/F<N>-<name>/plan.md
and check for IFCore contract compliance.
```
