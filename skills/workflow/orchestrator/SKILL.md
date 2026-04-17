# Orchestrator Agent

## Purpose

Single control layer for the full agentic development workflow. Accepts a trigger, clones the project repo, invokes each agent skill in order, tracks state through the run folder, and handles failures and escalations. Every development task goes through this agent.

## When to Use

- At the start of any development task
- This is the entry point to the entire workflow

## Trigger Input

Accept any of:
- Jira ticket ID or URL (`PROJ-123`)
- Free-text feature description with repo name
- One-line task description

## Step 0 — Pre-Flight

Before invoking any agent:

1. Check for an existing interrupted run for this user session:
   ```
   <agentic-flow-repo>/.agentic/runs/.current-run-<username>
   ```
   Where `<username>` is derived from `git config user.name` (slugified) or `$USER`.
   If this file exists and contains a valid run folder path, ask the user:
   > "A previous run was found at `<run-folder>`. Resume it or start fresh?"
   - Resume → skip to **Resuming an Interrupted Run**
   - Fresh → proceed with Step 0b

2. Only one active run per user session at a time. If `.current-run-<username>` exists and the user chooses fresh, archive the old run folder path to a `.current-run-<username>.bak` file before overwriting.

## Step 0b — Clone Repository

Invoke `skills/clone-agent/SKILL.md`.

The Clone Agent:
- Finds the project config in `projects/<PROJECT_KEY>/config.yaml`
- Clones the repo (or fetches if already cloned)
- Runs setup commands
- Returns `clone_path`, `platform`, `repo_slug`, `test_command`

Store the returned values — they are used by all subsequent agents as `<project-repo>`.

The `agentic-flow` repo absolute path is also now known. Use it for all script invocations:
```
AGENTIC_FLOW_DIR=<absolute path of this skill's parent directory>
```

## Token Usage Logging

After every stage completes, run:

```bash
python3 ${AGENTIC_FLOW_DIR}/scripts/log_token_usage.py \
  --stage <stage-name> \
  --run-folder <run-folder> \
  --project-dir ${AGENTIC_FLOW_DIR}
```

This is non-blocking — if it fails, log the error and continue. Stage names to use:

| Stage | `--stage` value |
|---|---|
| Clone | `clone` |
| GitNexus Index | `gitnexus-index` |
| Requirement Collection | `requirement` |
| Planner | `planner` |
| Branch creation | `branch` |
| Coder | `coder` |
| Code Review | `review` |
| Documentation | `documentation` |
| PR Creation | `pr` |
| Deploy to Test | `deploy` |
| Jira Update | `jira` |

Output is written to:
- `<run-folder>/token-usage.json` — per-stage + running totals (input, cache_read, cache_create, output)
- `<run-folder>/logs/token-usage.log` — human-readable one line per stage

**Important:** call this script using `--project-dir ${AGENTIC_FLOW_DIR}` (not the cloned project repo) — that is where the active Claude Code session file lives.

## Step 0c — GitNexus Index

Run after the Clone Agent completes, before Requirement Collection.

Before running: update `run-state.json` → `current_stage: "gitnexus-index"`.

```bash
cd <clone_path> && gitnexus analyze
```

On exit code 0: update `run-state.json` with `gitnexus_available: true` and `gitnexus_indexed_at: <UTC ISO 8601>`.

On non-zero exit code or any error:
- Log the error to `<project-repo>/.agentic/runs/<run-folder>/logs/gitnexus.log`
- Set `gitnexus_available: false` in `run-state.json`
- Continue — this stage is non-blocking

## Workflow

