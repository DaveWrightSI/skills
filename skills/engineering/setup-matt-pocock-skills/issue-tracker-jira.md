# Issue tracker — Jira

Issues for this repo live in Jira. Use the Jira REST API for all issue
operations. Never use `gh issue create` or write to `.scratch/`.

## Connection

- **Base URL:** `$JIRA_URL` (e.g. `https://smartinsider.atlassian.net`)
- **Auth:** HTTP Basic — email `$JIRA_EMAIL`, API token `$JIRA_PAT`
- **Project key:** <!-- e.g. WSCR or DT — set during /setup-matt-pocock-skills -->
- **API version:** REST API v3 (`/rest/api/3/`)

All three environment variables must be set in the shell session before running
any skill that touches the issue tracker. Never hardcode credentials.

## Creating an issue

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  -H "Content-Type: application/json" \
  -X POST "$JIRA_URL/rest/api/3/issue" \
  -d '{
    "fields": {
      "project": { "key": "<PROJECT_KEY>" },
      "summary": "<title>",
      "description": {
        "type": "doc",
        "version": 1,
        "content": [
          {
            "type": "paragraph",
            "content": [{ "type": "text", "text": "<body>" }]
          }
        ]
      },
      "issuetype": { "name": "<Story|Task|Bug>" }
    }
  }'
```

## Fetching an issue

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  "$JIRA_URL/rest/api/3/issue/<ISSUE_KEY>" | \
  jq '{key: .key, summary: .fields.summary, status: .fields.status.name,
       description: .fields.description}'
```

## Adding a comment

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  -H "Content-Type: application/json" \
  -X POST "$JIRA_URL/rest/api/3/issue/<ISSUE_KEY>/comment" \
  -d '{
    "body": {
      "type": "doc",
      "version": 1,
      "content": [
        {
          "type": "paragraph",
          "content": [{ "type": "text", "text": "<comment text>" }]
        }
      ]
    }
  }'
```

## Applying a label / transitioning status

Apply a label:
```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  -H "Content-Type: application/json" \
  -X PUT "$JIRA_URL/rest/api/3/issue/<ISSUE_KEY>" \
  -d '{"fields": {"labels": ["<label>"]}}'
```

Get available transitions:
```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  "$JIRA_URL/rest/api/3/issue/<ISSUE_KEY>/transitions"
```

Apply a transition:
```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  -H "Content-Type: application/json" \
  -X POST "$JIRA_URL/rest/api/3/issue/<ISSUE_KEY>/transitions" \
  -d '{"transition": {"id": "<transition_id>"}}'
```

## Creating a child issue (sub-task or Story under Epic)

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  -H "Content-Type: application/json" \
  -X POST "$JIRA_URL/rest/api/3/issue" \
  -d '{
    "fields": {
      "project": { "key": "<PROJECT_KEY>" },
      "summary": "<title>",
      "issuetype": { "name": "Subtask" },
      "parent": { "key": "<PARENT_ISSUE_KEY>" }
    }
  }'
```

## Searching issues (JQL)

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PAT" \
  "$JIRA_URL/rest/api/3/search?jql=project=<PROJECT_KEY>+AND+status='To Do'&fields=summary,status,labels"
```

## Notes

- Prefer Story for new feature work, Task for technical tasks, Bug for defects.
- Status transitions are human decisions — the agent may suggest a transition but
  must not apply it without explicit confirmation from the developer.
- When referencing a Jira issue in a commit message or PR description, use the
  full issue key (e.g. `WSCR-123`) so Azure DevOps and Jira can cross-link.
- The Jira description field uses Atlassian Document Format (ADF) JSON — use the
  structure shown above, not plain text strings.
