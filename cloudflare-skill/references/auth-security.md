# Authentication & Security

Cloudflare does NOT have a built-in user auth service. You bring your own.

## Decision Matrix

| Option | Type | Cost | Best For |
|--------|------|------|----------|
| **Better Auth** | Self-hosted OSS library | Free | Most apps (recommended) |
| **Custom JWT** | DIY | Free | Simple APIs, full control |
| **Clerk** | Managed SaaS | Free tier, then paid | Fast setup, don't want to manage auth |
| **Cloudflare Access** | Reverse proxy auth | Free (50 users) | Internal tools, staging environments |
| **Turnstile** | Bot protection | Free (1M req/mo) | Forms, signups, preventing abuse |

---

## Better Auth (Recommended)

Best current option for Cloudflare Workers. Works with D1 + Drizzle + Hono.

**Main project:** https://github.com/better-auth/better-auth (https://www.better-auth.com/)
**Cloudflare adapter:** https://github.com/zpg6/better-auth-cloudflare (community wrapper for CF integration)
**Hono example:** https://hono.dev/examples/better-auth-on-cloudflare
**Docs:** https://www.better-auth.com/docs

### Features

- Email/password, magic links
- OAuth providers (Google, GitHub, Discord, etc.)
- Session management (cookie-based)
- D1 (SQLite) database adapter
- Drizzle ORM integration
- Hono middleware
- Free, open source

### Setup

```bash
npm install better-auth
npm install -D @better-auth/cli
```

> **IMPORTANT:** Better Auth requires the `nodejs_compat` compatibility flag in `wrangler.toml`
> because it uses `node:async_hooks` internally. Without it, you get runtime errors.
> ```toml
> compatibility_flags = ["nodejs_compat"]
> ```

> **CPU LIMIT FIX:** Better Auth uses bcrypt by default, which **exceeds the Workers free-tier
> CPU limit (10ms)**. You'll get Error 1102 on signup/login. Fix: provide a custom PBKDF2
> hasher using Web Crypto API (runs natively on Workers, well within limits):
> ```typescript
> emailAndPassword: {
>   enabled: true,
>   password: {
>     hash: async (password) => { /* PBKDF2 — see auth.ts example below */ },
>     verify: async ({ password, hash }) => { /* PBKDF2 verify */ },
>   },
> },
> ```
> See the full PBKDF2 implementation in the "Auth Configuration" section below.

### Auth Configuration (API Side)

```typescript
// api/src/auth.ts
import { betterAuth } from 'better-auth'
import { drizzleAdapter } from 'better-auth/adapters/drizzle'
import { drizzle } from 'drizzle-orm/d1'

type AuthEnv = { DB: D1Database; BETTER_AUTH_SECRET: string; BETTER_AUTH_URL: string }

// Web Crypto PBKDF2 hasher — bcrypt exceeds Workers free-tier CPU limits (10ms)
async function hashPassword(password: string): Promise<string> {
  const salt = crypto.getRandomValues(new Uint8Array(16))
  const key = await crypto.subtle.importKey(
    'raw', new TextEncoder().encode(password), 'PBKDF2', false, ['deriveBits']
  )
  const hash = await crypto.subtle.deriveBits(
    { name: 'PBKDF2', salt, iterations: 100000, hash: 'SHA-256' }, key, 256
  )
  const saltB64 = btoa(String.fromCharCode(...salt))
  const hashB64 = btoa(String.fromCharCode(...new Uint8Array(hash)))
  return `pbkdf2:100000:${saltB64}:${hashB64}`
}

async function verifyPassword(data: { password: string; hash: string }): Promise<boolean> {
  const parts = data.hash.split(':')
  if (parts[0] !== 'pbkdf2') return false
  const iterations = parseInt(parts[1])
  const salt = Uint8Array.from(atob(parts[2]), c => c.charCodeAt(0))
  const expected = atob(parts[3])
  const key = await crypto.subtle.importKey(
    'raw', new TextEncoder().encode(data.password), 'PBKDF2', false, ['deriveBits']
  )
  const hash = await crypto.subtle.deriveBits(
    { name: 'PBKDF2', salt, iterations, hash: 'SHA-256' }, key, 256
  )
  const actual = String.fromCharCode(...new Uint8Array(hash))
  return actual === expected
}

// Factory function — D1 binding only available inside request handlers, not at module level
export function createAuth(env: AuthEnv) {
  const db = drizzle(env.DB)

  // IMPORTANT: drizzle adapter needs explicit { provider: 'sqlite' }
  // If you have a Drizzle schema, also pass { schema } to avoid BetterAuthError
  return betterAuth({
    database: drizzleAdapter(db, { provider: 'sqlite' }),
    secret: env.BETTER_AUTH_SECRET,
    baseURL: env.BETTER_AUTH_URL,
    basePath: '/auth',  // Mount at /auth/* (not default /api/auth) to match Vite proxy
    emailAndPassword: {
      enabled: true,
      password: { hash: hashPassword, verify: verifyPassword },  // PBKDF2 instead of bcrypt
    },
    // IMPORTANT: When frontend (Pages) and API (Worker) are on different origins,
    // Better Auth rejects requests with "Invalid origin". Add all frontend origins here:
    trustedOrigins: [
      'https://my-app.pages.dev',    // Cloudflare Pages production
      'http://localhost:5173',         // Vite dev server
    ],
  })
}
```

