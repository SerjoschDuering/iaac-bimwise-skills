# Debugging & Logging Guide

When something breaks (and it will), logs are your only window into what happened.
This guide teaches you how to see what's going on and fix it.

---

## Logging in Workers (Backend)

### Basic Logging

`console.log` works in Workers. It shows up in `wrangler tail` (live log stream)
and in the Cloudflare dashboard.

```typescript
app.post('/items', async (c) => {
  const data = c.req.valid('json')

  // Log what's coming in - helps you see exactly what the client sent
  console.log('[POST /items] Creating item:', JSON.stringify(data))

  try {
    const result = await db.insert(items).values(data).returning()
    console.log('[POST /items] Created successfully, id:', result[0].id)
    return c.json({ data: result[0] }, 201)
  } catch (error) {
    // Log the FULL error - don't just swallow it
    console.error('[POST /items] Failed to create item:', error)
    return c.json({ error: 'Failed to create item' }, 500)
  }
})
```

### Log Levels

```typescript
console.log('Info - normal operation')          // General info
console.warn('Warning - something unexpected')  // Potential issues
console.error('Error - something broke')        // Actual errors
console.debug('Debug - verbose details')        // Extra detail for debugging
```

### Structured Logging (Recommended)

Instead of string-smashing, log objects. They're searchable and parseable.

```typescript
/**
 * Simple structured logger.
 * Adds timestamp and request context to every log.
 * Makes it much easier to trace what happened and when.
 */
function log(level: 'info' | 'warn' | 'error', message: string, data?: any) {
  const entry = {
    level,
    message,
    timestamp: new Date().toISOString(),
    ...data,
  }
  if (level === 'error') console.error(JSON.stringify(entry))
  else if (level === 'warn') console.warn(JSON.stringify(entry))
  else console.log(JSON.stringify(entry))
}

// Usage
log('info', 'Item created', { itemId: 42, userId: 'abc' })
log('error', 'Database query failed', { query: 'SELECT...', error: err.message })
```

### Request Logging Middleware

Add this to see every request that hits your API:

```typescript
import { Hono } from 'hono'

/**
 * Logs every incoming request and its response status.
 * Helps you see the full traffic pattern and spot failures.
 *
 * Example output:
 *   → GET /items (from 192.168.1.1)
 *   ← GET /items 200 (45ms)
 */
app.use('*', async (c, next) => {
  const start = Date.now()
  const method = c.req.method
  const path = c.req.path

  console.log(`→ ${method} ${path}`)

  await next()

  const duration = Date.now() - start
  const status = c.res.status
  console.log(`← ${method} ${path} ${status} (${duration}ms)`)

  // Warn on slow responses (over 1 second)
  if (duration > 1000) {
    console.warn(`⚠ Slow response: ${method} ${path} took ${duration}ms`)
  }
})
```

---

## Viewing Logs

### Live Log Streaming (wrangler tail)

See logs in real-time from your deployed Worker:

```bash
# Stream all logs
wrangler tail

# Stream logs for preview environment
wrangler tail --env preview

# Filter by status (only errors)
wrangler tail --status error

# Filter by search term
wrangler tail --search "POST /items"

# JSON output (for piping to other tools)
wrangler tail --format json
```

**Docs:** https://developers.cloudflare.com/workers/observability/logs/real-time-logs/

### Cloudflare Dashboard Logs

1. Go to https://dash.cloudflare.com
2. Workers & Pages → your Worker → Logs
3. See recent invocations, errors, and `console.log` output
4. Filter by status code, time range, etc.

### Local Development Logs

When running `wrangler dev`, all `console.log` output appears directly in your terminal.
This is the easiest way to debug - just add logs and watch the terminal.

---

## Debugging Common Issues

### "My API returns 500 but I don't know why"

**Step 1:** Add try/catch with logging to EVERY route handler:

```typescript
app.get('/items', async (c) => {
  try {
    const db = createDb(c.env.DB)
    const result = await db.select().from(items).all()
    return c.json({ data: result })
  } catch (error) {
    // This is the most important line - log the actual error
    console.error('GET /items failed:', error instanceof Error ? error.message : error)
    console.error('Stack:', error instanceof Error ? error.stack : 'no stack')
    return c.json({ error: 'Internal server error' }, 500)
  }
})
```

**Step 2:** Run `wrangler tail` to see the error in real-time.

### "My D1 query returns nothing"

```typescript
// Add logging to see exactly what query runs and what comes back
const query = db.select().from(items).where(eq(items.userId, userId))
console.log('Running query for userId:', userId)

const result = await query.all()
console.log('Query returned', result.length, 'rows:', JSON.stringify(result))
// Often the issue is: userId is undefined, wrong type, or the table is empty
```

### "CORS error in browser console"

This means your frontend (localhost:3001) can't talk to your API (localhost:8787).

```typescript
// Make sure CORS middleware is FIRST, before any routes
import { cors } from 'hono/cors'

app.use('/*', cors({
  origin: ['http://localhost:3001', 'https://your-site.pages.dev'],
  allowMethods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowHeaders: ['Content-Type', 'Authorization'],
}))

// Routes come AFTER cors middleware
app.get('/items', ...)
```

