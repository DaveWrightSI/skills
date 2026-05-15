# Issue tracker — Jira

Issues for this repo live in Jira. **Prefer the MCP atlassian tools** for all
issue operations — they are configured for the smartinsider tenant and avoid
needing any shell at all. Use the PowerShell fallback only when the MCP
atlassian tools are not available.

Never use `gh issue create`, never write to `.scratch/`, never use `bash`,
`curl`, or `jq` for Jira (curl/jq are not installed on the team's Windows boxes
and the bash tool is not allowlisted for non-git commands).

## Connection

- **Base URL:** `$env:JIRA_URL` (e.g. `https://smartinsider.atlassian.net`)
- **Auth (PowerShell fallback only):** HTTP Basic — email `$env:JIRA_EMAIL`, API token `$env:JIRA_PAT`
- **Project key:** <!-- e.g. WSCR or DT — set during /setup-matt-pocock-skills -->
- **API version:** REST API v3 (`/rest/api/3/`)

The three env vars are auto-loaded from the workspace `.env` by the
`Import-DotEnv` profile function (or baked in by `Start-JiraTask` when a new
task tab is opened via the `jira` alias). The MCP atlassian server is
configured at the host level — no per-repo setup needed.

## Preferred path — MCP atlassian tools

For each common operation, use the MCP tool first. The cloudId is constant for
the smartinsider tenant; cache it from the first call in a session.

