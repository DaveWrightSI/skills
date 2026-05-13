---
name: to-prd
description: Turn the current conversation context into a PRD and attach it to an existing Jira issue (or, with no key, publish it as a new Story). Use when user wants to create a PRD from the current context, formalise a conversation into a spec, or document a feature before breaking it into issues.
---

# to-prd

This skill takes the current conversation context and codebase understanding and produces a PRD. Do NOT interview the user — just synthesize what you already know. If something is genuinely ambiguous, note it in `Further Notes` rather than asking before writing.

**Primary flow:** invoke as `/to-prd <PARENT-KEY>` (e.g. `/to-prd DT-514`). The PRD is rendered to Markdown and attached to that existing Jira issue as `Product Requirement Document.md`. The parent issue is NOT otherwise modified — no description rewrite, no comment, no status change.

**Fallback flow:** if no parent key is provided, create a new Jira **Story** with the PRD as the description (the original behaviour). See "No parent key — create a new Story" below.

The issue tracker and triage label vocabulary should have been provided to you — run `/setup-matt-pocock-skills` if not.

## Process

1. **Explore the repo** to understand the current state of the codebase, if you haven't already. Use the project's domain glossary vocabulary throughout the PRD, and respect any ADRs in the area you're touching.

2. **Sketch out the major modules** you will need to build or modify to complete the implementation. Actively look for opportunities to extract deep modules that can be tested in isolation.

   A deep module (as opposed to a shallow module) is one which encapsulates a lot of functionality in a simple, testable interface which rarely changes.

   Check with the user that these modules match their expectations. Check with the user which modules they want tests written for.

3. **Write the PRD** using the template below.

4. **Publish the PRD to Jira** (see "Publishing to Jira" below). With a parent key, attach as a Markdown file. With no parent key, create a new Story with the `ready-for-agent` label.

<prd-template>

## Problem Statement

The problem that the user is facing, from the user's perspective.

## Solution

The solution to the problem, from the user's perspective.

## User Stories

A LONG, numbered list of user stories. Each user story should be in the format of:

1. As an <actor>, I want a <feature>, so that <benefit>

<user-story-example>
1. As a mobile bank customer, I want to see balance on my accounts, so that I can make better informed decisions about my spending
</user-story-example>

This list of user stories should be extremely extensive and cover all aspects of the feature.

## Implementation Decisions

A list of implementation decisions that were made. This can include:

