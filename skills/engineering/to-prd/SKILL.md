---
name: to-prd
description: Turn the current conversation context into a PRD and publish it as a Story in Jira. Use when user wants to create a PRD from the current context, formalise a conversation into a spec, or document a feature before breaking it into issues.
---

# to-prd

This skill takes the current conversation context and codebase understanding and produces a PRD. Do NOT interview the user — just synthesize what you already know. If something is genuinely ambiguous, note it in `Further Notes` rather than asking before writing.

The issue tracker and triage label vocabulary should have been provided to you — run `/setup-matt-pocock-skills` if not.

## Process

1. **Explore the repo** to understand the current state of the codebase, if you haven't already. Use the project's domain glossary vocabulary throughout the PRD, and respect any ADRs in the area you're touching.

2. **Sketch out the major modules** you will need to build or modify to complete the implementation. Actively look for opportunities to extract deep modules that can be tested in isolation.

   A deep module (as opposed to a shallow module) is one which encapsulates a lot of functionality in a simple, testable interface which rarely changes.

   Check with the user that these modules match their expectations. Check with the user which modules they want tests written for.

3. **Write the PRD** using the template below.

4. **Publish to Jira** as a Story (see "Publishing to Jira" below). Apply the `ready-for-agent` triage label — no need for additional triage.

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

Create a new Jira issue of type **Story** in the project's configured project key. Use ADF (Atlassian Document Format) for the description body.

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
- Never hardcode `$JIRA_EMAIL`, `$JIRA_PAT`, or `$JIRA_URL` — always reference as environment variables
- If the Jira project key is ambiguous (the repo could belong to WSCR or DT), ask the user before creating the issue
- The PRD describes the **what and why** — not a step-by-step implementation plan. Implementation detail belongs in the child issues created by `/to-issues`
