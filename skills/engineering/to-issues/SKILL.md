---
name: to-issues
description: Break a plan, spec, or PRD into independently-grabbable Jira sub-tasks under an existing parent issue, using tracer-bullet vertical slices. Use when user wants to convert a plan into issues, create implementation tickets, or break down a PRD into workable slices.
---

# to-issues

Break a plan into independently-grabbable Jira **sub-tasks** under an existing parent issue, using vertical slices (tracer bullets).

**Primary flow:** invoke as `/to-issues <PARENT-KEY>` (e.g. `/to-issues DT-514`). All slices are created as sub-tasks of that parent issue — they appear inside the parent and inherit its project.

**Fallback flow:** if no parent key is provided, create standalone Tasks in the project (the original behaviour). See step 5.

The issue tracker and triage label vocabulary should have been provided to you — run `/setup-matt-pocock-skills` if not.

## Process

### 1. Gather context

Work from whatever is already in the conversation context. The parent Jira key passed as the skill argument identifies the issue under which all sub-tasks will be created. Fetch it and read its full body, attachments, and comments before proceeding — a PRD attached by `/to-prd` lives there as `Product Requirement Document.md`:

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  "$JIRA_URL/rest/api/3/issue/<PARENT_KEY>?expand=renderedFields" \
  -H "Accept: application/json"
```

If a `Product Requirement Document.md` attachment is present, download and read it for the full spec before drafting slices:

```bash
# The issue JSON above contains `fields.attachment[]`. Find the entry whose
# `filename` equals "Product Requirement Document.md" and fetch its `content` URL.
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" -L "<ATTACHMENT_CONTENT_URL>"
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

For each approved slice, create a Jira **Sub-task** under the parent issue identified by `<PARENT_KEY>`. Publish in dependency order (blockers first) so you can reference real issue keys in the "Blocked by" field.

**Labelling rule:** apply `ready-for-agent` **only to AFK slices**. HITL slices wait on a human, so labelling them `ready-for-agent` would mislead any AFK agent that's pulling work via the standard `labels = "ready-for-agent"` JQL. HITL Sub-tasks should be created with no labels (or whatever HITL-specific label the project uses).

**Issue type name varies by project.** The sub-task issue type is called `Sub-task` in company-managed Jira projects and `Subtask` in team-managed projects. Confirm the exact name for the parent's project once before the first create call — either via the MCP tool `getJiraIssueTypeMetaWithFields` or:

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  "$JIRA_URL/rest/api/3/issue/createmeta?projectKeys=<PROJECT_KEY>&expand=projects.issuetypes" \
  -H "Accept: application/json"
```

If neither name matches what the project exposes, ask the user before proceeding.

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

Use ADF for the description body. The sub-task inherits its project from the parent, but the `project.key` field is still required by the create endpoint — use the parent's project key.

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  -H "Content-Type: application/json" \
  -X POST "$JIRA_URL/rest/api/3/issue" \
  -d '{
    "fields": {
      "project": { "key": "<PROJECT_KEY>" },
      "summary": "<Slice title>",
      "issuetype": { "name": "Sub-task" },
      "parent": { "key": "<PARENT_KEY>" },
      "labels": ["ready-for-agent"],  // AFK only — omit for HITL slices
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

If no parent key was provided, fall back to creating standalone **Task** issues in the project — set `issuetype.name` to `Task` and omit the `parent` field.

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

Then add a comment on the parent issue listing all created sub-task keys so the thread stays connected. This summary comment is the only modification permitted on the parent — no description edits, no label changes, no status changes.

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
- The parent issue must NOT be edited beyond the single summary comment in step 6. No description rewrite, no label change, no status change.
- Sub-tasks inherit the parent's project — do not pass a different `project.key`.
- Never apply `ready-for-agent` to HITL slices — that label means "an AFK agent can grab this", and an agent grabbing a HITL slice will get stuck or do the wrong thing.
- Verify the sub-task issue type name (`Sub-task` vs `Subtask`) for the parent's project before the first create call; ask the user if neither matches.
- Never hardcode `$JIRA_EMAIL`, `$JIRA_PAT`, or `$JIRA_URL` — always environment variables
- If the project key is ambiguous (WSCR vs DT), ask before creating
- AFK issues should be genuinely self-contained — if an AFK issue actually requires a human decision buried inside it, reclassify it as HITL
- Commit messages for work on these issues must include the Jira key (e.g. `WSCR-125: Add IAnnouncementParser interface`) so Azure DevOps and Jira cross-link automatically