```
Trigger
  │
  ▼
[0] Pre-Flight (check .current-run-<username>)
  │
  ▼
[0b] Clone Agent (clone or fetch repo)
  │
  ▼
[0c] GitNexus Index
  │   Run: gitnexus analyze (in clone_path)
  │   On success: gitnexus_available: true in run-state.json
  │   On failure: warn + log, gitnexus_available: false, continue
  │
  ▼
[1] Requirement Collection Agent
  │   → on user 'yes': write run folder path to .current-run-<username>
  │
  ├─ complexity: low ──────────────────────────────────────────────┐
  │                                                                │
  ▼                                                                │
[2] Planner Agent                                                  │
  │  (human approval mandatory for medium/high)                   │
  │                                                                │
  ▼                                                                ▼
[2a] On-Demand Repo Clone  (if plan.additional_repos_needed is non-empty)
  │
  ▼
[2b] Create Feature Branch  ◄────────────────────────────────────-┘
  │   (both paths create a branch before coding)
  │
  ▼
[3] Coder Agent
  │  (implements + tests per subtask)
  │
  ▼
[4] Code Review Agent
  │  (correctness + quality + security + observability)
  │
  ├─ blocked ──► Coder Agent (fix) ──► Code Review (re-review)
  │              max 5 rounds, same issue 3x → discard + restart
  │
  ├─ plan broken ──► Planner Agent (human re-approval) ──► [2b] ──► Coder
  │
  ▼
[5] Documentation Agent
  │
  ▼
[6] PR Creation Agent
  │
  ▼
[6b] Deploy to Test  (optional — only if deploy_to_test: true)
  │
  ▼
[7] Jira Status Update  (if jira_ticket_id exists)
  │
  ▼
[8] Run Completion
```

## Stage Invocation

**Inline execution only.** Each stage skill is loaded with the `Read` tool and executed by the Orchestrator itself — not delegated to a subagent. Do NOT use the `Agent` tool for any main pipeline stage (clone, requirement, planner, coder, review, documentation, PR, deploy, Jira).

**Superpowers skill override:** During Stage [4] Code Review, load ONLY `skills/code-review-agent/SKILL.md`. Do NOT invoke `superpowers:requesting-code-review` — the pipeline has its own dedicated review stage that supersedes it.

The `Agent` tool is reserved exclusively for fix-iteration loops (Test-Fix Loop, Code Review fix loop). Always invoke fix-loop agents with `model: sonnet` to reduce cost — fix iterations require solid coding ability but do not need the full session model.

To execute a stage:
1. Use the `Read` tool to load the skill file path listed in the table below.
2. Follow all instructions in that skill file as the orchestrator.
3. The skill returns control when it writes its output JSON to the run folder.
4. Validate the output (see State Tracking table), then proceed to the next stage.

The Orchestrator never hands off permanently — it always resumes control after each stage.

| Stage | Skill |
|---|---|
| Clone | `skills/clone-agent/SKILL.md` |
| GitNexus Index | `gitnexus analyze` (direct command — no skill file) |
| Requirement Collection | `skills/requirement-collection/SKILL.md` |
| Planner | `skills/planner-agent/SKILL.md` |
| On-Demand Clone | `skills/clone-agent/SKILL.md` (targeted repo only) |
| Coder | `skills/coder-agent/SKILL.md` |
| Code Review | `skills/code-review-agent/SKILL.md` |
| Documentation | `skills/documentation-agent/SKILL.md` |
| PR Creation | `skills/pr-creation-agent/SKILL.md` |
| Deploy to Test (optional) | `skills/deploy-to-vercel/SKILL.md` |

Always pass the run folder path and `clone_path` when invoking each agent.

### Parallel Subtask Execution

When `plan.json` contains subtasks with `execution: parallel` and no unmet `depends_on` entries, dispatch them concurrently using multiple Agent tool calls in a single message — one Agent invocation per parallel subtask group. Each agent receives:
- Run folder path
- The specific subtask ID(s) it should implement
- Instruction: "Implement only the subtask(s) listed. Do not implement other subtasks."

Wait for all parallel agents to return before validating `implementation.json` and proceeding to Code Review. If any parallel agent fails its tests, trigger the Test-Fix Loop for that subtask only.

## Stage 2a — On-Demand Repo Clone