### Generate Auth Tables + Migrate

Better Auth needs its own tables (`user`, `session`, `account`, `verification`).

```bash
# Generate auth schema file from Better Auth config
npx @better-auth/cli generate --config src/auth.ts --output src/db/auth-schema.ts -y

# Add to barrel export: src/db/schema.ts
# export * from './auth-schema'

# Generate Drizzle migration (picks up auth tables automatically)
npx drizzle-kit generate

# Apply migrations
wrangler d1 migrations apply my-db          # local
wrangler d1 migrations apply my-db --remote  # production
```

> **CLI caveat:** `@better-auth/cli generate` does NOT work with the factory-pattern
> auth config required for Workers (where `env` is only available per-request). The CLI
> expects a default export or top-level `auth` variable. **Workaround:** Create the
> migration SQL manually with the tables: `user`, `session`, `account`, `verification`.
> See Better Auth source `get-tables.ts` for exact column definitions.

### Mount Auth Routes in Hono

```typescript
// api/src/index.ts
import { Hono } from 'hono'
import { cors } from 'hono/cors'
import { createAuth } from './auth'

type Bindings = { DB: D1Database; BETTER_AUTH_SECRET: string; BETTER_AUTH_URL: string }
const app = new Hono<{ Bindings: Bindings }>()

// CORS — include both dev and production origins, credentials required for auth cookies
app.use('/*', cors({
  origin: ['http://localhost:3001', 'https://your-app.pages.dev'],
  credentials: true,
}))

// Auth routes at /auth/* (matches basePath in auth config)
// In dev: frontend sends /api/auth/... → Vite proxy strips /api → Worker receives /auth/...
// In prod: frontend sends https://api.example.com/auth/... → Worker receives /auth/...
app.on(['GET', 'POST'], '/auth/*', (c) => {
  const auth = createAuth(c.env)
  return auth.handler(c.req.raw)
})

// Mount other routes
app.route('/items', itemsRouter)
```

### Auth Middleware (Protect Routes)

```typescript
// api/src/middleware/auth.ts
import { createAuth } from '../auth'

export function authMiddleware() {
  return async (c: any, next: any) => {
    const auth = createAuth(c.env)
    const session = await auth.api.getSession({ headers: c.req.raw.headers })
    if (!session) return c.json({ error: 'Unauthorized' }, 401)
    c.set('user', session.user)
    c.set('session', session.session)
    await next()
  }
}

// Usage: app.use('/items/*', authMiddleware())
// Access: const user = c.get('user')  // { id, email, name, ... }
```

