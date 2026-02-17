# JavaScript/TypeScript Patterns for Cloudflare

## Hono API (Workers-Native Framework)

Hono is ultra-lightweight (~14KB), built for edge. Used internally at Cloudflare.

### Entry Point
```typescript
// src/index.ts
import { Hono } from 'hono'
import { cors } from 'hono/cors'
import { usersRouter } from './domains/users/users.routes'
import { itemsRouter } from './domains/items/items.routes'

type Bindings = {
  DB: D1Database
  BUCKET: R2Bucket
  CACHE: KVNamespace
  ENVIRONMENT: string
}

const app = new Hono<{ Bindings: Bindings }>()

// Global middleware
app.use('/*', cors())

// Health check
app.get('/', (c) => c.text('API running'))

// Mount domain routers
const routes = app
  .route('/users', usersRouter)
  .route('/items', itemsRouter)

// Export type for client-side type inference
export type AppType = typeof routes
export default routes
```

### Domain Router (CRUD Example)
```typescript
// src/domains/items/items.routes.ts
import { Hono } from 'hono'
import { zValidator } from '@hono/zod-validator'
import { eq } from 'drizzle-orm'
import { createDb } from '../../db'
import { items, insertItemSchema } from './items.schema'

type Bindings = { DB: D1Database }
const app = new Hono<{ Bindings: Bindings }>()

// GET all
app.get('/', async (c) => {
  const db = createDb(c.env.DB)
  const result = await db.select().from(items).all()
  return c.json({ data: result })
})

// GET by id
app.get('/:id', async (c) => {
  const db = createDb(c.env.DB)
  const id = Number(c.req.param('id'))
  const result = await db.select().from(items).where(eq(items.id, id)).get()
  if (!result) return c.json({ error: 'Not found' }, 404)
  return c.json({ data: result })
})

// POST create
app.post('/', zValidator('json', insertItemSchema), async (c) => {
  const data = c.req.valid('json')
  const db = createDb(c.env.DB)
  const result = await db.insert(items).values(data).returning()
  return c.json({ data: result[0] }, 201)
})

// PUT update
app.put('/:id', zValidator('json', insertItemSchema.partial()), async (c) => {
  const id = Number(c.req.param('id'))
  const data = c.req.valid('json')
  const db = createDb(c.env.DB)
  const result = await db.update(items).set(data).where(eq(items.id, id)).returning()
  if (!result.length) return c.json({ error: 'Not found' }, 404)
  return c.json({ data: result[0] })
})

// DELETE
app.delete('/:id', async (c) => {
  const id = Number(c.req.param('id'))
  const db = createDb(c.env.DB)
  await db.delete(items).where(eq(items.id, id))
  return c.json({ data: { deleted: true } })
})

export { app as itemsRouter }
```

### API Response Wrapper Convention
All responses use: `{ data: T }` or `{ error: string }`

---

## Drizzle ORM + D1

### Schema Definition
```typescript
// src/domains/items/items.schema.ts
import { sqliteTable, integer, text, real } from 'drizzle-orm/sqlite-core'
import { createInsertSchema, createSelectSchema } from 'drizzle-zod'
import { z } from 'zod'

// Auto-increment integer ID
export const items = sqliteTable('items', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  title: text('title').notNull(),
  description: text('description'),
  price: real('price').notNull(),
  status: text('status', { enum: ['active', 'archived'] }).notNull().default('active'),
  createdAt: integer('created_at', { mode: 'timestamp' }).$defaultFn(() => new Date()),
})

// Text ID pattern
export const categories = sqliteTable('categories', {
  id: text('id').primaryKey(),    // UUID or slug
  name: text('name').notNull(),
})

// Auto-generate Zod schemas from Drizzle table
export const insertItemSchema = createInsertSchema(items, {
  title: z.string().min(1, 'Title required'),
  price: z.number().positive(),
})
export const selectItemSchema = createSelectSchema(items)

// Export types
export type Item = z.infer<typeof selectItemSchema>
export type NewItem = z.infer<typeof insertItemSchema>
```

### DB Factory
```typescript
// src/db/index.ts
import { drizzle } from 'drizzle-orm/d1'
import * as schema from './schema'

export const createDb = (d1: D1Database) => drizzle(d1, { schema })
```

```typescript
// src/db/schema.ts  (barrel export all domain schemas)
export * from '../domains/items/items.schema'
export * from '../domains/users/users.schema'
```

### Drizzle Config
```typescript
// drizzle.config.ts
import { defineConfig } from 'drizzle-kit'

export default defineConfig({
  schema: './src/domains/**/*.schema.ts',   // Auto-discovers all schemas!
  out: './migrations',
  dialect: 'sqlite',
  driver: 'd1-http',
})
```

### Migration Workflow
```bash
bun run drizzle-kit generate       # Create SQL migration from schema changes
wrangler d1 migrations apply my-db --remote   # Apply to production
wrangler d1 migrations apply my-db-preview --remote  # Apply to preview
```

---

## Type Sharing (API → Client)

No codegen, no published packages. Direct filesystem references.

