# to-prd

Turn the current conversation context into a PRD and publish it to Jira.

Use when the user wants to create a PRD from the current context, formalise a
conversation into a spec, or document a feature before breaking it into issues.

The issue tracker and triage label vocabulary should have been provided to you —
run `/setup-matt-pocock-skills` if not.

---

## Instructions

Do NOT interview the user. Synthesize what you already know from the conversation
and codebase. If something is genuinely ambiguous, note it as an open question in
the PRD rather than asking before writing.

### Step 1 — Gather context

Collect all available inputs before writing anything:

**From the conversation:**
- What the user wants to build or change
- Any constraints, preferences, or decisions already made
- Any existing Jira ticket key the user has referenced (e.g. `WSCR-123`)

**From Jira (if a ticket key was provided):**

Fetch the ticket and extract its description, acceptance criteria, and comments:

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  "$JIRA_URL/rest/api/3/issue/<TICKET_KEY>" \
  -H "Accept: application/json"
```

Read the full ticket body. Use its language, acceptance criteria, and any
decisions recorded in comments as additional context for the PRD. If the ticket
contradicts the conversation, surface the conflict as an open question rather
than silently choosing one.

**From the codebase:**
- Explore the repo to understand the current state if you haven't already
- Read `CONTEXT.md` for the project's domain glossary — use its vocabulary
  throughout the PRD
- Read any ADRs under `docs/adr/` that touch the area being changed
- Identify the modules that will need to change

### Step 2 — Write the PRD

Use the template below. Use the project's domain glossary throughout — never
invent synonyms for established terms. Respect ADRs in the area you're touching.

Sketch out the major modules you will need to build or modify. Actively look for
opportunities to extract deep modules — ones that encapsulate a lot of
functionality behind a simple, testable interface that rarely changes.

---

## PRD template

```markdown
# <Feature name in domain vocabulary>

## Summary
One paragraph. What this feature does and why it exists. Written for a developer
who hasn't seen the conversation.

## Related ticket
<Jira ticket key and URL if provided, e.g. WSCR-123 — https://smartinsider.atlassian.net/browse/WSCR-123>

## Context
What currently exists. What gap or problem this addresses. Reference any relevant
ADRs and why they constrain the approach.

## Scope

### In scope
- Bullet list of what this PRD covers

### Out of scope
- Explicit list of things that are deliberately NOT included in this PRD

## Modules

List the modules (classes, services, interfaces) to build or modify:

| Module | Action | Notes |
|---|---|---|
| `IAnnouncementParser` | Create | New interface, extracted for testability |
| `AnnouncementService` | Modify | Inject parser via constructor |
| `AnnouncementController` | Modify | Minor — endpoint already exists |

Mark any new module as a **deep module opportunity** if it encapsulates
significant complexity behind a simple interface.

## Acceptance criteria
A numbered list of verifiable statements. Each criterion should be independently
testable by `/tdd`.

1. Given X, when Y, then Z
2. ...

## Open questions
Any genuine ambiguities that need a decision before or during implementation.
If none, write "None."

## Implementation notes
Optional. C#-specific patterns, NuGet packages, existing utilities to reuse,
MySQL schema changes, or Azure DevOps pipeline considerations.
```

---

### Step 3 — Publish to Jira

Once the PRD is written, create a new Jira issue of type **Story** in the
project's configured project key. Use ADF (Atlassian Document Format) for the
description body.

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  -H "Content-Type: application/json" \
  -X POST "$JIRA_URL/rest/api/3/issue" \
  -d '{
    "fields": {
      "project": { "key": "<PROJECT_KEY>" },
      "summary": "<Feature name — matches PRD title>",
      "issuetype": { "name": "Story" },
      "description": {
        "type": "doc",
        "version": 1,
        "content": [
          {
            "type": "heading",
            "attrs": { "level": 2 },
            "content": [{ "type": "text", "text": "Summary" }]
          },
          {
            "type": "paragraph",
            "content": [{ "type": "text", "text": "<summary text>" }]
          },
          {
            "type": "heading",
            "attrs": { "level": 2 },
            "content": [{ "type": "text", "text": "Acceptance Criteria" }]
          },
          {
            "type": "orderedList",
            "content": [
              {
                "type": "listItem",
                "content": [
                  {
                    "type": "paragraph",
                    "content": [{ "type": "text", "text": "<criterion 1>" }]
                  }
                ]
              }
            ]
          }
        ]
      }
    }
  }'
```

After creating the issue, output:
- The new Jira issue key (e.g. `WSCR-124`)
- The full URL (e.g. `https://smartinsider.atlassian.net/browse/WSCR-124`)
- A one-line summary of what was created

If a source ticket key was provided, add a comment on the source ticket linking
to the new PRD issue:

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  -H "Content-Type: application/json" \
  -X POST "$JIRA_URL/rest/api/3/issue/<SOURCE_KEY>/comment" \
  -d '{
    "body": {
      "type": "doc",
      "version": 1,
      "content": [
        {
          "type": "paragraph",
          "content": [
            { "type": "text", "text": "PRD created: " },
            {
              "type": "inlineCard",
              "attrs": { "url": "<NEW_ISSUE_URL>" }
            }
          ]
        }
      ]
    }
  }'
```

---

## Guardrails

- Never invent domain terminology — use only vocabulary from `CONTEXT.md`
- Never mark a PRD issue as Done or In Progress — status changes are human
  decisions
- Never hardcode `$JIRA_EMAIL`, `$JIRA_PAT`, or `$JIRA_URL` — always reference
  as environment variables
- If the Jira project key is ambiguous (the repo could belong to WSCR or DT),
  ask the user before creating the issue
- The PRD describes the **what and why** — not a step-by-step implementation
  plan. Implementation detail belongs in the child issues created by `/to-issues`
