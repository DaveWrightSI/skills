---
name: to-issues
description: Break a plan, spec, or PRD into independently-grabbable Jira issues using tracer-bullet vertical slices. Use when user wants to convert a plan into issues, create implementation tickets, or break down a PRD into workable slices.
---

# to-issues

Break a plan into independently-grabbable Jira issues using vertical slices (tracer bullets).

The issue tracker and triage label vocabulary should have been provided to you — run `/setup-matt-pocock-skills` if not.

## Process

### 1. Gather context

Work from whatever is already in the conversation context. If the user passes a Jira issue key as an argument (e.g. `WSCR-124`), fetch it and read its full body and comments before proceeding:

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  "$JIRA_URL/rest/api/3/issue/<ISSUE_KEY>?expand=renderedFields" \
  -H "Accept: application/json"
```

### 2. Explore the codebase (optional)

If you have not already explored the codebase, do so to understand the current state of the code. Issue titles and descriptions should use the project's domain glossary vocabulary, and respect ADRs in the area you're touching.

### 3. Draft vertical slices

Break the plan into **tracer bullet** issues. Each issue is a thin vertical slice that cuts through ALL integration layers end-to-end, NOT a horizontal slice of one layer.

Slices may be 'HITL' or 'AFK'. HITL slices require human interaction, such as an architectural decision or a design review. AFK slices can be implemented and merged without human interaction. Prefer AFK over HITL where possible.

<vertical-slice-rules>
- Each slice delivers a narrow but COMPLETE path through every relevant layer (MySQL schema + repository + service + controller + tests — whatever applies to this repo)
- A completed slice is demoable or verifiable on its own
- Prefer many thin slices over few thick ones
- Each slice must be independently grabbable — a dev should be able to pick it up without needing another slice to be done first (except where explicitly noted)
</vertical-slice-rules>

### 4. Quiz the user

Present the proposed breakdown as a numbered list. For each slice, show:

- **Title**: short descriptive name in domain vocabulary
- **Type**: HITL / AFK
- **Layers**: which layers it touches (e.g. "MySQL schema + repository + service + tests")
- **Blocked by**: which other slices (if any) must complete first
- **User stories covered**: which user stories this addresses (if the source material has them)

Ask the user:

- Does the granularity feel right? (too coarse / too fine)
- Are the dependency relationships correct?
- Should any slices be merged or split further?
- Are the correct slices marked as HITL and AFK?

Iterate until the user approves the breakdown. **Do not create any Jira issues until the user explicitly approves the full list.**

### 5. Publish the issues to Jira

For each approved slice, create a Jira **Task** as a child of the parent PRD Story (if one was provided). Publish in dependency order (blockers first) so you can reference real issue keys in the "Blocked by" field. These issues are considered ready for AFK agents, so publish them with the `ready-for-agent` label unless instructed otherwise.

<issue-template>

## Parent

A reference to the parent PRD Jira issue (if the source was an existing issue, otherwise omit).

## What to build

A concise description of this vertical slice. Describe the end-to-end behavior, not layer-by-layer implementation.

Avoid specific file paths or code snippets — they go stale fast. Exception: if a prototype produced a snippet that encodes a decision more precisely than prose can (state machine, reducer, schema, type shape), inline it here and note briefly that it came from a prototype. Trim to the decision-rich parts — not a working demo, just the important bits.

## Layers touched

C# / VB.NET / MySQL / Azure DevOps pipeline — list only what applies.

## Acceptance criteria

- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

## Type

HITL or AFK.

## Blocked by

- A reference to the blocking ticket (if any)

Or "None - can start immediately" if no blockers.

</issue-template>

Use ADF for the description body:

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
      "labels": ["ready-for-agent"],
      "description": {
        "type": "doc",
        "version": 1,
        "content": [
          {
            "type": "heading",
            "attrs": { "level": 2 },
            "content": [{ "type": "text", "text": "What to build" }]
          },
          {
            "type": "paragraph",
            "content": [{ "type": "text", "text": "<description>" }]
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
            "type": "taskList",
            "attrs": { "localId": "ac" },
            "content": [
              {
                "type": "taskItem",
                "attrs": { "localId": "ac-1", "state": "TODO" },
                "content": [{ "type": "text", "text": "<criterion>" }]
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
      }
    }
  }'
```

If no parent PRD key was provided, create standalone Tasks in the project.

If a slice depends on another slice, add a comment on the dependent issue after creation noting the dependency by key (e.g. `Depends on WSCR-125`).

### 6. Output a summary

After all issues are created, output a summary table:

```
| # | Key | Title | Type | URL |
|---|---|---|---|---|
| 1 | WSCR-125 | Add IAnnouncementParser interface | AFK | https://... |
| 2 | WSCR-126 | Implement XML parser for RNS format | AFK | https://... |
| 3 | WSCR-127 | Review parser output format with analyst | HITL | https://... |
```

Then add a comment on the parent PRD issue (if one exists) listing all created child issue keys so the thread stays connected:

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

Do NOT close or modify any parent issue.

## Guardrails

- Never create issues without explicit user approval of the breakdown
- Never create more than one issue per slice — if a slice feels too big, propose splitting it before creating anything
- Never mark any issue as In Progress or Done — status changes are human decisions
- Never hardcode `$JIRA_EMAIL`, `$JIRA_PAT`, or `$JIRA_URL` — always environment variables
- If the project key is ambiguous (WSCR vs DT), ask before creating
- AFK issues should be genuinely self-contained — if an AFK issue actually requires a human decision buried inside it, reclassify it as HITL
- Commit messages for work on these issues must include the Jira key (e.g. `WSCR-125: Add IAnnouncementParser interface`) so Azure DevOps and Jira cross-link automatically
