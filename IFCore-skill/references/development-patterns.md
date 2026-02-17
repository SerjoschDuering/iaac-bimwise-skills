# Development Patterns

## How to Work With the User

You're helping someone who may be new to coding, AI, or both. Before doing anything:

- **Ask what they want to build.** Don't assume — let them explain in their own words.
- **Suggest an approach and explain why.** If you recommend something, say what it does
  and why it's a good fit. Offer to explain any term they might not know.
- **Break things into small steps.** One thing at a time. Check in after each step.
- **Use plain language.** Say "a function that checks door widths" not "a callable that
  validates dimensional properties." If you must use a technical term, explain it inline.

## When Someone Wants to Add a New Check

Start a conversation — don't jump straight to code.

1. **Understand what they want.** Ask: "What rule or regulation should this check enforce?"
   Get a concrete example — "doors must be at least 900mm wide" is better than "door check."
2. **Talk through the approach.** Explain what the check function will do, what IFC data
   it needs, and roughly how it works. Ask if that sounds right.
3. **Suggest a plan.** Offer to write a short plan first (see template below). This is
   a simple document that describes the goal and what "done" looks like. It helps avoid
   building the wrong thing. If the user prefers to skip it, that's fine — go straight to code.
4. **Build it together.** Write the check function, test it, and walk through the result.
   Explain what each part does. Update AGENTS.md with anything you learned.
5. **Review when done.** Suggest a quick review — either in a fresh chat or with a subagent
   if available. This catches things you both might have missed.

## Feature Plans (Optional but Helpful)

For bigger features, a plan folder keeps things organized:

```
feature-plans/
└── F1-door-width-check/
    ├── plan.md          # what we're building and why
    ├── phase-a/         # if the feature has multiple phases
    │   └── plan.md      # scope for this phase only
    └── learnings.md     # what went wrong, what to do differently
```

A "plan" (sometimes called a PRD — Product Requirements Document) is just a short
description of what you're building. It helps you think before you code.

## Plan Template

```markdown
# F<N>: <Feature Name>

## What Are We Building?
<!-- Explain the goal in plain language. What problem does this solve?
     What will users be able to do after this is built? -->

## What Does "Done" Look Like?
- [ ] ...
<!-- A checklist. When every box is ticked, the feature is done. -->

## How It Works
<!-- Brief description of the approach. What IFC data do we need?
     What does the check function do? -->

## Phases (if it's a big feature)
- Phase A: ...
- Phase B: ...
```

Don't overthink this. A few sentences per section is fine. Skip "Phases" for small checks.

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

If subagents are available, offer to run the review automatically.
