# Development Patterns

## New Feature Workflow

When adding a new feature:

1. **Read** the IFCore skill and understand the codebase
2. **Create** a folder: `feature-plans/F<N>-<name>/`
3. **Write** `PRD.md` in that folder (see template below)
4. **Build** — implement the feature, update AGENTS.md with learnings
5. **Review** — when done, open a fresh chat and paste the handover prompt

## Feature Folder Structure

```
feature-plans/
└── F1-door-width-check/
    ├── PRD.md
    ├── phase-a/        # if the feature has multiple phases
    │   └── PRD.md      # scope for this phase only
    └── learnings.md    # what went wrong, what to do differently
```

For large features, divide into phases. Each phase gets its own subfolder with a PRD.

## PRD Template

```markdown
# F<N>: <Feature Name>

## Executive Summary
<!-- Explain the goal in plain language — as if to someone non-technical.
     What problem does this solve? What will users be able to do? -->

## User Stories
- As a [user], I want to [action] so that [benefit]
- ...

## Acceptance Criteria
- [ ] ...

## Technical Notes
<!-- Frameworks, libraries, constraints — brief -->

## Phases (if large)
- Phase A: ...
- Phase B: ...
```

## After You Finish

Suggest this to the user:

> "Feature complete. Recommend reviewing in a fresh chat to catch anything missed.
> Copy this prompt to start the review:"

```
I just finished implementing [feature name].
Codebase is at [path]. Key files changed: [list].
Please review against the PRD at feature-plans/F<N>-<name>/PRD.md
and check for IFCore schema compliance and code conventions.
```

If subagents are available, offer to run the review automatically.