### "TypeError: c.env.DB is undefined"

Your D1 binding isn't configured. Check `wrangler.toml`:

```toml
# Make sure this exists and database_id is correct
[[d1_databases]]
binding = "DB"
database_name = "my-db"
database_id = "your-actual-id-here"  # NOT a placeholder!
```

Run `wrangler d1 list` to see your actual database IDs.

### "My frontend shows stale data"

React Query caches data. After a mutation (create/update/delete),
invalidate the cache:

```typescript
const createItem = useMutation({
  mutationFn: (data) => apiFetch('/items', { method: 'POST', body: JSON.stringify(data) }),
  onSuccess: () => {
    // This tells React Query: "the items list is outdated, refetch it"
    queryClient.invalidateQueries({ queryKey: ['items'] })
  },
})
```

### "My deploy failed"

```bash
# Check wrangler version
wrangler --version

# Check auth
wrangler whoami

# Deploy with verbose output
wrangler deploy --log-level debug

# Check if your wrangler.toml has valid IDs
wrangler d1 list        # verify DB ID
wrangler r2 bucket list # verify bucket name
wrangler kv namespace list  # verify KV ID
```

---

## Frontend Debugging

### React Query DevTools

Add this to see all cached queries, their status, and data:

```typescript
// app/providers.tsx
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'

export function AppProviders({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      {/* Shows a floating panel in dev mode with all query states */}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  )
}
```

Install: `npm install @tanstack/react-query-devtools`

### Network Tab (Browser DevTools)

1. Open browser DevTools (F12 or Cmd+Option+I)
2. Go to Network tab
3. Filter by "Fetch/XHR"
4. Click any request to see:
   - **Headers** - what was sent
   - **Response** - what came back
   - **Status** - 200 (ok), 404 (not found), 500 (server error)

### Console Logging in Components

```tsx
function ItemList() {
  const { data, isLoading, error } = useItems()

  // Temporary debug logging - remove before shipping!
  console.log('[ItemList] render state:', { isLoading, error, itemCount: data?.data?.length })

  if (isLoading) return <p>Loading...</p>
  if (error) return <p>Error: {error.message}</p>

  return (
    <ul>
      {data.data.map(item => <li key={item.id}>{item.title}</li>)}
    </ul>
  )
}
```

---

## Error Handling Pattern

### API: Global Error Handler

Catch ALL unhandled errors in one place:

```typescript
import { Hono } from 'hono'

const app = new Hono()

/**
 * Global error handler.
 * If any route throws an unhandled error, this catches it,
 * logs the details (for you), and returns a clean error (for the user).
 * The user never sees ugly stack traces.
 */
app.onError((err, c) => {
  console.error('Unhandled error:', {
    path: c.req.path,
    method: c.req.method,
    error: err.message,
    stack: err.stack,
  })

  // Don't leak internal details to the client
  return c.json({ error: 'Internal server error' }, 500)
})

// 404 handler - when no route matches
app.notFound((c) => {
  console.warn('404 Not found:', c.req.method, c.req.path)
  return c.json({ error: `Not found: ${c.req.path}` }, 404)
})
```

### Client: Error Boundaries

Catch React rendering errors so the whole app doesn't crash:

```tsx
// components/ErrorBoundary.tsx
import { Component } from 'react'

/**
 * Wraps a section of UI. If anything inside throws during render,
 * this shows a fallback instead of crashing the entire page.
 */
class ErrorBoundary extends Component<
  { children: React.ReactNode; fallback?: React.ReactNode },
  { hasError: boolean; error?: Error }
> {
  state = { hasError: false, error: undefined as Error | undefined }

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error }
  }

  componentDidCatch(error: Error, info: React.ErrorInfo) {
    console.error('React error boundary caught:', error, info.componentStack)
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || (
        <div style={{ padding: 20, color: 'red' }}>
          <h2>Something went wrong</h2>
          <pre>{this.state.error?.message}</pre>
        </div>
      )
    }
    return this.props.children
  }
}
```

---

## Quick Debug Checklist

When something doesn't work, check in this order:

1. **Browser console** - any JavaScript errors?
2. **Network tab** - is the API request going out? What status code?
3. **API response body** - does it contain an error message?
4. **`wrangler tail`** - what does the server log say?
5. **wrangler.toml** - are bindings (DB, KV, R2) configured with real IDs?
6. **`.dev.vars`** - are local secrets set?
7. **`wrangler dev` terminal** - any startup errors?

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Workers logging | https://developers.cloudflare.com/workers/observability/logs/ |
| Real-time logs (wrangler tail) | https://developers.cloudflare.com/workers/observability/logs/real-time-logs/ |
| Workers observability | https://developers.cloudflare.com/workers/observability/ |
| Debugging Workers | https://developers.cloudflare.com/workers/observability/dev-tools/ |
| Error handling | https://developers.cloudflare.com/workers/observability/errors/ |
| React Query DevTools | https://tanstack.com/query/latest/docs/framework/react/devtools |
