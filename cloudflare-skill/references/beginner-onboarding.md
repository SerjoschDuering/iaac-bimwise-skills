# Beginner Onboarding: What Is All This?

No tech background needed. Start here.

---

## What Are We Even Building?

A **web app** is anything you use in a browser: Instagram, Google Docs, Airbnb, a to-do list.
Every web app has the same basic parts, even if it looks completely different on the surface.

Think of it like a restaurant:

| Restaurant | Web App | Cloudflare Name |
|---|---|---|
| The menu and dining room (what customers see) | **Frontend** - buttons, forms, pages | Workers + Static Assets |
| The kitchen (where orders are processed) | **Backend** - the logic that runs when you click something | Workers (edge functions) |
| The recipe book and order history | **Database** - stores information (users, posts, orders) | D1 |
| The pantry and walk-in fridge | **File storage** - images, PDFs, videos | R2 |
| The lock on the door + ID check | **Authentication** - login, signup, "who are you?" | Better Auth |
| The suggestion box | **Forms + bot protection** - making sure real humans submit things | Turnstile |

**That's it.** Every web app is just these pieces wired together.

---

## What Does Cloudflare Actually Do For Me?

Without Cloudflare, to put an app on the internet you'd need to:
- Rent a server ($20-100/month)
- Install an operating system
- Set up a web server (nginx, Apache)
- Configure SSL certificates (HTTPS)
- Set up a database server
- Handle backups, scaling, monitoring
- Pray nothing crashes at 3 AM

**With Cloudflare, all of that is handled for you.** Write code, type `wrangler deploy`, done.
Your app runs on 300+ servers worldwide, scales automatically, and the free tier covers
most hobby/learning projects.

### The magic of "serverless"

"Serverless" doesn't mean no servers - it means *you don't think about servers*.
Cloudflare runs your code when someone visits your app, and stops when nobody's using it.
No server to manage, no bills when idle.

### The magic of "edge"

"Edge" means your code runs close to the user. If someone in Tokyo visits your app,
it runs in Tokyo. Someone in London? Runs in London. Fast for everyone, automatically.

---

## What Can I Build?

Here are real project ideas, sorted from simplest to most complex.
Each one tells you which Cloudflare services it uses.

### Starter Projects (1-2 days)

**Personal portfolio / blog**
- What: A website showing your work, projects, resume
- Services: Workers + Static Assets (just frontend, no backend needed)
- Learn: HTML/CSS/React basics, deployment

**Link shortener (like bit.ly)**
- What: Enter a long URL, get a short one that redirects
- Services: Workers (backend logic) + KV (store the URL mappings)
- Learn: How APIs work, key-value storage

**Contact form with email notifications**
- What: A form on your website that sends you an email when submitted
- Services: Workers (process form) + Turnstile (block spam bots)
- Learn: Form handling, bot protection

### Intermediate Projects (1-2 weeks)

**To-do app with user accounts**
- What: Users sign up, create/manage their own to-do lists
- Services: Workers + D1 (database) + Better Auth (login/signup)
- Learn: CRUD operations, authentication, database design

**Image gallery / file sharing**
- What: Upload images, browse gallery, share links
- Services: Workers + D1 (metadata) + R2 (store images)
- Learn: File uploads, object storage, presigned URLs

**Recipe book / note-taking app**
- What: Create, search, and organize recipes or notes with categories
- Services: Workers + D1 (content + categories) + R2 (photos)
- Learn: Relational data, search, rich content

**Real-time poll / voting app**
- What: Create polls, share links, see results update live
- Services: Workers + D1 (poll data) + KV (fast vote counting)
- Learn: Real-time updates, caching strategies

### Advanced Projects (2-4 weeks)

**Multi-user task board (like Trello)**
- What: Teams create boards, add cards, assign tasks, drag to reorder
- Services: Workers + D1 + Better Auth + KV (sessions)
- Learn: Complex state management (Zustand), multi-user data, drag-and-drop

**Marketplace / classifieds**
- What: Users post items for sale, browse listings, filter by category
- Services: Workers + D1 + R2 (product images) + Better Auth + Turnstile
- Learn: Full CRUD, image uploads, search/filter, pagination

**AI-powered content tool**
- What: Summarize articles, generate descriptions, analyze text
- Services: Workers + Workers AI (LLM inference) + D1 (save results)
- Learn: AI integration, streaming responses, prompt engineering

**SaaS dashboard with analytics**
- What: Users connect data sources, see charts/graphs, export reports
- Services: Workers + D1 + Hyperdrive (external DB) + Better Auth + Queues (background processing)
- Learn: Data visualization, background jobs, external database integration

---

## Picking Your First Project - A Guide

Not sure where to start? Answer these questions:

