# grill-with-docs

Grilling session that challenges your plan against the existing domain model,
sharpens terminology, and updates documentation (CONTEXT.md, ADRs) inline as
decisions crystallise.

Use when the user wants to stress-test a plan against their project's language
and documented decisions. Accepts an optional Jira ticket key as an argument
(e.g. `/grill-with-docs WSCR-123`).

---

## Instructions

### Step 1 — Gather context

**If a Jira ticket key was provided as an argument**, fetch it before doing
anything else:

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  "$JIRA_URL/rest/api/3/issue/<TICKET_KEY>?expand=renderedFields" \
  -H "Accept: application/json"
```

Extract and hold in context:
- Summary (ticket title)
- Description (the stated requirement or problem)
- Acceptance criteria (if present in the description or a custom field)
- Comments (decisions or clarifications already made)
- Reporter and assignee (useful for understanding who owns the intent)

If the ticket content contradicts what the user has said in conversation, do not
silently resolve it — surface it as the first grilling question.

If `$JIRA_URL`, `$JIRA_EMAIL`, or `$JIRA_PAT` are not set, note it and continue
the grilling session without ticket context rather than halting.

**Explore the codebase** to understand the current state. Look for:

```
/
├── CONTEXT.md
├── CONTEXT-MAP.md       ← if present, repo has multiple contexts
├── docs/
│   └── adr/
│       ├── 0001-*.md
│       └── 0002-*.md
└── src/
```

If a `CONTEXT-MAP.md` exists at the root, the repo has multiple contexts. Read
the map to find the relevant context for the area being changed.

Read all existing ADRs in the area the plan touches. If a decision has already
been made and documented, challenge any plan that contradicts it before
proceeding.

---

### Step 2 — Run the grilling session

Interview the user relentlessly about every aspect of the plan until reaching a
shared understanding. Use the ticket content (if fetched) as additional context
— weave ticket details into questions naturally rather than reciting them back.

Walk down each branch of the design tree, resolving dependencies between
decisions one-by-one.

For each question:
- Provide your recommended answer based on what you know about the codebase,
  the ticket, and the domain
- Ask one question at a time, waiting for the user's response before continuing
- If a question can be answered by exploring the codebase, explore it instead
  of asking

Areas to probe (adapt to what's relevant for this plan):

- **Scope** — what is explicitly out of scope? What are the boundaries?
- **Domain language** — are the terms being used consistent with `CONTEXT.md`?
  If not, is this a new term that should be added, or a misuse of an existing one?
- **Modules** — which existing modules are touched? Are any new modules needed?
  Are there opportunities for deep modules — simple interfaces over complex
  internals?
- **Data** — what MySQL schema changes are needed, if any? Are they reversible?
- **ADR conflicts** — does this plan respect existing architectural decisions?
  If it deviates, is that intentional and worth a new ADR?
- **Testing** — how will this be verified? What's the acceptance criterion that
  `/tdd` will target?
- **Edge cases** — what happens with null/empty input, concurrent access,
  network failure, unexpected data formats?
- **Rollback** — if this goes wrong in production, what's the recovery path?

---

### Step 3 — Update docs inline

When a term is resolved during the grilling session, update `CONTEXT.md` right
there. Do not batch these up — capture them as they happen.

Use the format in `CONTEXT-FORMAT.md`. Do not couple `CONTEXT.md` to
implementation details. Only include terms that are meaningful to domain experts.

Only offer to create an ADR when **all three** are true:
- **Hard to reverse** — the cost of changing your mind later is meaningful
- **Surprising without context** — a future reader will wonder "why did they do
  it this way?"
- **Result of a real trade-off** — there were genuine alternatives and a
  specific one was chosen for specific reasons

If any of the three is missing, skip the ADR. Use the format in `ADR-FORMAT.md`.

---

### Step 4 — Close out

When the grilling session is complete, give a brief summary of:
- What was agreed
- Any new terms added to `CONTEXT.md`
- Any ADRs created
- Any open questions that weren't resolved (to carry into `/to-prd`)

The user can then run `/to-prd` with no argument — it will synthesise this
conversation into a PRD and publish it to Jira.
