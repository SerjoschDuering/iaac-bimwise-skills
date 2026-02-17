# Tutorial: Build a To-Do App (End-to-End)

Step-by-step from zero to deployed fullstack app with database.
Follow every step in order — nothing is skipped.

---

## Step 1: Create the Project

```bash
mkdir todo-app && cd todo-app
mkdir -p api/src/domains/items api/src/db api/migrations
cd api
npm init -y
npm install hono drizzle-orm drizzle-zod @hono/zod-validator zod
npm install -D wrangler @cloudflare/workers-types drizzle-kit typescript
```

## Step 2: Create tsconfig.json

Create `api/tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "types": ["@cloudflare/workers-types"],
    "jsx": "react-jsx",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src"]
}
```

Without this file, TypeScript won't recognize `D1Database` and other Cloudflare types.

## Step 3: Create wrangler.toml

```toml
# api/wrangler.toml
name = "todo-api"
main = "src/index.ts"
compatibility_date = "2025-01-01"
compatibility_flags = ["nodejs_compat"]

[[d1_databases]]
binding = "DB"
database_name = "todo-db"
database_id = "LOCAL_PLACEHOLDER"
# ↑ Replace after running: wrangler d1 create todo-db
```

## Step 4: Define the Schema

```typescript
// api/src/domains/items/items.schema.ts
import { sqliteTable, integer, text } from 'drizzle-orm/sqlite-core'
import { createInsertSchema, createSelectSchema } from 'drizzle-zod'
import { z } from 'zod'

export const items = sqliteTable('items', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  title: text('title').notNull(),
  completed: integer('completed', { mode: 'boolean' }).notNull().default(false),
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
})

export const insertItemSchema = createInsertSchema(items, {
  title: z.string().min(1, 'Title is required').max(200),
})
export const selectItemSchema = createSelectSchema(items)
export type Item = z.infer<typeof selectItemSchema>
export type NewItem = z.infer<typeof insertItemSchema>
```

## Step 5: Create DB Factory + Drizzle Config

```typescript
// api/src/db/index.ts
import { drizzle } from 'drizzle-orm/d1'
import * as schema from './schema'
export const createDb = (d1: D1Database) => drizzle(d1, { schema })
```

```typescript
// api/src/db/schema.ts — barrel export all domain schemas
export * from '../domains/items/items.schema'
```

```typescript
// api/drizzle.config.ts
import { defineConfig } from 'drizzle-kit'
export default defineConfig({
  schema: './src/domains/**/*.schema.ts',
  out: './migrations',
  dialect: 'sqlite',
  driver: 'd1-http',
})
```

## Step 6: Create CRUD Routes

```typescript
// api/src/domains/items/items.routes.ts
import { Hono } from 'hono'
import { zValidator } from '@hono/zod-validator'
import { eq } from 'drizzle-orm'
import { createDb } from '../../db'
import { items, insertItemSchema } from './items.schema'

type Bindings = { DB: D1Database }
const app = new Hono<{ Bindings: Bindings }>()

app.get('/', async (c) => {
  const db = createDb(c.env.DB)
  const result = await db.select().from(items).all()
  return c.json({ data: result })
})

app.post('/', zValidator('json', insertItemSchema), async (c) => {
  const data = c.req.valid('json')
  const db = createDb(c.env.DB)
  const result = await db.insert(items).values(data).returning()
  return c.json({ data: result[0] }, 201)
})

app.put('/:id', async (c) => {
  const id = Number(c.req.param('id'))
  const db = createDb(c.env.DB)
  const existing = await db.select().from(items).where(eq(items.id, id)).get()
  if (!existing) return c.json({ error: 'Not found' }, 404)
  const result = await db.update(items).set({ completed: !existing.completed })
    .where(eq(items.id, id)).returning()
  return c.json({ data: result[0] })
})

app.delete('/:id', async (c) => {
  const id = Number(c.req.param('id'))
  const db = createDb(c.env.DB)
  await db.delete(items).where(eq(items.id, id))
  return c.json({ data: { deleted: true } })
})

export { app as itemsRouter }
```

## Step 7: Create the Entry Point

```typescript
// api/src/index.ts
import { Hono } from 'hono'
import { cors } from 'hono/cors'
import { itemsRouter } from './domains/items/items.routes'

type Bindings = { DB: D1Database }
const app = new Hono<{ Bindings: Bindings }>()

// CORS — required so the frontend (port 3001) can talk to the API (port 8787)
app.use('/*', cors())
app.get('/', (c) => c.json({ status: 'running' }))
app.route('/items', itemsRouter)

export default app
```

## Step 8: Generate Migration + Create D1 Database

> **First time using Wrangler?** Run `wrangler login` first — it opens a browser window
> to authenticate with your Cloudflare account. Only needed once per machine.

```bash
# From api/ directory:
wrangler login                                   # One-time auth (opens browser)
npx drizzle-kit generate                        # Creates migrations/0000_*.sql
wrangler d1 create todo-db                      # Creates DB, outputs database_id
# ↑ Copy the database_id into wrangler.toml (replace LOCAL_PLACEHOLDER)
wrangler d1 migrations apply todo-db            # Apply migrations locally
```

## Step 9: Test the API Locally

```bash
wrangler dev    # Ready on http://localhost:8787
```

Test: `curl http://localhost:8787/items` → `{"data":[]}`.
Create: `curl -X POST http://localhost:8787/items -H "Content-Type: application/json" -d '{"title":"Buy groceries"}'`

## Step 10: Deploy the API