### 1. API exports types
```typescript
// api/src/schemas.ts
export type { Item, NewItem } from './domains/items/items.schema'
export type { User, NewUser } from './domains/users/users.schema'
export type ApiResponse<T> = { data: T }
export type ApiError = { error: string }
```

### 2. Vite alias points to API source
```typescript
// client/vite.config.ts
resolve: {
  alias: {
    "@api": path.resolve(__dirname, "../api/src"),
  },
}
```

### 3. TypeScript includes API schemas
```json
// client/tsconfig.app.json
{
  "compilerOptions": {
    "paths": { "@api/*": ["../api/src/*"] }
  },
  "include": ["src", "../api/src/schemas.ts", "../api/src/domains/**/*.schema.ts"]
}
```

### 4. Client imports types
```typescript
import type { Item, ApiResponse } from '@api/schemas'
```

---

## Frontend Patterns (React + Vite)

### Vite Config
```typescript
// client/vite.config.ts
import path from 'path'
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { TanStackRouterVite } from '@tanstack/router-plugin/vite'

export default defineConfig({
  plugins: [
    TanStackRouterVite(),
    react({ babel: { plugins: [['babel-plugin-react-compiler']] } }),
  ],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@api': path.resolve(__dirname, '../api/src'),
    },
  },
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

### API Client
```typescript
// client/src/lib/api.ts
const baseURL = import.meta.env.VITE_API_URL || '/api'

export async function apiFetch<T>(path: string, options?: RequestInit): Promise<T> {
  const res = await fetch(`${baseURL}${path}`, {
    headers: { 'Content-Type': 'application/json' },
    ...options,
  })
  if (!res.ok) throw new Error((await res.json()).error || res.statusText)
  return res.json()
}
```

### Query Key Factory
```typescript
export const itemKeys = {
  all: ['items'] as const,
  list: () => [...itemKeys.all, 'list'] as const,
  detail: (id: number) => [...itemKeys.all, 'detail', id] as const,
}
```

### React Query Hooks
```typescript
// client/src/domains/items/items.api.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { apiFetch } from '@/lib/api'
import type { Item, NewItem, ApiResponse } from '@api/schemas'

export function useItems() {
  return useQuery({
    queryKey: itemKeys.list(),
    queryFn: () => apiFetch<ApiResponse<Item[]>>('/items'),
  })
}

export function useCreateItem() {
  const qc = useQueryClient()
  return useMutation({
    mutationFn: (data: NewItem) =>
      apiFetch<ApiResponse<Item>>('/items', {
        method: 'POST',
        body: JSON.stringify(data),
      }),
    onSuccess: () => qc.invalidateQueries({ queryKey: itemKeys.all }),
  })
}
```

### Zustand Store (UI State)
```typescript
// client/src/domains/items/items.store.ts
import { create } from 'zustand'

interface ItemsStore {
  selectedId: number | null
  setSelected: (id: number | null) => void
  filterStatus: 'all' | 'active' | 'archived'
  setFilter: (status: 'all' | 'active' | 'archived') => void
}

export const useItemsStore = create<ItemsStore>((set) => ({
  selectedId: null,
  setSelected: (id) => set({ selectedId: id }),
  filterStatus: 'all',
  setFilter: (filterStatus) => set({ filterStatus }),
}))
```

---

## Key Dependencies

### API — Install Commands

```bash
# Core dependencies
npm install hono drizzle-orm drizzle-zod @hono/zod-validator zod

# Dev dependencies (build tools + types)
npm install -D wrangler @cloudflare/workers-types drizzle-kit typescript
```

Package versions for reference:
```json
{
  "dependencies": {
    "hono": "^4.x",
    "drizzle-orm": "^0.45.x",
    "drizzle-zod": "^0.8.x",
    "@hono/zod-validator": "^0.7.x",
    "zod": "^3.x"
  },
  "devDependencies": {
    "wrangler": "^4.x",
    "@cloudflare/workers-types": "^4.x",
    "drizzle-kit": "^0.31.x"
  }
}
```

Useful `package.json` scripts:
```json
{
  "scripts": {
    "dev": "wrangler dev",
    "deploy": "wrangler deploy",
    "db:generate": "drizzle-kit generate",
    "db:migrate": "wrangler d1 migrations apply my-db --remote",
    "db:migrate:local": "wrangler d1 migrations apply my-db"
  }
}
```

### Client — Install Commands

```bash
# Core dependencies
npm install react react-dom @tanstack/react-query @tanstack/react-router zustand

# Dev dependencies
npm install -D vite @vitejs/plugin-react babel-plugin-react-compiler @tanstack/router-plugin typescript
```

Package versions for reference:
```json
{
  "dependencies": {
    "react": "^19.x",
    "react-dom": "^19.x",
    "@tanstack/react-query": "^5.x",
    "@tanstack/react-router": "^1.x",
    "zustand": "^5.x"
  },
  "devDependencies": {
    "vite": "^7.x",
    "@vitejs/plugin-react": "^5.x",
    "babel-plugin-react-compiler": "^1.x",
    "@tanstack/router-plugin": "^1.x"
  }
}
```