Run immediately after plan approval, before branch creation.

Read `additional_repos_needed` from `plan.json`. For each repo listed:
1. Invoke `skills/clone-agent/SKILL.md` for that repo only — pass the repo name so the clone agent targets it directly without asking the user
2. Add the new `clone_path` to the existing `clone_paths` map in `.clone-state`
3. Update `run-state.json` with the new path

If `additional_repos_needed` is empty, skip this stage entirely.

## Stage 2b — Create Feature Branch

Run after planning is confirmed **and** after requirement collection for low-complexity tasks. Both paths go through branch creation before the Coder runs.

**Low-complexity G4A stub:** For low-complexity tasks (where the Planner is skipped), write a minimal `planner-reasoning.json` to preserve the audit trail:

```json
{
  "schema_version": "1.0",
  "agent": "planner",
  "run_folder": "<run-folder-name>",
  "task": "Plan: <requirement title>",
  "started_at": "<UTC ISO 8601>",
  "completed_at": "<UTC ISO 8601>",
  "context": { "trigger": "skipped — low-complexity task", "files_read": [], "key_symbols": [] },
  "reasoning": {
    "hypothesis": "Low complexity — single change, no decomposition required",
    "approach": "Planner skipped per complexity rules",
    "alternatives_considered": [],
    "confidence": 1.0
  },
  "outcome": { "status": "skipped", "summary": "Planner not invoked for low-complexity task." }
}
```

Save to `<project-repo>/.agentic/runs/<run-folder>/g4a/planner-reasoning.json`.

```bash
# Sanitize title: lowercase, replace anything not alphanumeric/hyphen with hyphen, collapse multiple hyphens, strip leading/trailing hyphens
SAFE_TITLE=$(echo "<title>" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-//;s/-$//')

# Branch naming
# With Jira ticket:
git checkout -b feature/PROJ-123-${SAFE_TITLE}
# Without Jira ticket:
git checkout -b feature/${SAFE_TITLE}-$(date +%Y-%m-%d)
```

If the branch already exists (resumed run):
```bash
git checkout feature/<branch-name>
git pull origin feature/<branch-name> 2>/dev/null || true
```

Record the branch name in `run-state.json`.

## Token Usage in run-state.json

After each token logging call, merge the latest totals from `token-usage.json` into `run-state.json`:

```json
{
  "token_usage": {
    "input_tokens": 12800,
    "cache_read_input_tokens": 980000,
    "cache_creation_input_tokens": 4200,
    "output_tokens": 3330,
    "total_tokens": 20330
  }
}
```

`total_tokens` = `input_tokens` + `cache_creation_input_tokens` + `output_tokens` (cache reads are not billed as new tokens).

## State Tracking

The Orchestrator validates each stage output before proceeding:

| Stage | Expected output | Validation |
|---|---|---|
| Clone | `.clone-state` in agentic-flow | exists + valid JSON |
| GitNexus Index | — | `gitnexus_available` present in `run-state.json` (any value — non-blocking) |
| Requirement Collection | `requirement.json` | exists + valid JSON + `schema_version` present |
| Planner | `plan.json` | exists + valid JSON + `schema_version` present + `design` key present |
| Branch creation | — | `git branch --show-current` matches expected branch |
| Coder | `implementation.json` | exists + valid JSON + `all_tests_pass: true` |
| Coder (G4A) | `g4a/coder-reasoning.json` | exists (warn if missing, do not block) |
| Code Review | `review.json` | exists + valid JSON + `result: approved` |
| Code Review (G4A) | `g4a/review-reasoning.json` | exists (warn if missing, do not block) |
| Documentation | `documentation.json` | exists + valid JSON |
| PR Creation | `pr.json` | exists + valid JSON + `pr_url` not empty |
| Deploy to Test | `deploy.json` | exists + valid JSON + `deploy_url` not empty (only if triggered) |

