# PR Creation Agent

## Purpose

Package the completed, reviewed, and documented work into a pull request. Produces a consistent, informative PR that gives reviewers everything they need without opening any files.

## When to Use

- After the Documentation agent has saved `documentation.json`

## Input

Read from the run folder:

```
<project-repo>/.agentic/runs/<run-folder>/requirement.json
<project-repo>/.agentic/runs/<run-folder>/plan.json
<project-repo>/.agentic/runs/<run-folder>/implementation.json
<project-repo>/.agentic/runs/<run-folder>/review.json
<project-repo>/.agentic/runs/<run-folder>/documentation.json
```

## Step 1 — Push Branch to Remote

Before creating the PR, push the feature branch to the remote:

```bash
git push -u origin <current-branch>
```

If push fails (auth, remote not reachable, etc.), stop and tell the user — do not proceed to PR creation without a pushed branch.

## Step 2 — Detect Platform and Default Branch


```bash
git remote get-url origin
```

- URL contains `github.com` → platform: GitHub (`gh` CLI)
- URL contains `bitbucket.org` → platform: Bitbucket (helper script)

Detect the repository's default branch — do not assume `main`:

```bash
git remote show origin | grep "HEAD branch" | awk '{print $NF}'
```

Use this as `destination_branch`. Fall back to `main` only if detection fails.

## Step 3 — Verify Branch State

Before creating the PR:

```bash
git status          # ensure no uncommitted changes
git log origin/main..HEAD --oneline   # confirm commits exist on this branch
```

If there are uncommitted changes, stop and tell the user to commit them first.

## Step 4 — Build PR Description

Construct the PR description from the run folder data. Keep it clear and scannable — the reviewer should not need to open any file.

```markdown
## What this changes
<2-3 sentences from requirement.json final_requirement>

## Why
<context from requirement.json — Jira ticket link if available, business reason>

## What was done
<if plan.json exists: bullet list of subtasks from plan.json, one line each>
<if plan.json does not exist (low-complexity): derive from implementation.json subtasks or from requirement.json final_requirement>

## Tests
<summary of tests added from implementation.json — what was tested, pass/fail>

## Review notes
<known_risks from implementation.json, important_notes from review.json>

## Docs updated
<files updated from documentation.json>

## Checklist
- [x] Tests written and passing
- [x] Code review passed
- [x] Security review passed
- [x] Observability review passed
- [x] Documentation updated
- [ ] Manual functional validation pending
```

## Step 5 — Create PR

### GitHub
```bash
gh pr create \
  --title "<ticket-id>: <short requirement title>" \
  --body "<description from Step 3>" \
  --base main \
  --head <current-branch>
```

### Bitbucket
```bash
python /path/to/agentic-flow/scripts/create_pr_bitbucket.py \
  --repo <repo-slug> \
  --title "<ticket-id>: <short requirement title>" \
  --source <current-branch> \
  --dest main \
  --description "<description from Step 3>"
```

**PR title format:** `PROJ-123: Add login button` — ticket ID prefix always, short description, sentence case.
If no Jira ticket: `feat: <short description>` or `fix: <short description>`.

## Step 6 — Save Output

```
<project-repo>/.agentic/runs/<run-folder>/pr.json
```

```json
{
  "schema_version": "1.0",
  "run_folder": "<run-folder>",
  "created_at": "<UTC ISO 8601 timestamp>",
  "platform": "github|bitbucket",
  "pr_id": "<PR number>",
  "pr_url": "<full URL to the PR>",
  "title": "<PR title>",
  "source_branch": "<branch name>",
  "destination_branch": "<detected default branch>",
  "jira_ticket_id": "<from requirement.json or null>"
}
```

Append a summary to `.agentic/runs/<run-folder>/logs/pr.log`.

## Handoff

Once the PR is created:

1. Share the PR URL with the user
2. Remind the user that **manual functional validation is mandatory** before merging — this is a non-negotiable step that cannot be automated
3. The run is complete. The `.agentic/runs/<run-folder>/` folder now contains the full audit trail:

```
requirement.json      ← what was asked
plan.json             ← how it was planned
implementation.json   ← what was built and tested
review.json           ← what was reviewed
documentation.json    ← what was documented
pr.json               ← the PR created
logs/                 ← step-by-step logs
```
