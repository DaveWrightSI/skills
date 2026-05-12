---
name: grill-with-docs
description: Grilling session that challenges your plan against the existing domain model, sharpens terminology, and updates documentation (CONTEXT.md, ADRs) inline as decisions crystallise. Use when user wants to stress-test a plan against their project's language and documented decisions. Accepts an optional Jira key as an argument (e.g. `/grill-with-docs WSCR-123`).
---

<what-to-do>

Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer based on what you know about the codebase, the ticket, and the domain.

Ask the questions one at a time, waiting for feedback on each question before continuing.

If a question can be answered by exploring the codebase, explore the codebase instead.

</what-to-do>

<supporting-info>

## Smart Insider context

This repo's stack is C# / VB.NET / MySQL on Azure DevOps with Jira (`smartinsider.atlassian.net`) as the issue tracker. Issue keys typically look like `WSCR-123` or `DT-456`.

### Pull Jira ticket context if a key was supplied

If a Jira key was passed as an argument, fetch it before anything else and weave its details into the grilling questions naturally — don't recite the ticket back.

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  "$JIRA_URL/rest/api/3/issue/<TICKET_KEY>?expand=renderedFields" \
  -H "Accept: application/json"
```

Extract: summary, description, acceptance criteria, comments, reporter, assignee. If the ticket contradicts what the user said in conversation, surface it as the first grilling question.

If `$JIRA_URL`, `$JIRA_EMAIL`, or `$JIRA_PAT` aren't set, note it and continue without ticket context rather than halting.

## Domain awareness

During codebase exploration, also look for existing documentation:

### File structure

Most repos have a single context:

```
/
├── CONTEXT.md
├── docs/
│   └── adr/
│       ├── 0001-event-sourced-orders.md
│       └── 0002-mysql-for-write-model.md
└── src/
```

If a `CONTEXT-MAP.md` exists at the root, the repo has multiple contexts. The map points to where each one lives:

```
/
├── CONTEXT-MAP.md
├── docs/
│   └── adr/                          ← system-wide decisions
├── src/
│   ├── Ordering/
│   │   ├── CONTEXT.md
│   │   └── docs/adr/                 ← context-specific decisions
│   └── Billing/
│       ├── CONTEXT.md
│       └── docs/adr/
```

Create files lazily — only when you have something to write. If no `CONTEXT.md` exists, create one when the first term is resolved. If no `docs/adr/` exists, create it when the first ADR is needed.

## During the session

### Challenge against the glossary

When the user uses a term that conflicts with the existing language in `CONTEXT.md`, call it out immediately. "Your glossary defines 'cancellation' as X, but you seem to mean Y — which is it?"

### Sharpen fuzzy language

When the user uses vague or overloaded terms, propose a precise canonical term. "You're saying 'account' — do you mean the Customer or the User? Those are different things."

### Discuss concrete scenarios

When domain relationships are being discussed, stress-test them with specific scenarios. Invent scenarios that probe edge cases and force the user to be precise about the boundaries between concepts.

### Cross-reference with code

When the user states how something works, check whether the code agrees. If you find a contradiction, surface it: "Your code cancels entire Orders, but you just said partial cancellation is possible — which is right?"

### Probe stack-specific concerns

Adapt the depth of these to what's relevant for the plan:

- **MySQL schema** — what schema changes are needed, if any? Are they reversible? Is the migration safe under concurrent writes?
- **C# / VB.NET module boundaries** — which classes / interfaces are touched? Are there opportunities for deep modules — simple interfaces over complex internals?
- **Azure DevOps pipeline** — does this change require pipeline updates (new build steps, gated approvals, environment variables)?
- **Rollback** — if this goes wrong in production, what's the recovery path? Can it be backed out without a DB migration?

### Update CONTEXT.md inline

When a term is resolved, update `CONTEXT.md` right there. Don't batch these up — capture them as they happen. Use the format in [CONTEXT-FORMAT.md](./CONTEXT-FORMAT.md).

Don't couple `CONTEXT.md` to implementation details. Only include terms that are meaningful to domain experts.

### Offer ADRs sparingly

Only offer to create an ADR when all three are true:

1. **Hard to reverse** — the cost of changing your mind later is meaningful
2. **Surprising without context** — a future reader will wonder "why did they do it this way?"
3. **The result of a real trade-off** — there were genuine alternatives and you picked one for specific reasons

If any of the three is missing, skip the ADR. Use the format in [ADR-FORMAT.md](./ADR-FORMAT.md).

## Closing out

When the grilling session is complete, give a brief summary of:

- What was agreed
- Any new terms added to `CONTEXT.md`
- Any ADRs created
- Any open questions that weren't resolved (to carry into `/to-prd`)

The user can then run `/to-prd` with no argument — it will synthesise this conversation into a PRD and publish it to Jira.

</supporting-info>
