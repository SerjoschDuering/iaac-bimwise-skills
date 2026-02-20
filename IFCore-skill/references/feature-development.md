# Feature Development Workflow

A step-by-step process for adding new features to the platform.
Works with any AI coding assistant (Claude Code, Cursor, Copilot, etc.).

## The Process

```
1. Describe what you want    (write a short spec)
2. Plan with your AI         (AI proposes an approach, you approve)
3. Build on a branch         (implement + test locally)
4. Test on staging           (push to staging branch, verify)
5. Ship to production        (PR from staging to main)
```

Every feature follows this loop. Even small ones.

## Step 1: Write a Spec

Before touching code, write what you want in plain language. This is your "spec"
(specification). It does not need to be long — 3-5 sentences is enough for small features.

**Template:**
```
What: [one sentence describing the feature]
Why: [what problem does it solve]
Where: [which part of the platform — backend check, frontend UI, API, etc.]
Done when: [how you know it works]
```

**Example:**
```
What: Export check results to a CSV file
Why: Teams want to share results with people who don't use the platform
Where: Frontend — add a download button to the results table
Done when: Clicking the button downloads a CSV with all check results for the current project
```

## Step 2: Plan With Your AI

Give the spec to your AI assistant and ask it to plan the implementation.
The AI has the IFCore skill, so it knows the platform structure and contracts.

### Sample Prompts

**For a new check function:**
```
I want to add a check that verifies all rooms have a minimum ceiling height
of 2.5 meters. The check should look at IfcSpace elements and their height.

Plan the implementation:
- Which checker file should this go in?
- What IFC properties do I need to read?
- What should the check return for pass vs fail?
```

**For a frontend feature:**
```
I want to add a button to the results table that exports all check results
as a CSV file. The CSV should have columns: element name, check name,
status, actual value, required value.

Plan the implementation:
- Which component should the button go in?
- Do I need a new API endpoint or can I do it client-side?
- Show me the file changes needed.
```

**For a backend API endpoint:**
```
I want to add an endpoint that returns check results grouped by team.
The frontend team report panel needs this data.

Plan the implementation:
- Should this be a new route on the CF Worker or on the HF backend?
- What is the response format?
- How does it query the D1 database?
```

**For a platform feature (bigger scope):**
```
I want to add a feature that compares check results between two different
IFC file versions. Users upload v1 and v2 of a building model, and the
platform shows which checks improved, worsened, or stayed the same.

This is a bigger feature. Help me:
1. Break it into smaller tasks
2. Identify which files change (backend, frontend, database)
3. Suggest a phased approach (what to build first)
```

## Step 3: Build on a Branch

Never work directly on `main` or `staging`. Create a feature branch:

```bash
git checkout staging
git pull origin staging
git checkout -b feature/csv-export    # descriptive name
```

**Build and test locally:**
```bash
# Terminal 1 — backend
cd backend && uvicorn main:app --reload --port 7860

# Terminal 2 — frontend
cd frontend && npm run dev
```

**Test your changes** by using the platform in the browser at `http://localhost:5173`.

### Useful Prompts While Building

**When something breaks:**
```
I'm getting this error: [paste the error]
The file I changed is [filename]. Here is what I did: [brief description].
Help me debug this.
```

**When you want a review:**
```
I just finished implementing [feature name].
Key files I changed: [list files].
Review my code for:
- IFCore contract compliance (check functions)
- Any bugs or edge cases I missed
- Whether the approach follows platform patterns
```

**When you are stuck:**
```
I'm trying to [goal] but I don't know how to [specific problem].
The platform uses [relevant tech: React/FastAPI/D1/etc.].
What is the simplest way to do this?
```

## Step 4: Test on Staging

Push your feature branch and merge it into staging:

```bash
git push origin feature/csv-export

# Create a PR from feature branch to staging (via GitHub)
# Or merge locally:
git checkout staging
git merge feature/csv-export
git push origin staging
```

This triggers a deploy to the staging environment. Wait 2-3 minutes, then test
on the staging URLs:
- Frontend: `https://ifcore-platform-staging.tralala798.workers.dev`
- Backend health: `https://serjd-ifcore-platform-staging.hf.space/health`

## Step 5: Ship to Production

Once staging looks good, create a Pull Request from `staging` to `main` on GitHub.

**PR description template:**
```
## What
[One sentence: what this PR adds/changes]

## Why
[One sentence: what problem it solves]

## How to test
1. [Step 1]
2. [Step 2]
3. [Expected result]

## Files changed
- [file1] — [what changed]
- [file2] — [what changed]
```

After 1 approving review, merge. GitHub Actions deploys to production automatically.

## Spec-Driven Development Patterns

These patterns help you work effectively with AI assistants on bigger features.

### Pattern 1: Start With the Contract

For check functions, always start by defining the expected input/output:

```
I want a check that [description].

Before writing code, show me:
1. The function signature (name, arguments)
2. An example return value for a passing element
3. An example return value for a failing element
4. Edge cases (what if the property is missing?)

Follow the IFCore check function contract.
```

### Pattern 2: Build the Smallest Thing First

For frontend features, ask for a minimal version first:

```
Let's build this in two phases:
Phase 1: Just make it work — hardcoded data, no styling, simplest possible.
Phase 2: Hook it up to real data, add error handling, make it look good.

Start with Phase 1.
```

### Pattern 3: Ask for the File Map

Before any multi-file change, ask which files are involved:

```
Before we start coding, show me a file map:
- Which existing files will we modify?
- Do we need any new files?
- Which files should we NOT touch?

List them with a one-line description of what changes in each.
```

### Pattern 4: Test Before You Ship

After implementing, ask your AI to help verify:

```
We just finished [feature]. Before I push:
1. Run the check function against the Duplex Apartment model locally
2. Verify the return format matches the IFCore contract
3. Check that the function handles missing properties gracefully
```

### Pattern 5: Learn From Failures

When something goes wrong, capture it:

```
This bug took a while to find. Add it to my AGENTS.md (or CLAUDE.md)
under "Learnings" so we don't hit it again:
- What happened: [description]
- Root cause: [why]
- Fix: [what we did]
```

## Related Skills

When your AI assistant has these skills installed, it gets deeper knowledge:

- **IFCore Skill** — check function contracts, team integration, validation schema
- **Cloudflare Skill** — how the frontend Worker, D1 database, and R2 storage work
- **HuggingFace Deploy Skill** — how the Docker Space builds and runs
- **PydanticAI Skill** — building AI agents, tools, chat patterns (for the AI chat feature)
