# Database & Storage Services

## Decision Matrix

| Need | Service | Free Tier |
|------|---------|-----------|
| Relational data, SQL | **D1** (SQLite) | 5M reads/day, 5GB |
| Files, images, videos | **R2** (S3-compatible) | 10GB, zero egress |
| Config, sessions, cache | **KV** (key-value) | 100K reads/day |
| Background jobs | **Queues** | 10K ops/day |
| Existing Postgres/MySQL | **Hyperdrive** (proxy) | Included (100K queries/day free) |

---

## D1 - Serverless SQLite Database

SQLite database managed by Cloudflare. Scale-to-zero (no charges when idle).
Best for: CRUD apps, user data, relational data under 10GB.

**Docs:** https://developers.cloudflare.com/d1/

### Pricing

| | Free | Paid ($5/mo) |
|---|---|---|
| Rows read | 5M/day | 25B/month incl, then $0.001/M |
| Rows written | 100K/day | 50M/month incl, then $1.00/M |
| Storage | 5GB total | 5GB incl, then $0.75/GB-mo |
| Max DB size | 500MB | 10GB |
| Databases/account | 10 | 50,000 |

**Pricing docs:** https://developers.cloudflare.com/d1/platform/pricing/

### Limits

- Max 10GB per database (hard limit)
- Max row/BLOB size: 2MB
- 100 columns per table
- 100KB max SQL statement
- Query timeout: 30 seconds
- Queries per Worker invocation: 50 (free), 1,000 (paid)
- Time Travel (point-in-time restore): 7 days (free), 30 days (paid)

**Limits docs:** https://developers.cloudflare.com/d1/platform/limits/

### CLI Commands

```bash
wrangler d1 create my-db                                  # Create database
wrangler d1 info my-db                                    # Show DB info
wrangler d1 execute my-db --command "SELECT 1"            # Run SQL
wrangler d1 execute my-db --file ./schema.sql             # Run SQL file
wrangler d1 execute my-db --file ./schema.sql --remote    # Run on remote DB
wrangler d1 migrations apply my-db                        # Apply migrations (local)
wrangler d1 migrations apply my-db --remote               # Apply migrations (remote)
wrangler d1 time-travel restore my-db --timestamp="..."   # Point-in-time restore
```

### Wrangler Config

```toml
[[d1_databases]]
binding = "DB"                    # Access as env.DB in code
database_name = "my-db"
database_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
migrations_dir = "migrations"     # Optional: where migration SQL files live
```

### Usage in JS/TS (Raw SQL)

```typescript
// Read
const { results } = await c.env.DB.prepare("SELECT * FROM users WHERE id = ?").bind(userId).all()

// Write
await c.env.DB.prepare("INSERT INTO users (name, email) VALUES (?, ?)").bind(name, email).run()

// First row
const user = await c.env.DB.prepare("SELECT * FROM users WHERE id = ?").bind(1).first()

// Batch (multiple statements in one round-trip)
await c.env.DB.batch([
  c.env.DB.prepare("INSERT INTO users (name) VALUES (?)").bind("Alice"),
  c.env.DB.prepare("INSERT INTO users (name) VALUES (?)").bind("Bob"),
])
```

### Usage with Drizzle ORM (Recommended for JS/TS)

```typescript
// db/index.ts
import { drizzle } from 'drizzle-orm/d1'
import * as schema from './schema'
export const createDb = (d1: D1Database) => drizzle(d1, { schema })

// Usage in route
const db = createDb(c.env.DB)
const users = await db.select().from(usersTable).all()
await db.insert(usersTable).values({ name: 'Alice' }).returning()
```

Drizzle config for D1:
```typescript
// drizzle.config.ts
import { defineConfig } from 'drizzle-kit'
export default defineConfig({
  schema: './src/domains/**/*.schema.ts',
  out: './migrations',
  dialect: 'sqlite',
  driver: 'd1-http',
})
```

**Drizzle + D1 docs:** https://developers.cloudflare.com/d1/tutorials/d1-and-drizzle/

### Usage in Python

```python
results = await self.env.DB.prepare("SELECT * FROM users").run()
user = await self.env.DB.prepare("SELECT * FROM users WHERE id = ?").bind(1).first()
await self.env.DB.prepare("INSERT INTO users (name) VALUES (?)").bind("Alice").run()
```

**Python + D1 docs:** https://developers.cloudflare.com/d1/examples/query-d1-from-python-workers/

### When NOT to Use D1

- Dataset > 10GB → use Hyperdrive + external Postgres
- High write throughput → consider external DB
- Full-text search at scale → consider external search service

---