> **Note:** If Coder validation fails (`all_tests_pass` not true), the orchestrator does NOT stop — it triggers the Test-Fix Loop above. `<project-repo>/.agentic/runs/<run-folder>/test-fix-state.json` tracks coder fix rounds separately from `fix-state.json` (which tracks code review fix rounds).

If validation fails after a stage, handle as follows:

- **Coder (`all_tests_pass` is false or missing):** Do NOT stop. Trigger the **Test-Fix Loop** (see below).
- **All other stages:** Write to `logs/error.log`, stop, tell the user which stage failed and why.

## Human Gates

| Gate | When | What to wait for |
|---|---|---|
| Requirement confirmation | After Requirement Collection (always) | User replies 'yes' |
| Plan approval | After Planner (medium/high only) | User replies 'yes' |
| Escalation | After any iteration limit is hit | User guidance before resuming |

## Test-Fix Loop (Stage 3 → Coder)

When `implementation.json` is missing or `all_tests_pass` is not `true` after the Coder returns:

1. Read the failing test output from `logs/coder.log` (the Coder appends failure details there after each failed run).
2. Read or create `<project-repo>/.agentic/runs/<run-folder>/test-fix-state.json`:

```json
{
  "total_rounds": 0,
  "rounds": []
}
```

3. Increment `total_rounds`.
3a. Append a new entry to `rounds`: `{"round": <total_rounds>, "failing_tests": [<failing test names from coder.log>], "resolved": false}`
4. Invoke the **Agent tool** with `model: sonnet` to run `skills/coder-agent/SKILL.md`, passing:
   - The run folder path
   - The list of failing tests from this round
   - Instruction: "Fix only the failing tests listed. Do not rewrite passing code. **Disable your internal fix loop (Step 5 of Per-Subtask Workflow) — attempt once and report results.** The Orchestrator manages all retry iteration; internal retries will double-count attempts."

5. After the Coder returns, check `implementation.json` again:
   - If `all_tests_pass: true` → mark this round `resolved: true`, continue to Code Review.
   - If still failing → repeat from step 2.

6. If `total_rounds` reaches **5** without `all_tests_pass: true`: write `status: paused` to `run-state.json`, escalate to the user with the full failure history from `test-fix-state.json`. Do not proceed to Code Review.

7. If the exact same failing test appears in **3 consecutive rounds**: discard `implementation.json` (archive to `logs/implementation-discarded-test-round-N.json`), overwrite `test-fix-state.json` with `{"total_rounds": 0, "rounds": []}`, restart the Coder from scratch with no pre-loaded failure context.

## Fix-Round Counter

Persisted in the run folder:

```
<project-repo>/.agentic/runs/<run-folder>/fix-state.json
```

```json
{
  "total_fix_rounds": 0,
  "rounds": [
    {
      "round": 1,
      "issues": [
        {
          "id": "<normalized issue ID — e.g. security-login.py-SQL_INJECTION>",
          "description": "<human-readable description>",
          "severity": "blocking",
          "file": "<file path>"
        }
      ],
      "resolved": false
    }
  ]
}
```

- **Increment `total_fix_rounds`** each time the Coder is invoked for a review fix
- The Code Review agent reads this file and uses `id` (not description) to detect the same issue across rounds
- If `total_fix_rounds` reaches 5: escalate immediately
- If the same issue `id` appears in 3 consecutive rounds: discard implementation, reset `fix-state.json`, **archive** `implementation.json` to `logs/implementation-discarded-round-N.json`, restart Coder from scratch

## run-state.json

Written by the **Orchestrator** at every stage transition — not by individual agents.

```
<project-repo>/.agentic/runs/<run-folder>/run-state.json
```

```json
{
  "status": "running|paused|complete|failed",
  "current_stage": "<stage name>",
  "paused_reason": "<why it was paused, or null>",
  "branch": "<feature branch name>",
  "last_updated": "<UTC ISO 8601 timestamp>",
  "gitnexus_available": true,
  "gitnexus_indexed_at": "<UTC ISO 8601 timestamp or null>"
}
```