```bash
wrangler deploy                                 # Output: https://todo-api.your-name.workers.dev
wrangler d1 migrations apply todo-db --remote   # Apply migrations to production DB
```

Verify: `curl https://todo-api.your-name.workers.dev/items` → `{"data":[]}`

---

## Step 11: Create the Frontend

```bash
# From the todo-app/ root (not api/):
npm create vite@latest client -- --template react-ts
cd client && npm install && npm install @tanstack/react-query
```

## Step 12: API Client + Vite Proxy

```bash
# Create the lib directory first (Vite scaffold doesn't include it)
mkdir -p client/src/lib
```

```typescript
// client/src/lib/api.ts
const baseURL = import.meta.env.VITE_API_URL || '/api'

export async function apiFetch<T>(path: string, options?: RequestInit): Promise<T> {
  const res = await fetch(`${baseURL}${path}`, {
    headers: { 'Content-Type': 'application/json' },
    ...options,
  })
  if (!res.ok) {
    const body = await res.json().catch(() => ({}))
    throw new Error(body.error || res.statusText)
  }
  return res.json()
}
```

```typescript
// client/vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 3001,
    proxy: {
      '/api': {
        target: 'http://127.0.0.1:8787',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
    },
  },
})
```

## Step 13: Build the To-Do UI

```typescript
// client/src/App.tsx
import { QueryClient, QueryClientProvider, useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { apiFetch } from './lib/api'
import { useState } from 'react'

const queryClient = new QueryClient()

interface Item { id: number; title: string; completed: boolean; createdAt: string }

function TodoApp() {
  const [title, setTitle] = useState('')
  const qc = useQueryClient()

  const { data, isLoading } = useQuery({
    queryKey: ['items'],
    queryFn: () => apiFetch<{ data: Item[] }>('/items'),
  })
  const createMutation = useMutation({
    mutationFn: (title: string) =>
      apiFetch('/items', { method: 'POST', body: JSON.stringify({ title }) }),
    onSuccess: () => { qc.invalidateQueries({ queryKey: ['items'] }); setTitle('') },
  })
  const toggleMutation = useMutation({
    mutationFn: (id: number) => apiFetch(`/items/${id}`, { method: 'PUT' }),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['items'] }),
  })
  const deleteMutation = useMutation({
    mutationFn: (id: number) => apiFetch(`/items/${id}`, { method: 'DELETE' }),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['items'] }),
  })

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault()
    if (title.trim()) createMutation.mutate(title.trim())
  }

  if (isLoading) return <p>Loading...</p>
  return (
    <div style={{ maxWidth: 500, margin: '2rem auto', fontFamily: 'sans-serif' }}>
      <h1>To-Do App</h1>
      <form onSubmit={handleSubmit} style={{ display: 'flex', gap: 8, marginBottom: 16 }}>
        <input value={title} onChange={(e) => setTitle(e.target.value)}
          placeholder="What needs to be done?" style={{ flex: 1, padding: 8 }} />
        <button type="submit">Add</button>
      </form>
      <ul style={{ listStyle: 'none', padding: 0 }}>
        {data?.data.map((item) => (
          <li key={item.id} style={{ display: 'flex', alignItems: 'center', gap: 8, padding: '4px 0' }}>
            <input type="checkbox" checked={item.completed}
              onChange={() => toggleMutation.mutate(item.id)} />
            <span style={{ flex: 1, textDecoration: item.completed ? 'line-through' : 'none' }}>
              {item.title}
            </span>
            <button onClick={() => deleteMutation.mutate(item.id)}>x</button>
          </li>
        ))}
      </ul>
    </div>
  )
}

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <TodoApp />
    </QueryClientProvider>
  )
}
```

## Step 14: Run Full Stack Locally

Two terminals:
```bash
# Terminal 1 — API:    cd todo-app/api && wrangler dev
# Terminal 2 — Client: cd todo-app/client && npm run dev
```

Open `http://localhost:3001` — add, toggle, delete items.

## Step 15: Deploy Everything

```bash
# 1. API (from api/)
wrangler d1 migrations apply todo-db --remote
wrangler deploy
# Output: https://todo-api.your-name.workers.dev

# 2. Frontend (from client/)
# Set the production API URL, then build and deploy:
# Linux/macOS:
VITE_API_URL=https://todo-api.your-name.workers.dev npm run build
# Windows (PowerShell):
# $env:VITE_API_URL="https://todo-api.your-name.workers.dev"; npm run build

npx wrangler pages deploy dist --project-name=todo-client
# Output: https://todo-client.pages.dev
```

## Final Project Structure

```
todo-app/
├── api/
│   ├── src/
│   │   ├── index.ts                  # Hono entry + CORS
│   │   ├── db/
│   │   │   ├── index.ts              # createDb factory
│   │   │   └── schema.ts             # barrel export
│   │   └── domains/items/
│   │       ├── items.schema.ts       # table + Zod validators
│   │       └── items.routes.ts       # CRUD endpoints
│   ├── migrations/                   # Generated by drizzle-kit
│   ├── drizzle.config.ts
│   ├── tsconfig.json
│   ├── wrangler.toml
│   └── package.json
├── client/
│   ├── src/
│   │   ├── App.tsx                   # Main component
│   │   └── lib/api.ts               # API fetch helper
│   ├── vite.config.ts               # Proxy config
│   └── package.json
```

## Next Steps

- Add user auth → `references/auth-security.md`
- Add file uploads → `references/database-storage.md` (R2 section)
- Add GitHub Actions CI/CD → `references/deployment-cicd.md`
- Improve architecture → `references/architecture-bestpractices.md`
