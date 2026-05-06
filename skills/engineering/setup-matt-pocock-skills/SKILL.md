# setup-matt-pocock-skills

Sets up an `## Agent skills` block in AGENTS.md/CLAUDE.md and `docs/agents/` so
the engineering skills know this repo's issue tracker (GitHub, Jira, or local
markdown), triage label vocabulary, and domain doc layout.

Run before first use of `to-issues`, `to-prd`, `triage`, `diagnose`, `tdd`,
`improve-codebase-architecture`, or `zoom-out` — or if those skills appear to be
missing context about the issue tracker, triage labels, or domain docs.

---

## What this skill configures

- **Issue tracker** — where issues live (Jira by default for Smart Insider repos;
  GitHub and local markdown also supported)
- **Triage labels** — the strings used for the five canonical triage roles
- **Domain docs** — where CONTEXT.md and ADRs live, and the consumer rules for
  reading them
- **Test runner** — how to run tests in this repo (`dotnet test` for C# projects)

---

## Instructions

This is a prompt-driven skill, not a deterministic script. Explore, present what
you found, confirm with the user, then write.

### Step 1 — Explore

Look at the current repo to understand its starting state:

- AGENTS.md and CLAUDE.md at the repo root — does either exist? Is there already
  an `## Agent skills` section in either?
- `docs/agents/` — does it exist? What files are in it?
- Project file — is there a `.csproj`, `.sln`, `package.json`, or other build
  descriptor? What language/stack is this repo?
- Test projects — is there an `*.Tests.csproj` or similar? What test framework
  (xUnit, NUnit, MSTest)?

Summarise what's present and what's missing. Then walk the user through the four
decisions one at a time — present a section, get the user's answer, then move to
the next. Don't dump all four at once. Assume the user does not know what these
terms mean.

Each section starts with a short explainer (what it is, why these skills need it,
what changes if they pick differently). Then show the choices and the default.

---

### Section A — Issue tracker

**Explainer:** The "issue tracker" is where issues live for this repo. Skills like
`to-issues`, `triage`, `to-prd`, and `qa` read from and write to it — they need
to know whether to call the Jira REST API, call `gh issue create`, write a
markdown file under `.scratch/`, or follow some other workflow. Pick the place you
actually track work for this repo.

**Choices:**

1. **Jira** *(default for Smart Insider repos)* — issues are created and updated
   via the Jira REST API. Requires `JIRA_URL`, `JIRA_EMAIL`, and `JIRA_PAT`
   environment variables. Read the bundled `issue-tracker-jira.md` for the full
   config block to write.

2. **GitHub Issues** — issues live in this repo's GitHub Issues tab. Requires the
   `gh` CLI to be authenticated. Read the bundled `issue-tracker-github.md` for
   the full config block to write.

3. **Local files** — issues are markdown files under `.scratch/issues/`. No
   external credentials needed. Good for repos not yet connected to a tracker.
   Read the bundled `issue-tracker-local.md` for the full config block to write.

4. **Other** — ask the user to describe the tracker and write a custom config
   block.

---

### Section B — Triage labels

**Explainer:** These are the label strings your issue tracker actually uses for the
five canonical triage roles. The `/triage` skill applies these labels — if the
strings are wrong the labels land on the wrong issues.

The five canonical roles and their default Jira equivalents:

| Role | Jira default | GitHub default |
|---|---|---|
| needs-triage | `Needs Triage` | `needs-triage` |
| needs-info | `Needs Info` | `needs-info` |
| ready-for-agent | `Ready for Agent` | `ready-for-agent` |
| ready-for-human | `Ready for Human` | `ready-for-human` |
| wontfix | `Won't Fix` | `wontfix` |

Ask the user to confirm or override each label string for their chosen tracker.
Read the bundled `triage-labels.md` for the file format to write to
`docs/agents/triage-labels.md`.

---

### Section C — Domain docs

**Explainer:** A shared language document (`CONTEXT.md`) and Architecture Decision
Records (`docs/adr/`) help the agent use the right terminology for this codebase
and respect past decisions. This section tells the agent where those files live
and how to use them.

Ask:
- Where does CONTEXT.md live? (default: repo root)
- Where do ADRs live? (default: `docs/adr/`)

Read the bundled `domain.md` for the file format to write to
`docs/agents/domain.md`.

---

### Section D — Test runner

**Explainer:** The `/tdd` skill runs tests in a red-green-refactor loop. It needs
to know how to invoke tests in this repo so it can verify each change compiles
and passes.

**Choices:**

1. **dotnet test** *(default for C# repos)* — runs all test projects in the
   solution. If a specific test project exists, use
   `dotnet test path/to/Project.Tests.csproj`. Write the test command to the
   Agent skills block.

2. **npm test / vitest / jest** — for TypeScript/JavaScript repos.

3. **Other** — ask the user to provide the exact command.

Detect the stack from the files found in Step 1 and pre-select the most likely
option before asking for confirmation.

---

### Section E — Environment variables

**Explainer:** The Jira-connected skills (grill-with-docs, to-prd, to-issues)
call the Jira REST API using credentials stored as environment variables. These
must never be committed to the repo — they live in a local .env file that is
gitignored, or in your shell profile.

Check whether a .env file already exists at the repo root:
- If it exists, check whether JIRA_URL, JIRA_EMAIL, and JIRA_PAT are already
  present. If they are, confirm with the user and skip.
- If it doesn't exist, create it with the three variables below.

Ask the user for:
1. Their Jira base URL (e.g. https://smartinsider.atlassian.net)
2. Their Jira email address
3. Their Jira API token (remind them to generate one at
   https://id.atlassian.com/manage-profile/security/api-tokens if they don't
   have one)

Write the .env file:
---

### Step 2 — Write

Once all four sections are confirmed, write the following:

**In AGENTS.md or CLAUDE.md** (edit the one that already exists; create
CLAUDE.md if neither exists — never create both):

```markdown
## Agent skills

### Issue tracker
<content from the relevant issue-tracker-*.md, filled in with user's choices>

### Triage labels
<content from triage-labels.md, filled in with user's label strings>

### Domain docs
<content from domain.md, filled in with user's paths>

### Test runner
Test command: `<command confirmed in Section D>`
Test framework: <xUnit | NUnit | MSTest | Jest | other>
```

**In docs/agents/:**
- `issue-tracker.md` — copy of the issue tracker config block
- `triage-labels.md` — the five role → label-string mappings
- `domain.md` — domain doc locations and consumer rules

---

## Guardrails

- Never create AGENTS.md when CLAUDE.md already exists (or vice versa) — always
  edit the one that's already there.
- If an `## Agent skills` block already exists in the chosen file, update its
  contents in-place rather than appending a duplicate.
- Don't overwrite user edits to the surrounding sections.
- Never write secrets (PAT tokens, passwords) into any committed file. Always
  reference environment variables by name only (e.g. `$JIRA_PAT`).
- Never read or display the contents of .env
- If .env is accidentally staged for commit, warn the user immediately and
  unstage it with `git reset HEAD .env`
- If the repo has no test project yet, write a placeholder test command and note
  that it should be updated when a test project is added.