## R2 - S3-Compatible Object Storage

Zero egress fees. Fully S3-compatible API. Store files, images, videos, backups.

> **Prereq:** R2 is free but **requires a credit card** on the Cloudflare account.
> Enable it first: Dashboard → R2 Object Storage → "Get Started". CLI commands will
> fail with error 10042 until R2 is activated in the dashboard.

**Docs:** https://developers.cloudflare.com/r2/

### Pricing

| | Free | Paid |
|---|---|---|
| Storage | 10GB/month | $0.015/GB-mo |
| Class A (PUT, POST, LIST) | 1M/month | $4.50/M |
| Class B (GET, HEAD) | 10M/month | $0.36/M |
| Egress | **Free** | **Free** |
| Delete | Free | Free |

The killer feature: **zero egress fees**. AWS S3 charges $0.09/GB for egress.

**Pricing docs:** https://developers.cloudflare.com/r2/pricing/

### CLI Commands

```bash
wrangler r2 bucket create my-files                        # Create bucket
wrangler r2 bucket list                                   # List buckets
wrangler r2 object put my-files/photo.jpg --file=./photo.jpg  # Upload file
wrangler r2 object get my-files/photo.jpg                 # Download file
wrangler r2 object delete my-files/photo.jpg              # Delete file
```

### Wrangler Config

```toml
[[r2_buckets]]
binding = "BUCKET"
bucket_name = "my-files"
```

### Usage in JS/TS

```typescript
// Upload
await c.env.BUCKET.put('images/photo.jpg', imageBuffer, {
  httpMetadata: { contentType: 'image/jpeg' },
  customMetadata: { uploadedBy: 'user-123' },
})

// Download
const object = await c.env.BUCKET.get('images/photo.jpg')
if (object) {
  return new Response(object.body, {
    headers: { 'Content-Type': object.httpMetadata?.contentType || 'application/octet-stream' },
  })
}

// List files
const listing = await c.env.BUCKET.list({ prefix: 'images/', limit: 100 })
for (const obj of listing.objects) {
  console.log(obj.key, obj.size)
}

// Delete
await c.env.BUCKET.delete('images/photo.jpg')

// Check if exists
const head = await c.env.BUCKET.head('images/photo.jpg')
```

### Usage in Python

```python
await self.env.BUCKET.put("files/doc.pdf", file_bytes)
obj = await self.env.BUCKET.get("files/doc.pdf")
listing = await self.env.BUCKET.list(prefix="files/")
await self.env.BUCKET.delete("files/doc.pdf")
```

### Presigned URLs (Direct Client Uploads)

For large file uploads, let the client upload directly to R2:
1. Create R2 API credentials in dashboard
2. Generate presigned URL in your Worker
3. Client uploads directly to the presigned URL

**Docs:** https://developers.cloudflare.com/r2/api/s3/presigned-urls/

---

## KV - Key-Value Store

Globally distributed, eventually consistent. Optimized for high-read, low-write.

**Docs:** https://developers.cloudflare.com/kv/

### Pricing

| | Free | Paid |
|---|---|---|
| Reads | 100K/day | 10M/month incl, $0.50/M |
| Writes | 1K/day | 1M/month incl, $5.00/M |
| Deletes | 1K/day | 1M/month incl, $5.00/M |
| Storage | 1GB | 1GB incl, $0.50/GB-mo |
| Max value size | 25MB | 25MB |

**Pricing docs:** https://developers.cloudflare.com/kv/platform/pricing/

### Key Characteristics

- **Eventually consistent** - writes propagate globally in ~60 seconds
- 1 write/second per key limit
- Hot keys cached at edge for <10ms reads
- TTL (time-to-live) support per key — **minimum TTL is 60 seconds** (lower values cause 500 errors)
- NOT for data needing strong consistency or frequent writes

### CLI Commands

```bash
wrangler kv namespace create MY_CACHE               # Create namespace
wrangler kv namespace list                           # List namespaces
wrangler kv key put --binding=MY_CACHE "key" "val"   # Write key
wrangler kv key get --binding=MY_CACHE "key"         # Read key
wrangler kv key delete --binding=MY_CACHE "key"      # Delete key
wrangler kv key list --binding=MY_CACHE              # List keys
```

### Wrangler Config

```toml
[[kv_namespaces]]
binding = "CACHE"
id = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

### Usage in JS/TS

```typescript
// Write (with optional TTL)
await c.env.CACHE.put('user:123', JSON.stringify(userData), { expirationTtl: 3600 })

// Read
const raw = await c.env.CACHE.get('user:123')
const data = await c.env.CACHE.get('user:123', 'json')     // Auto-parse JSON