- The modules (C# / VB.NET classes, services, interfaces) that will be built/modified
- The interfaces of those modules that will be modified
- Technical clarifications from the developer
- Architectural decisions
- MySQL schema changes
- API contracts
- Specific interactions
- Azure DevOps pipeline considerations

Do NOT include specific file paths or code snippets. They may end up being outdated very quickly.

Exception: if a prototype produced a snippet that encodes a decision more precisely than prose can (state machine, reducer, schema, type shape), inline it within the relevant decision and note briefly that it came from a prototype. Trim to the decision-rich parts — not a working demo, just the important bits.

## Testing Decisions

A list of testing decisions that were made. Include:

- A description of what makes a good test (only test external behavior, not implementation details)
- Which modules will be tested
- Prior art for the tests (i.e. similar types of tests in the codebase — typically xUnit/NUnit/MSTest under `*.Tests.csproj` or `*.Tests.vbproj`)

## Out of Scope

A description of the things that are out of scope for this PRD.

## Further Notes

Any further notes about the feature, including open questions and ambiguities that need a decision before or during implementation.

</prd-template>

## Publishing to Jira

### With parent key — attach PRD as Markdown file (primary flow)

When invoked as `/to-prd <PARENT-KEY>`, render the PRD body as Markdown, write it to a tempfile, and upload it as an attachment to the existing issue. The filename shown in Jira is taken from the upload — name it `Product Requirement Document.md`.

```bash
# 1. Render the PRD body to a tempfile. Use mktemp so the file lives in the
#    OS temp dir and is not written into the working repo.
TMPFILE="$(mktemp --suffix=.md)"

cat > "$TMPFILE" <<'MD'
# Product Requirement Document

## Problem Statement
<problem text>

## Solution
<solution text>

## User Stories
1. As a <actor>, I want a <feature>, so that <benefit>
2. ...

## Implementation Decisions
- <decision 1>
- <decision 2>

## Testing Decisions
- <decision 1>

## Out of Scope
- <item 1>

## Further Notes
- <note 1>
MD

# 2. Upload as an attachment. X-Atlassian-Token: no-check is required by Jira
#    for the attachments endpoint. The ;filename= override controls the name
#    that appears in the Jira UI, regardless of the tempfile's actual name.
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  -X POST \
  -H "X-Atlassian-Token: no-check" \
  -F "file=@$TMPFILE;filename=Product Requirement Document.md" \
  "$JIRA_URL/rest/api/3/issue/<PARENT_KEY>/attachments"

# 3. Clean up the tempfile.
rm -f "$TMPFILE"
```

After upload, output:

- The parent Jira issue key + URL (e.g. `DT-514` → `https://smartinsider.atlassian.net/browse/DT-514`)
- The attachment filename (`Product Requirement Document.md`)
- A one-line summary of what was attached

Do NOT modify the parent issue's description, labels, status, or comments. The attachment is the entire output.

### No parent key — create a new Story (fallback flow)

If the user did not provide a parent key, fall back to creating a new Jira issue of type **Story** in the project's configured project key. Use ADF (Atlassian Document Format) for the description body.

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  -H "Content-Type: application/json" \
  -X POST "$JIRA_URL/rest/api/3/issue" \
  -d '{
    "fields": {
      "project": { "key": "<PROJECT_KEY>" },
      "summary": "<Feature name — matches PRD title>",
      "issuetype": { "name": "Story" },
      "labels": ["ready-for-agent"],
      "description": {
        "type": "doc",
        "version": 1,
        "content": [
          {
            "type": "heading",
            "attrs": { "level": 2 },
            "content": [{ "type": "text", "text": "Problem Statement" }]
          },
          {
            "type": "paragraph",
            "content": [{ "type": "text", "text": "<problem text>" }]
          },
          {
            "type": "heading",
            "attrs": { "level": 2 },
            "content": [{ "type": "text", "text": "Solution" }]
          },
          {
            "type": "paragraph",
            "content": [{ "type": "text", "text": "<solution text>" }]
          },
          {
            "type": "heading",
            "attrs": { "level": 2 },
            "content": [{ "type": "text", "text": "User Stories" }]
          },
          {
            "type": "orderedList",
            "content": [
              {
                "type": "listItem",
                "content": [
                  {
                    "type": "paragraph",
                    "content": [{ "type": "text", "text": "<story 1>" }]
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

Extend the `content` array with headings + paragraphs for `Implementation Decisions`, `Testing Decisions`, `Out of Scope`, and `Further Notes` in the same shape.

After creating the issue, output:

- The new Jira issue key (e.g. `WSCR-124`)
- The full URL (e.g. `https://smartinsider.atlassian.net/browse/WSCR-124`)
- A one-line summary of what was created

If a source ticket key was referenced in the conversation, add a comment on the source ticket linking to the new PRD issue:

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

## Guardrails

- Never invent domain terminology — use only vocabulary from `CONTEXT.md`
- Never mark a PRD issue as Done or In Progress — status changes are human decisions
- In the primary (attach) flow, the parent issue must NOT be edited beyond adding the attachment. No description rewrite, no comment, no label change, no status change.
- Always clean up the tempfile after upload (`rm -f "$TMPFILE"`) so PRD content does not linger on disk.
- Never hardcode `$JIRA_EMAIL`, `$JIRA_PAT`, or `$JIRA_URL` — always reference as environment variables
- If the Jira project key is ambiguous (the repo could belong to WSCR or DT), ask the user before creating the issue
- The PRD describes the **what and why** — not a step-by-step implementation plan. Implementation detail belongs in the child issues created by `/to-issues`