| Operation | MCP tool |
|---|---|
| Resolve cloudId | `mcp__atlassian__getAccessibleAtlassianResources` (cache the first result's `id`) |
| Who am I | `mcp__atlassian__atlassianUserInfo` |
| Fetch an issue | `mcp__atlassian__getJiraIssue` |
| Search issues (JQL) | `mcp__atlassian__searchJiraIssuesUsingJql` |
| List available transitions | `mcp__atlassian__getTransitionsForJiraIssue` |
| Apply a transition | `mcp__atlassian__transitionJiraIssue` |
| Add a comment | `mcp__atlassian__addCommentToJiraIssue` |
| Edit fields (incl. labels) | `mcp__atlassian__editJiraIssue` |
| Create an issue | `mcp__atlassian__createJiraIssue` |
| Get issue-type metadata | `mcp__atlassian__getJiraIssueTypeMetaWithFields` |
| List project issue types | `mcp__atlassian__getJiraProjectIssueTypesMetadata` |
| List visible projects | `mcp__atlassian__getVisibleJiraProjects` |
| Lookup an account ID by email | `mcp__atlassian__lookupJiraAccountId` |
| Get remote issue links | `mcp__atlassian__getJiraIssueRemoteIssueLinks` |
| Create an issue link | `mcp__atlassian__createIssueLink` |
| List issue link types | `mcp__atlassian__getIssueLinkTypes` |
| Add a worklog entry | `mcp__atlassian__addWorklogToJiraIssue` |

When searching with JQL, request only the fields you need (`summary`, `status`,
`created`, `project`, etc.) and keep `maxResults` reasonable — the response is
JSON-rich and can exceed the tool result size limit.

When creating or commenting, Jira description bodies must be in **Atlassian
Document Format (ADF)** — a JSON structure with `type`, `version`, `content`.
Plain strings will be rejected.

## Fallback — PowerShell

If the MCP atlassian tools are unavailable in the current session, use
`Invoke-RestMethod` from the PowerShell tool. Never wrap this in `pwsh
-Command` via the Bash tool.

```powershell
$auth    = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes("$($env:JIRA_EMAIL):$($env:JIRA_PAT)"))
$headers = @{ Authorization = "Basic $auth"; Accept = 'application/json' }
$base    = $env:JIRA_URL.TrimEnd('/') + '/rest/api/3'
```

**Fetch an issue**
```powershell
$issue = Invoke-RestMethod -Uri "$base/issue/<ISSUE_KEY>" -Headers $headers -Method Get
$issue | Select-Object -ExpandProperty fields | Select-Object summary, status, labels
```

**Search (JQL)**
```powershell
$jql = "project = <PROJECT_KEY> AND status = 'To Do' ORDER BY created ASC"
$uri = "$base/search?jql=$([uri]::EscapeDataString($jql))&fields=summary,status,labels"
(Invoke-RestMethod -Uri $uri -Headers $headers -Method Get).issues
```

**Add a comment** (ADF body)
```powershell
$body = @{
    body = @{
        type    = 'doc'
        version = 1
        content = @(@{
            type    = 'paragraph'
            content = @(@{ type = 'text'; text = '<comment text>' })
        })
    }
} | ConvertTo-Json -Depth 10
Invoke-RestMethod -Uri "$base/issue/<ISSUE_KEY>/comment" -Headers $headers -Method Post -ContentType 'application/json' -Body $body
```

**Get transitions, then apply one**
```powershell
$t = (Invoke-RestMethod -Uri "$base/issue/<ISSUE_KEY>/transitions" -Headers $headers).transitions
$t | Select-Object id, name
$payload = @{ transition = @{ id = '<transition_id>' } } | ConvertTo-Json
Invoke-RestMethod -Uri "$base/issue/<ISSUE_KEY>/transitions" -Headers $headers -Method Post -ContentType 'application/json' -Body $payload
```

**Apply a label**
```powershell
$payload = @{ fields = @{ labels = @('<label>') } } | ConvertTo-Json
Invoke-RestMethod -Uri "$base/issue/<ISSUE_KEY>" -Headers $headers -Method Put -ContentType 'application/json' -Body $payload
```

**Create an issue**
```powershell
$payload = @{
    fields = @{
        project   = @{ key = '<PROJECT_KEY>' }
        summary   = '<title>'
        issuetype = @{ name = '<Story|Task|Bug>' }
        description = @{
            type    = 'doc'
            version = 1
            content = @(@{
                type    = 'paragraph'
                content = @(@{ type = 'text'; text = '<body>' })
            })
        }
    }
} | ConvertTo-Json -Depth 10
Invoke-RestMethod -Uri "$base/issue" -Headers $headers -Method Post -ContentType 'application/json' -Body $payload
```

**Create a sub-task under a parent**
```powershell
$payload = @{
    fields = @{
        project   = @{ key = '<PROJECT_KEY>' }
        summary   = '<title>'
        issuetype = @{ name = 'Subtask' }
        parent    = @{ key = '<PARENT_ISSUE_KEY>' }
    }
} | ConvertTo-Json -Depth 10
Invoke-RestMethod -Uri "$base/issue" -Headers $headers -Method Post -ContentType 'application/json' -Body $payload
```

## Issue links (Blocks / is blocked by)

Use real Jira issue links — NOT comments — to express dependencies between
issues. Comments are invisible to filtering, blocker queries, and Jira's
built-in link views.

### Mental model — direction is built into the link type

The `Blocks` link type has direction. Its two verbs:

- Outward verb: **"blocks"**
- Inward verb:  **"is blocked by"**

The payload field names map to those verbs, not to the issue you're "on":

- `outwardIssue` = the issue at the **"blocks"** end → **the BLOCKER** → the one that must complete first
- `inwardIssue`  = the issue at the **"is blocked by"** end → **the BLOCKED** → the one that has to wait

The trap to avoid: "Blocked by" sounds inward, so it's tempting to put the
blocker in `inwardIssue`. That's wrong. **`outwardIssue` is always the
blocker**, regardless of which side of the relationship you happen to be
reading from.

### Worked example

Suppose `WSCR-200` is foundational work and `WSCR-201` depends on it. The
`/to-issues` skill writes `"Blocked by: WSCR-200"` into WSCR-201's body. That
phrase has a single, mechanical translation:

| Slice                   | Role in link | Payload field           |
|-------------------------|--------------|-------------------------|
| WSCR-200 (the blocker)  | BLOCKER      | `outwardIssue.key`      |
| WSCR-201 (this slice)   | BLOCKED      | `inwardIssue.key`       |

Reads in Jira after creation:

- Link panel on **WSCR-200** shows: *"blocks WSCR-201"*
- Link panel on **WSCR-201** shows: *"is blocked by WSCR-200"*

### Preferred — MCP tool

Call `mcp__atlassian__createIssueLink` for each dependency. Use
`mcp__atlassian__getIssueLinkTypes` once if you need to confirm the available
type names. The MCP tool takes structured parameters, so it sidesteps Pitfall
1 below entirely.

### Fallback — PowerShell `Invoke-RestMethod`

```powershell
# WSCR-200 blocks WSCR-201 (from the worked example above).
$body = @{
    type         = @{ name = 'Blocks' }
    outwardIssue = @{ key = 'WSCR-200' }   # the BLOCKER — must complete first
    inwardIssue  = @{ key = 'WSCR-201' }   # the BLOCKED — has to wait
} | ConvertTo-Json -Depth 5
Invoke-RestMethod -Uri "$base/issueLink" -Headers $headers -Method Post -ContentType 'application/json' -Body $body
```

### Sanity check after creation

The cheapest way to verify direction is to GET the BLOCKED issue and confirm
its `issuelinks` entry points back at the BLOCKER:

```powershell
$blocked = Invoke-RestMethod -Uri ([Uri]"$base/issue/WSCR-201") -Headers $headers
$blocked.fields.issuelinks |
    Where-Object { $_.type.name -eq 'Blocks' -and $_.outwardIssue } |
    ForEach-Object { "WSCR-201 is blocked by $($_.outwardIssue.key)" }
# Expected output:
#   WSCR-201 is blocked by WSCR-200
```

If you see the keys swapped (e.g. *"WSCR-200 is blocked by WSCR-201"*) then
the create payload had `outwardIssue` and `inwardIssue` flipped — delete the
link and recreate with the values swapped.

### Pitfall 1 — query strings in PowerShell URIs (REST fallback only)

When verifying via `GET /rest/api/3/issue/<KEY>?fields=issuelinks`, pass the
URI as `[Uri]"..."` not a plain string. `Invoke-RestMethod` will URL-encode
the `?` in a plain string as `%3F`, so Jira reads the entire thing including
the query as the issue key and returns `404 — Issue does not exist or you do
not have permission to see it`.

```powershell
# WRONG — '?' is URL-encoded, Jira sees "DT-537%3Ffields=issuelinks" as the key
Invoke-RestMethod -Uri "$base/issue/$k?fields=issuelinks" -Headers $headers

# RIGHT — cast to [Uri] so PowerShell parses path + query separately
Invoke-RestMethod -Uri ([Uri]"$base/issue/$k?fields=issuelinks") -Headers $headers

# ALSO RIGHT — drop the query param, fetch all fields
Invoke-RestMethod -Uri "$base/issue/$k" -Headers $headers
```

This pitfall is REST-fallback-specific. The MCP atlassian tools take
structured parameters and never hit it.

### Pitfall 2 — link direction in the response is from the OTHER end (MCP and REST)

`fields.issuelinks[]` entries have EITHER an `inwardIssue` OR an
`outwardIssue` property — populated for whichever end the OTHER issue sits
on, NOT relative to the issue you queried.

For `Blocks` links:

- Querying the BLOCKER (e.g. `DT-537`) → entries show `inwardIssue = DT-538`
  (the blocked end is the "other" side from this issue's perspective)
- Querying the BLOCKED (e.g. `DT-538`) → entries show `outwardIssue = DT-537`
  (the blocker end is the "other" side)

So to enumerate **what blocks an issue you're looking at**, filter on entries
where `.outwardIssue` is set AND `type.name == "Blocks"`:

```powershell
$blockers = $issue.fields.issuelinks |
    Where-Object { $_.type.name -eq 'Blocks' -and $_.outwardIssue } |
    ForEach-Object { $_.outwardIssue.key }
```

Filtering on `.inwardIssue` returns the issues that THIS issue blocks — the
inverse direction. This pitfall applies regardless of whether you fetched
the issue via MCP or REST, because both return Jira's native response shape.

## Notes

- Prefer Story for new feature work, Task for technical tasks, Bug for defects.
- Status transitions are human decisions — the agent may propose a transition
  but must not apply it without explicit confirmation from the developer.
- When referencing a Jira issue in a commit message or PR description, use the
  full issue key (e.g. `WSCR-123`) so Azure DevOps and Jira can cross-link.
- The Jira description field uses Atlassian Document Format (ADF) JSON — use
  the structure shown above, not plain text strings.
- For Azure DevOps PR linkage, use `mcp__atlassian__getJiraIssueRemoteIssueLinks`
  on the Jira side; parse `pullrequest/(\d+)` out of the returned URLs to get
  AzDO PR IDs.