**What interests you?**
- "I want something visual" → Portfolio, Image gallery
- "I want to learn backend/APIs" → Link shortener, To-do app
- "I want to build something useful" → Note-taking app, Task board
- "I want to try AI" → AI content tool
- "I just want to deploy something fast" → Portfolio (can be done in 30 minutes)

**How much time do you have?**
- An afternoon → Link shortener or portfolio
- A weekend → To-do app or contact form
- A week → Image gallery or recipe book
- Two weeks → Task board or marketplace

**What do you want to learn?**
- "Databases" → To-do app (simplest DB project)
- "File uploads" → Image gallery
- "Authentication" → Any project with user accounts
- "The whole stack" → Marketplace (touches everything)

---

## How a Request Flows Through Your App

When someone clicks a button on your app, here's what actually happens:

```
1. User clicks "Save" in the browser
         ↓
2. Frontend (React) sends a request to your API
   POST /api/items  { "title": "Buy milk" }
         ↓
3. Cloudflare routes the request to the nearest Worker
         ↓
4. Worker (your backend code) runs:
   - Validates the data (is "title" a real string?)
   - Checks authentication (is this user logged in?)
   - Saves to D1 database
   - Returns a response
         ↓
5. Frontend receives the response
   { "data": { "id": 42, "title": "Buy milk" } }
         ↓
6. React updates the screen to show the new item
```

Every feature in every app follows this same pattern. Once you understand it,
you can build anything.

---

## Building Steps (For AI Agents + Students)

> **Primary audience:** AI coding agents (Claude Code, etc.) executing on behalf of students.
> Students make architecture decisions and approve; the AI writes and deploys code.

### Step 1: Set Up + Deploy Hello World

Account, CLI, first deploy. The AI agent handles all commands.

> **Execute using:** `references/getting-started.md` (full setup flow)

### Step 2: Add Database + API Routes

Create D1 database, define Drizzle schema, build CRUD endpoints with Hono.

> **Execute using:** `references/tutorial-todo-app.md` (complete walkthrough)
> **Reference:** `references/database-storage.md` for D1/R2/KV details, `references/js-patterns.md` for patterns

### Step 3: Build Frontend

React + Vite + TanStack Query. Connect to API via Vite proxy. Add CORS to the API.

> **Execute using:** `references/tutorial-todo-app.md` (Steps 11-15)
> **Reference:** `references/js-patterns.md` for frontend patterns

### Step 4: Add Auth

Better Auth for login/signup, session management, route protection.

> **Execute using:** `references/auth-security.md` (Better Auth section has full code)

### Step 5: Deploy Full Stack

Deploy API to Workers, build frontend with production URL, deploy to Pages.

> **Execute using:** `references/deployment-cicd.md` for CLI + GitHub Actions
> **Reference:** `references/tutorial-todo-app.md` Step 16

### Step 6: Extend

Add file uploads (R2), background jobs (Queues), AI features (Workers AI), custom domain.

> **Reference:** `references/database-storage.md` for R2/KV/Queues, `references/compute.md` for Workers AI and containers

---

## Glossary: Tech Words in Plain English

| Term | What It Actually Means |
|------|----------------------|
| **API** | A set of URLs your frontend calls to get/send data. Like a waiter taking orders. |
| **Backend** | Code that runs on a server, not in the browser. The kitchen. |
| **Frontend** | Code that runs in the browser. What users see and click. The dining room. |
| **Database** | Where data is stored permanently. Like a filing cabinet. |
| **Deploy** | Putting your code on the internet so others can use it. |
| **Edge** | Running code close to users worldwide, not in one data center. |
| **Serverless** | Code runs on-demand, no server to manage. Pay only when used. |
| **CLI** | Command Line Interface. The terminal where you type commands. |
| **CRUD** | Create, Read, Update, Delete. The 4 basic operations on data. |
| **ORM** | A library that lets you write database queries in your programming language instead of SQL. |
| **Endpoint** | A specific URL your API responds to, like `GET /items` or `POST /users`. |
| **Middleware** | Code that runs before your main logic. Like a bouncer checking IDs. |
| **Environment variable** | A setting stored outside your code. Like a sticky note on the fridge. |
| **Secret** | An environment variable that's sensitive (passwords, API keys). Never in code. |
| **Migration** | A script that changes your database structure. Like remodeling the filing cabinet. |
| **CI/CD** | Automation that deploys your code when you push to GitHub. Set-and-forget. |
| **Static assets** | Files that don't change: HTML, CSS, JS, images. |
| **Binding** | How Cloudflare connects your code to services (database, storage, etc.). |
| **Wrangler** | Cloudflare's CLI tool. The one command you'll use for everything. |
| **Workers** | Cloudflare's serverless functions. Where your backend code runs. |
| **D1** | Cloudflare's database (SQLite). Where your data lives. |
| **R2** | Cloudflare's file storage (S3-compatible). Where your files live. |
| **KV** | Cloudflare's key-value store. Fast lookups for simple data. |
