# Clone Agent

## Purpose

Clone the project repository and set up the working environment before any development work begins. Uses a project-specific config to know which repo to clone, which branch to start from, and what setup steps are needed.

## When to Use

- At the very start of a run, before Requirement Collection
- Invoked by the Orchestrator as Stage 0b
- Skip if the repo is already cloned and the working directory is already set to the correct repo

## Project Config

Each project has a config file in the agentic-flow repo:

```
<agentic-flow-repo>/projects/<project-name>/config.yaml
```

If a Jira ticket ID is available, match the ticket's project key to find the config:
- `PROJ-123` → look for `projects/PROJ/config.yaml`
- If no match found: ask the user for the repo URL and branch directly

### Config format

```yaml
project_key: PROJ          # Jira project key prefix (optional)
repo_url: git@github.com:org/repo.git   # or HTTPS URL
default_branch: main       # branch to clone from
jira_project_key: PROJ     # matches Jira ticket prefix
setup_commands:            # run after clone, in order
  - npm install
  - cp .env.example .env
test_command: npm test     # how to run tests (used by Coder agent)
platform: github           # github | bitbucket
repo_slug: repo-name       # used by Bitbucket PR script
description: "Short description of what this project is"
```

## Step 1 — Find Project Config

1. If Jira ticket ID is known, extract the project key (e.g. `PROJ` from `PROJ-123`)
2. Look for `projects/<PROJECT_KEY>/config.yaml` in the agentic-flow repo
3. If found: use the config — do not ask the user for repo details
4. If not found: ask the user:
   > "I don't have a config for this project yet. What is the repository URL and which branch should I start from?"
   > Record the answer and offer to create a config file for future runs.

## Multi-Repo Projects

If `config.yaml` contains `type: multi_repo`, follow this section instead of Steps 2–4.

### 1. Infer repos from the task description

Build a single text blob from all available input — concatenate (lowercased):
- The raw trigger text (ticket ID title, free-text description, or both)
- If a Jira ticket ID is present, also fetch it and append its `title`, `description`, `labels`, and `components`

Match this text blob against each repo's `signals` list in `config.yaml`. A repo is selected if **any** of its signals appear anywhere in the text.

**Decision rules:**
- One or more repos matched → use those; do not ask the user
- No signals matched → ask the user: "Which repos does this task touch? (all / backend / monorepo / admin-dashboard)"

Only clone the selected repos. Skip the rest entirely — do not clone, pull, or run setup commands for unselected repos.

**Then ask task type** (always required):
> "Is this a **normal** task or a **hotfix**?"

Use the answer to select the branch for each repo (from the `branch.normal` or `branch.hotfix` field).

### 2. Clone or pull each repo

For each **selected** repo entry in `repos`, repeat this pattern:

```bash
AGENTIC_FLOW_DIR=<absolute path to agentic-flow repo>
TICKET=<jira_ticket_id or slugified task title>
REPO_PATH=$AGENTIC_FLOW_DIR/workspace/$TICKET/<repo-name>

if [ -d "$REPO_PATH" ]; then
  cd $REPO_PATH && git pull && cd -
else
  git clone -b <branch> <repo-url> $REPO_PATH
fi
```

### 3. Run setup commands

After cloning each repo, run its `setup_commands` in order inside its directory. If any fail, log to `logs/clone.log` and ask the user whether to continue.

### 4. Save output

Write to `.agentic/runs/.clone-state-<task-slug>` in the agentic-flow repo. `<task-slug>` is the Jira ticket ID or slugified task title. Store an array:

```json
{
  "type": "multi_repo",
  "task_type": "normal|hotfix",
  "ticket": "<ticket id or slug>",
  "repos": [
    {
      "name": "auditor-app-monorepo",
      "clone_path": "<absolute path>",
      "branch": "staging",
      "platform": "bitbucket",
      "repo_slug": "webyes-apps/auditor-app-monorepo",
      "cloned_at": "<UTC ISO 8601>",
      "setup_commands_run": ["npm install"],
      "setup_failed": false,
      "skipped": false
    }
  ]
}
```

Set `"skipped": true` (and omit `clone_path`, `cloned_at`, `setup_commands_run`) for any repo the user chose not to clone.
```

### 5. Pass context to Orchestrator

Return:
- `clone_paths`: map of `repo-name → absolute path`
- `primary_clone_path`: first repo in the list (used as `<project-repo>` for run folder)
- `task_type`: `normal` or `hotfix`
- `platform`: `bitbucket`

The Orchestrator stores this and passes `primary_clone_path` as `<project-repo>` for all subsequent agents.

---

## Step 2 — Check if Already Cloned

Check if the target repo already exists locally:

```bash
git -C <clone-path> remote get-url origin 2>/dev/null
```

- If it exists and the remote matches: `git fetch origin && git checkout <default_branch> && git pull`
- If it exists but remote doesn't match: warn the user and ask whether to re-clone or use the existing directory
- If it doesn't exist: clone it

**Clone path convention:** `<agentic-flow-repo>/workspace/<task-name>/` where `<task-name>` is the Jira ticket ID (e.g. `WEBYES-123`) or a slugified task title.

```bash
AGENTIC_FLOW_DIR=<absolute path to agentic-flow repo>
TASK=<ticket-id or slugified task title>
git clone <repo_url> $AGENTIC_FLOW_DIR/workspace/$TASK/<project-name>
cd $AGENTIC_FLOW_DIR/workspace/$TASK/<project-name>
git checkout <default_branch>
git pull
```

## Step 3 — Run Setup Commands

After cloning, run setup commands from the config in order:

```bash
cd <clone-path>
<setup_command_1>
<setup_command_2>
...
```

If any setup command fails:
- Log the failure and the command to `logs/clone.log`
- Ask the user whether to continue anyway or stop
- Do not silently skip failed setup steps

## Step 4 — Save Output

Write to `.agentic/runs/.clone-state-<task-slug>` in the **agentic-flow repo** (not the project repo — the project may not have `.agentic/` yet at this stage). `<task-slug>` is the Jira ticket ID (e.g. `WEBYES-123`) or the slugified task title — the same identifier used for the workspace folder. This prevents collisions when multiple runs are active simultaneously.

```json
{
  "project_name": "<project-name>",
  "repo_url": "<repo_url>",
  "clone_path": "<absolute path to cloned repo>",
  "default_branch": "<branch>",
  "platform": "github|bitbucket",
  "repo_slug": "<slug for Bitbucket or owner/repo for GitHub>",
  "cloned_at": "<UTC ISO 8601 timestamp>",
  "setup_commands_run": ["<cmd1>", "<cmd2>"],
  "setup_failed": false
}
```

## Step 5 — Pass Context to Orchestrator

Return to the Orchestrator with:
- `clone_path`: absolute path to the cloned project repo
- `platform`: github or bitbucket
- `repo_slug`: for use by PR creation script
- `test_command`: from config, for Coder agent to use

The Orchestrator stores this in the run folder once it's created.

## Handoff

Return control to the Orchestrator. The Orchestrator will:
1. Set `clone_path` as the active `<project-repo>` for all subsequent agents
2. Proceed to Requirement Collection