### Frontend Auth Client

```typescript
// client/src/lib/auth-client.ts
import { createAuthClient } from 'better-auth/react'

export const authClient = createAuthClient({
  baseURL: import.meta.env.VITE_API_URL || '/api',
  basePath: '/auth',  // Must match server basePath
  fetchOptions: { credentials: 'include' },  // Required for auth cookies
})

export const { useSession } = authClient
```

### Frontend Login/Signup Component

```typescript
// client/src/components/AuthForm.tsx
import { useState } from 'react'
import { authClient } from '../lib/auth-client'

export function LoginForm() {
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const [error, setError] = useState('')

  const handleLogin = async (e: React.FormEvent) => {
    e.preventDefault()
    const { error } = await authClient.signIn.email({ email, password })
    if (error) setError(error.message)
  }

  const handleSignup = async (e: React.FormEvent) => {
    e.preventDefault()
    const { error } = await authClient.signUp.email({ email, password, name: email })
    if (error) setError(error.message)
  }

  return (
    <form>
      <input type="email" value={email} onChange={e => setEmail(e.target.value)} placeholder="Email" />
      <input type="password" value={password} onChange={e => setPassword(e.target.value)} placeholder="Password" />
      {error && <p style={{ color: 'red' }}>{error}</p>}
      <button type="button" onClick={handleLogin}>Log In</button>
      <button type="button" onClick={handleSignup}>Sign Up</button>
    </form>
  )
}
```

### Auth-Aware Route Protection (Frontend)

```typescript
// client/src/components/ProtectedRoute.tsx
import { LoginForm } from './AuthForm'
import { useSession } from '../lib/auth-client'

export function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const { data: session, isPending } = useSession()

  if (isPending) return <p>Loading...</p>
  if (!session) return <LoginForm />
  return <>{children}</>
}
```

### Secrets to Set

```bash
wrangler secret put BETTER_AUTH_SECRET    # openssl rand -base64 32
wrangler secret put BETTER_AUTH_URL       # https://todo-api.your-name.workers.dev
```

For local dev, create `.dev.vars`:
```env
BETTER_AUTH_SECRET=local-dev-secret-change-me
BETTER_AUTH_URL=http://localhost:8787
```

### When to Use

- Auth that runs entirely on Cloudflare (no external services)
- Using D1 + Drizzle (native integration)
- Need OAuth + email/password
- Want to own user data

---

## Custom JWT Pattern (Minimal DIY)

For maximum control with zero auth dependencies. Good for APIs.

### Login Endpoint

```typescript
import { SignJWT, jwtVerify } from 'jose'

app.post('/auth/login', async (c) => {
  const { email, password } = await c.req.json()
  const db = createDb(c.env.DB)

  // Verify credentials (hash comparison omitted for brevity)
  const user = await db.select().from(users).where(eq(users.email, email)).get()
  if (!user || !await verifyPassword(password, user.passwordHash)) {
    return c.json({ error: 'Invalid credentials' }, 401)
  }

  // Generate JWT
  const secret = new TextEncoder().encode(c.env.JWT_SECRET)
  const token = await new SignJWT({ sub: user.id, role: user.role })
    .setProtectedHeader({ alg: 'HS256' })
    .setExpirationTime('24h')
    .sign(secret)

  return c.json({ data: { token } })
})
```

### Auth Middleware

```typescript
app.use('/api/*', async (c, next) => {
  const token = c.req.header('Authorization')?.replace('Bearer ', '')
  if (!token) return c.json({ error: 'Unauthorized' }, 401)

  try {
    const secret = new TextEncoder().encode(c.env.JWT_SECRET)
    const { payload } = await jwtVerify(token, secret)
    c.set('userId', payload.sub as string)
    c.set('userRole', payload.role as string)
    await next()
  } catch {
    return c.json({ error: 'Invalid token' }, 401)
  }
})
```