Update `run-state.json` before invoking each stage and immediately after each stage completes.

## Failure Handling

| Situation | Action |
|---|---|
| Stage output missing or invalid JSON | Write to `logs/error.log`, stop, tell user which stage failed |
| Clone fails | Stop run — cannot proceed without a repo |
| GitNexus analyze fails | Log to `<run-folder>/logs/gitnexus.log`, set `gitnexus_available: false`, continue |
| Script fails (Jira/Figma/etc.) | Agent falls back to manual — orchestrator continues |
| Test-fix loop exhausted (5 rounds) | Write `status: paused` to `run-state.json`, escalate to user with full `test-fix-state.json` failure history |
| Code review fix loop exhausted (5 rounds) | Write paused state, escalate to human |
| Same failing test 3x consecutive (test-fix loop) | Archive `implementation.json` to `logs/implementation-discarded-test-round-N.json`, reset `test-fix-state.json`, restart Coder |
| Same issue `id` 3x (code review fix loop) | Archive `implementation.json` to `logs/implementation-discarded-round-N.json`, reset `fix-state.json`, restart Coder |
| Plan is fundamentally broken | Write paused state, return to Planner with human re-approval |
| Requirement revision limit (3 rounds) | Write paused state, escalate to human |
| Unrecoverable error | Write to `logs/error.log`, update `run-state.json` → failed, run cleanup (see below), stop |

## Cleanup on Failure

When a run ends in `status: failed` (unrecoverable error or iteration limit exhausted without recovery):

1. Stash any uncommitted changes in the project repo:
   ```bash
   git -C <clone_path> stash push -m "agentic-run-<run-folder>-failed"
   ```
2. Report to the user:
   - Which stage failed and why (from `logs/error.log`)
   - The stash ref so they can restore or discard: `git stash list`
   - The run folder path for the full audit trail
3. Clear `.current-run-<username>` (delete the file) so future runs start clean
4. Do NOT delete the feature branch — the user may want to inspect or continue manually

## Resuming an Interrupted Run

If `.current-run-<username>` exists:

1. Read the run folder path
2. Read `run-state.json` to understand last known state
3. Validate each stage's output file (exists + valid JSON + `schema_version`)
4. Resume from the first stage whose output is missing or invalid
5. Check out the branch recorded in `run-state.json` before resuming
6. Do not re-run stages with valid output files

## Iteration Limits (Global)

- Test-fix loop (coder): max 5 rounds
- Code review fix loop: max 5 rounds
- Same failing test 3x consecutive (test-fix loop): discard `implementation.json`, reset `test-fix-state.json`, restart Coder
- Same issue `id` 3x consecutive (code review fix loop): discard `implementation.json`, reset `fix-state.json`, restart Coder
- Plan revisions: max 3 rounds
- Requirement revisions: max 3 rounds

## Stage 6b — Deploy to Test Environment (Optional)

This stage runs **only if** `requirement.json` contains `"deploy_to_test": true` OR `plan.json` contains `"deploy_to_test": true`. If neither is set, skip to Stage 7.

### When to Set `deploy_to_test`

The Requirement Collection agent sets `deploy_to_test: true` in `requirement.json` when the user's task description explicitly requests a test/preview deployment (e.g., "deploy for QA", "deploy to staging", "I need a preview link"). The Planner agent may override this in `plan.json` if architectural analysis determines a deployment is needed for integration testing.

### Stack Check

Before loading the deploy skill, verify the project is a Vercel-compatible frontend:

```bash
ls <clone_path>/vercel.json <clone_path>/next.config.js <clone_path>/next.config.ts 2>/dev/null | head -1
```

If none of those files exist, also check for `package.json` with a `vercel` script or `"framework": "nextjs"` in `.vercel/project.json`.

If no Vercel indicators are found: skip this stage, write `deploy.json` with `"deploy_url": null` and `"skipped_reason": "not a Vercel project"`, continue to Stage 7.

