# to-issues

Break a plan, spec, or PRD into independently-grabbable Jira issues using
tracer-bullet vertical slices.

Use when the user wants to convert a plan into issues, create implementation
tickets, or break down a PRD into workable slices.

The issue tracker and triage label vocabulary should have been provided to you —
run `/setup-matt-pocock-skills` if not.

---

## Instructions

### Step 1 — Gather context

Work from whatever is already in the conversation. Then:

**If the user passes a Jira issue key as an argument** (e.g. `WSCR-124`), fetch
it and read its full body and comments before proceeding:

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  "$JIRA_URL/rest/api/3/issue/<ISSUE_KEY>?expand=renderedFields" \
  -H "Accept: application/json"
```

**If you have not already explored the codebase**, do so now to understand the
current state of the code. Read `CONTEXT.md` for domain vocabulary and any ADRs
under `docs/adr/` that touch the area being changed.

Issue titles and descriptions must use the project's domain glossary vocabulary
and respect ADRs in the area you're touching.

---

### Step 2 — Propose the breakdown

Break the plan into tracer bullet issues. Each issue is a thin **vertical slice**
that cuts through ALL integration layers end-to-end — NOT a horizontal slice of
one layer.

**Vertical slice rules:**
- Each slice delivers a narrow but complete path through every relevant layer
  (data access, service, controller, tests — whatever applies to this repo)
- A completed slice is independently demoable or verifiable on its own
- Prefer many thin slices over few thick ones
- Each slice must be independently grabbable — a dev should be able to pick it up
  without needing another slice to be done first (except where explicitly noted)

**Slice classification — every slice is either HITL or AFK:**

| Type | Meaning | Examples |
|---|---|---|
| **HITL** | Requires human interaction to complete or verify | Architectural decision, design review, stakeholder sign-off, manual QA of a UI |
| **AFK** | Can be implemented, tested, and merged without human interaction | Pure logic, data transformation, API endpoint with clear spec, test coverage |

Prefer AFK over HITL wherever possible. If a slice could go either way, make it
AFK and note the assumption.

**Present the proposed breakdown as a numbered list.** For each slice show:

```
1. [AFK] <Slice title in domain vocabulary>
   Why: <One sentence — what this delivers and why it's a slice boundary>
   Layers: <Which layers it touches, e.g. "MySQL schema + repository + service + tests">
   Depends on: <Issue number(s) if any, else "none">
```

Iterate with the user until the breakdown is approved. Do not create any Jira
issues until the user explicitly approves the full list.

---

### Step 3 — Create Jira issues

For each approved slice, create a Jira **Task** as a child of the parent PRD
Story (if one was provided). Use ADF for the description body.

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  -H "Content-Type: application/json" \
  -X POST "$JIRA_URL/rest/api/3/issue" \
  -d '{
    "fields": {
      "project": { "key": "<PROJECT_KEY>" },
      "summary": "<Slice title>",
      "issuetype": { "name": "Task" },
      "parent": { "key": "<PARENT_PRD_KEY>" },
      "description": {
        "type": "doc",
        "version": 1,
        "content": [
          {
            "type": "heading",
            "attrs": { "level": 2 },
            "content": [{ "type": "text", "text": "What this slice delivers" }]
          },
          {
            "type": "paragraph",
            "content": [{ "type": "text", "text": "<why sentence>" }]
          },
          {
            "type": "heading",
            "attrs": { "level": 2 },
            "content": [{ "type": "text", "text": "Layers touched" }]
          },
          {
            "type": "paragraph",
            "content": [{ "type": "text", "text": "<layers>" }]
          },
          {
            "type": "heading",
            "attrs": { "level": 2 },
            "content": [{ "type": "text", "text": "Acceptance criteria" }]
          },
          {
            "type": "orderedList",
            "content": [
              {
                "type": "listItem",
                "content": [
                  {
                    "type": "paragraph",
                    "content": [{ "type": "text", "text": "<criterion>" }]
                  }
                ]
              }
            ]
          },
          {
            "type": "heading",
            "attrs": { "level": 2 },
            "content": [{ "type": "text", "text": "Type" }]
          },
          {
            "type": "paragraph",
            "content": [{ "type": "text", "text": "<HITL or AFK>" }]
          }
        ]
      },
      "labels": ["<needs-triage label string from triage-labels.md>"]
    }
  }'
```

If no parent PRD key was provided, create standalone Tasks in the project.

If a slice depends on another slice, add a comment on the dependent issue after
creation noting the dependency by key (e.g. `Depends on WSCR-125`).

---

### Step 4 — Output a summary

After all issues are created, output a summary table:

```
| # | Key | Title | Type | URL |
|---|---|---|---|---|
| 1 | WSCR-125 | Add IAnnouncementParser interface | AFK | https://... |
| 2 | WSCR-126 | Implement XML parser for RNS format | AFK | https://... |
| 3 | WSCR-127 | Review parser output format with analyst | HITL | https://... |
```

Then add a comment on the parent PRD issue (if one exists) listing all created
child issue keys so the thread stays connected:

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  -H "Content-Type: application/json" \
  -X POST "$JIRA_URL/rest/api/3/issue/<PARENT_KEY>/comment" \
  -d '{
    "body": {
      "type": "doc",
      "version": 1,
      "content": [
        {
          "type": "paragraph",
          "content": [
            {
              "type": "text",
              "text": "Broken into <N> issues: <KEY-1>, <KEY-2>, <KEY-3>"
            }
          ]
        }
      ]
    }
  }'
```

---

## Guardrails

- Never create issues without explicit user approval of the breakdown
- Never create more than one issue per slice — if a slice feels too big, propose
  splitting it before creating anything
- Never mark any issue as In Progress or Done — status changes are human decisions
- Never hardcode `$JIRA_EMAIL`, `$JIRA_PAT`, or `$JIRA_URL` — always environment
  variables
- If the project key is ambiguous (WSCR vs DT), ask before creating
- AFK issues should be genuinely self-contained — if an AFK issue actually
  requires a human decision buried inside it, reclassify it as HITL
- Commit messages for work on these issues must include the Jira key
  (e.g. `WSCR-125: Add IAnnouncementParser interface`) so Azure DevOps and Jira
  cross-link automatically