Store the secret: `wrangler secret put JWT_SECRET`

### When to Use

- Simple API with token-based auth
- You want full control, minimal dependencies
- You understand JWT security implications

---

## Clerk (Managed Auth)

Fully managed authentication service. Best DX but adds external dependency.

**Website:** https://clerk.com
**Workers SDK:** https://clerk.com/docs/guides/development/sdk-development/backend-only

### Pros
- 5-minute setup
- Pre-built UI components (sign-in, sign-up, user profile)
- Social login, MFA, organization management
- Free tier: 10,000 monthly active users

### Cons
- External dependency (vendor lock-in)
- Costs at scale ($0.02/MAU after free tier)
- Data stored on Clerk's servers, not yours

---

## Cloudflare Access (Internal Tools)

Protects entire applications behind identity provider login. NOT a user auth
system - it's a gate that sits in front of your app.

**Docs:** https://developers.cloudflare.com/cloudflare-one/policies/access/

### Pricing

| | Free | Paid |
|---|---|---|
| Users | 50 | $3/user/month (Access only) |
| Identity providers | GitHub, Google, Azure AD, Okta | Same + SAML/OIDC |

### When to Use

- Protect admin panels, staging environments, internal dashboards
- You want zero-code auth (no changes to your app)
- You have <50 internal users

### How It Works

1. Set up an Access Application in CF dashboard
2. Configure identity provider (e.g., GitHub login)
3. Cloudflare puts a login screen in front of your app
4. Only authenticated users can reach your Worker/Pages

**Zero Trust docs:** https://developers.cloudflare.com/cloudflare-one/

---

## Turnstile (Bot Protection / CAPTCHA Alternative)

Invisible bot detection. No visual puzzles for real users. Privacy-preserving.

**Docs:** https://developers.cloudflare.com/turnstile/

### Pricing

- Free: 1M requests/month, 20 widgets
- Beyond free: Enterprise Bot Management only ($2,000+/mo)

### Modes

- **Managed** - shows checkbox only when suspicious
- **Non-interactive** - no user interaction needed
- **Invisible** - completely hidden from users

### Client-Side

```html
<script src="https://challenges.cloudflare.com/turnstile/v0/api.js" async defer></script>
<div class="cf-turnstile" data-sitekey="YOUR_SITE_KEY"></div>
```

### Server-Side Verification

```typescript
app.post('/submit', async (c) => {
  const { 'cf-turnstile-response': token } = await c.req.json()

  const verification = await fetch('https://challenges.cloudflare.com/turnstile/v0/siteverify', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      secret: c.env.TURNSTILE_SECRET,
      response: token,
    }),
  })

  const result = await verification.json()
  if (!result.success) return c.json({ error: 'Bot detected' }, 403)

  // Process form...
})
```

### Setup

1. Cloudflare dashboard → Turnstile → Add Widget
2. Get site key (client) + secret key (server)
3. Store secret: `wrangler secret put TURNSTILE_SECRET`

### When to Use

- Login/signup forms (prevent brute force)
- Any public form (contact, comments)
- API rate limiting complement
- Replace reCAPTCHA (better privacy, better UX)

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Secrets management | https://developers.cloudflare.com/workers/configuration/secrets/ |
| Environment variables | https://developers.cloudflare.com/workers/configuration/environment-variables/ |
| Cloudflare Access | https://developers.cloudflare.com/cloudflare-one/policies/access/ |
| Zero Trust overview | https://developers.cloudflare.com/cloudflare-one/ |
| Zero Trust pricing | https://www.cloudflare.com/plans/zero-trust-services/ |
| Turnstile | https://developers.cloudflare.com/turnstile/ |
| Turnstile server-side | https://developers.cloudflare.com/turnstile/get-started/server-side-validation/ |
| Better Auth + Hono | https://hono.dev/examples/better-auth-on-cloudflare |
| better-auth-cloudflare | https://github.com/zpg6/better-auth-cloudflare |