// Delete
await c.env.CACHE.delete('user:123')

// List keys
const keys = await c.env.CACHE.list({ prefix: 'user:' })
```

### Best For

- Configuration / feature flags
- Session tokens
- Cached API responses
- Rate limiting (approximate)
- NOT for: frequently updated data, strong consistency needs

---

## Queues - Message Queue

Guaranteed message delivery. Batching, retries, dead letter queues.

**Docs:** https://developers.cloudflare.com/queues/

### Pricing

| | Free | Paid |
|---|---|---|
| Operations | 10K/day | 1M/month incl, $0.40/M |
| Retention | 24 hours | Up to 14 days |

1 operation = 64KB of data. Larger messages count as multiple operations.

**Pricing docs:** https://developers.cloudflare.com/queues/platform/pricing/

### Wrangler Config

```bash
# Create queue (IMPORTANT: must specify retention period, default exceeds API max)
wrangler queues create my-queue --message-retention-period-secs 86400
```

```toml
[[queues.producers]]
binding = "MY_QUEUE"
queue = "my-queue"

[[queues.consumers]]
queue = "my-queue"
max_batch_size = 10
max_batch_timeout = 30
```

> **Local dev note:** Queue consumers do NOT fire during `wrangler dev`. The HTTP
> producer endpoints work locally, but the consumer handler only executes in production
> after `wrangler deploy`. Use `wrangler tail` to verify consumer processing.

### Usage

```typescript
// Producer: send a message from API handler
await c.env.MY_QUEUE.send({ type: 'send-email', to: 'user@example.com', subject: 'Welcome!' })

// Consumer: process messages (add queue export alongside fetch export)
export default {
  async fetch(request, env) { /* API handlers */ },
  async queue(batch, env) {
    for (const msg of batch.messages) {
      try { await processMessage(msg.body, env); msg.ack() }
      catch (e) { msg.retry() }
    }
  },
}
```

### Use Cases

- Background job processing (emails, image processing)
- Webhook fan-out
- Async task offloading
- Decoupling services

---

## Hyperdrive - External Database Accelerator

Connection pooling + query caching for external PostgreSQL/MySQL.
Included at no extra cost (Free plan: 100K queries/day; Paid plan: unlimited).

Use when: you already have a Postgres/MySQL database (Neon, Supabase, RDS, etc.)
and want to access it from Workers without connection overhead.

**Docs:** https://developers.cloudflare.com/hyperdrive/

### Setup

```bash
wrangler hyperdrive create my-pg \
  --connection-string="postgres://user:pass@host:5432/mydb"
```

### Wrangler Config

```toml
[[hyperdrive]]
binding = "HYPERDRIVE"
id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

### Usage

```typescript
import { Client } from 'pg'

app.get('/users', async (c) => {
  const client = new Client({ connectionString: c.env.HYPERDRIVE.connectionString })
  await client.connect()
  const { rows } = await client.query('SELECT * FROM users')
  return c.json({ data: rows })
})
```

**Supported:** PostgreSQL, MySQL, Neon, PlanetScale, CockroachDB, Supabase, AWS RDS, Google Cloud SQL

**Pricing docs:** https://developers.cloudflare.com/hyperdrive/platform/pricing/

---

## Official Documentation Links

| Service | Docs | Pricing | Limits |
|---------|------|---------|--------|
| D1 | https://developers.cloudflare.com/d1/ | https://developers.cloudflare.com/d1/platform/pricing/ | https://developers.cloudflare.com/d1/platform/limits/ |
| R2 | https://developers.cloudflare.com/r2/ | https://developers.cloudflare.com/r2/pricing/ | https://developers.cloudflare.com/r2/platform/limits/ |
| KV | https://developers.cloudflare.com/kv/ | https://developers.cloudflare.com/kv/platform/pricing/ | https://developers.cloudflare.com/kv/platform/limits/ |
| Queues | https://developers.cloudflare.com/queues/ | https://developers.cloudflare.com/queues/platform/pricing/ | https://developers.cloudflare.com/queues/platform/limits/ |
| Hyperdrive | https://developers.cloudflare.com/hyperdrive/ | https://developers.cloudflare.com/hyperdrive/platform/pricing/ | - |
| D1 + Drizzle guide | https://orm.drizzle.team/docs/guides/d1-http-with-drizzle-kit | - | - |
| D1 + Python | https://developers.cloudflare.com/d1/examples/query-d1-from-python-workers/ | - | - |
| R2 presigned URLs | https://developers.cloudflare.com/r2/api/s3/presigned-urls/ | - | - |