### Invocation

Load `skills/deploy-to-vercel/SKILL.md` and run it against the project repo (`clone_path`). The deploy-to-vercel skill handles Vercel CLI detection, linking, and deployment automatically.

For the agentic context, always target a **preview deployment** (never production). The deploy-to-vercel skill defaults to preview — do not pass `--prod`.

After the skill completes, capture the deploy URL from its output and write:

```
<project-repo>/.agentic/runs/<run-folder>/deploy.json
```

```json
{
  "schema_version": "1.0",
  "deploy_url": "https://my-app-abc123.vercel.app",
  "deployed_at": "<UTC ISO 8601 timestamp>",
  "environment": "preview"
}
```

### PR and Jira Integration

- Append the deploy URL to the PR description:
  ```
  **Test Deployment:** https://my-app-abc123.vercel.app
  ```
- Include the deploy URL in the Jira comment (Stage 7) alongside the PR URL:
  ```
  PR created: <pr_url>
  Test deployment: <deploy_url>
  ```

### Failure Handling

If the deploy fails (CLI not installed, auth missing, network error):
- Log to `logs/error.log`
- Write a partial `deploy.json` with `"deploy_url": null` and `"error": "<reason>"`
- Continue to Stage 7 — a failed test deployment must not block the PR or Jira update
- Tell the user the deployment failed and what to do next (e.g., run `vercel deploy` manually)

## Stage 7 — Jira Status Update

If `requirement.json` contains a non-null `jira_ticket_id`:

```bash
# If deploy.json exists and deploy_url is not null:
COMMENT="PR created: <pr_url>\nTest deployment: <deploy_url>"

# Otherwise:
COMMENT="PR created: <pr_url>"

python ${AGENTIC_FLOW_DIR}/scripts/update_jira.py \
  --ticket <jira_ticket_id> \
  --status "In Review" \
  --comment "$COMMENT"
```

`AGENTIC_FLOW_DIR` is the absolute path to the agentic-flow repo (resolved in Step 0b). Never use relative paths.

If this script fails: log to `logs/error.log` and continue — Jira update failure must not block run completion.

## Stage 8 — Run Completion

Once `pr.json` is confirmed valid:

1. Update `run-state.json` → `status: complete`
2. Write:

```
<project-repo>/.agentic/runs/<run-folder>/run-summary.json
```

```json
{
  "schema_version": "1.0",
  "run_folder": "<run-folder>",
  "trigger": "<original trigger text>",
  "jira_ticket_id": "<or null>",
  "complexity": "low|medium|high",
  "completed_at": "<UTC ISO 8601>",
  "pr_url": "<PR URL>",
  "deploy_url": "<preview URL or null if not deployed>",
  "branch": "<feature branch name>",
  "clone_path": "<absolute path to project repo>",
  "stages_completed": ["clone", "gitnexus-index", "requirement", "coder", "review", "documentation", "pr"],
  "stages_skipped": ["planner", "deploy"],
  "fix_rounds_used": 0,
  "total_subtasks": 0,
  "token_usage": {
    "input_tokens": 0,
    "cache_read_input_tokens": 0,
    "cache_creation_input_tokens": 0,
    "output_tokens": 0,
    "total_tokens": 0
  }
}
```

Note: `stages_completed` lists only stages that actually ran. `stages_skipped` lists stages bypassed by design (e.g. `planner` for low-complexity tasks).

3. Clear `.current-run-<username>` (delete the file)
4. Report to the user:
   - PR URL
   - Deploy URL (if `deploy.json` exists and `deploy_url` is not null)
   - Jira ticket updated to "In Review" (or skipped if no ticket)
   - What was built (one-line summary)
   - Reminder: **manual functional validation is mandatory before merging**
   - Minimum validation: run the feature in a staging or local environment (use deploy URL if available), verify primary acceptance criteria manually, leave a comment on the PR confirming it was tested
