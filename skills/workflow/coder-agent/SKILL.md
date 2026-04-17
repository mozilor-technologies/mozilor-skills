# Coder Agent

## Purpose

Implement code and tests for each subtask defined in the plan, run the tests, and verify everything passes before handing off to the Code Review agent. Testing is part of this agent — there is no separate tester.

## When to Use

- After the Planner agent has saved `plan.json` and it has been human-approved (medium/high complexity)
- Directly after the Requirement Collection agent for low-complexity tasks (no `plan.json` in that case)

## Input

Read from the run folder in the project repo:

```
<project-repo>/.agentic/runs/<run-folder>/requirement.json
<project-repo>/.agentic/runs/<run-folder>/plan.json   ← may not exist for low-complexity tasks
```

If `plan.json` is missing and complexity is not low, stop and tell the user to run the Planner agent first.

## Skill Loading Order

Before writing any code, load skills in this order. **Always prefer more specific over more general.**

### 1. Project-specific skill (highest priority)
Check the project repo for a project-level skill:
```
<project-repo>/.claude/skills/coder/SKILL.md
<project-repo>/skills/coder/SKILL.md
<project-repo>/CLAUDE.md         ← always read this if it exists
```
If found, follow it. It overrides everything below for any convention it defines.

### 2. Detect the stack
Inspect the project repo root to identify the stack before choosing a skill:

| File present | Stack |
|---|---|
| `go.mod` | Go |
| `requirements.txt` or `pyproject.toml` | Python |
| `composer.json` | PHP |
| `package.json` with `next` in dependencies | Next.js |
| `package.json` (no `next`) | Node.js |
| `*.sql` migrations only, no app code | PostgreSQL patterns only |

For polyglot repos, load the skill that matches the files being changed. If still ambiguous, escalate to the user before coding.

### 3. Language / framework skill (fallback)
Load the matching skill from the agentic-flow skills library using the detected stack. All paths are relative to `${AGENTIC_FLOW_DIR}`.

| Stack | Skill to load |
|---|---|
| Python + FastAPI | `${AGENTIC_FLOW_DIR}/skills/python/fastapi/SKILL.md` |
| Python (general) | `${AGENTIC_FLOW_DIR}/skills/python/python-patterns/SKILL.md` |
| Node.js backend (Express / Fastify) | `${AGENTIC_FLOW_DIR}/skills/nodejs/nodejs-backend-patterns/SKILL.md` |
| Next.js + Supabase | `${AGENTIC_FLOW_DIR}/skills/nodejs/nextjs-supabase/SKILL.md` |
| Go | `${AGENTIC_FLOW_DIR}/skills/go/go-test/SKILL.md` + `${AGENTIC_FLOW_DIR}/skills/go/go-integration/SKILL.md` (load both; use go-integration for service/HTTP integration tests, go-test for unit tests) |
| PHP / Laravel | `${AGENTIC_FLOW_DIR}/skills/php/laravel/SKILL.md` |
| TypeScript / React | `${AGENTIC_FLOW_DIR}/skills/typescript/vercel-react-best-practices/SKILL.md` |
| PostgreSQL | `${AGENTIC_FLOW_DIR}/skills/postgres/postgres-patterns/SKILL.md` |

### 4. Layering
Project skill and language skill are not mutually exclusive. Load both where applicable — project skill conventions take precedence on any point of conflict.

## Execution Order

Read `depends_on` and `execution` fields from each subtask in `plan.json`.

- **Sequential subtasks**: execute one at a time, in dependency order
- **Parallel subtasks**: subtasks with no unmet dependencies and `execution: parallel` may be implemented concurrently
- Never start a subtask before all its `depends_on` subtasks are complete and tested

For low-complexity tasks with no `plan.json`: treat the entire requirement as a single subtask.

## Per-Subtask Workflow

Repeat this for every subtask:

### 1. Understand Before Coding
- Read the subtask's `description`, `acceptance_criteria`, and `test_cases`
- Read existing files that will be impacted (`files_impacted`)
- Understand existing patterns before writing anything new
- If anything is unclear, check `requirement.json` source materials (`extracted_content`)
- If `gitnexus_available` is `true` in `<project-repo>/.agentic/runs/<run-folder>/run-state.json`, load `${AGENTIC_FLOW_DIR}/skills/gitnexus/SKILL.md` and follow the **Architecture Exploration** workflow to understand execution flows through the affected files, and the **Impact Analysis** workflow to understand what depends on what you are about to change. Do this before writing any code. If `gitnexus_available` is absent or `false`, skip this step.

### 2. Implement
- Write only what is needed for this subtask — no more
- Follow the conventions from the loaded language/framework skill
- Do not refactor unrelated code
- Do not add features not in the acceptance criteria
- Keep changes focused: touch only the files listed in `files_impacted` unless unavoidable

### 3. Write Tests
Write tests immediately after implementing — not after all subtasks are done.

Test coverage requirements:
- All acceptance criteria must have a corresponding test
- Happy path must be covered
- Edge cases defined in `test_cases` must be covered
- For API changes: write API-level tests (request/response validation, status codes, error responses)
- For logic changes: write unit tests for the changed function/module
- Do not test framework internals or third-party library behavior

Follow the testing conventions from the loaded language skill (e.g. `${AGENTIC_FLOW_DIR}/skills/python/python-testing/SKILL.md`, `${AGENTIC_FLOW_DIR}/skills/go/go-test/SKILL.md`).

### 4. Run Tests
Before running tests, verify a test suite exists:

```bash
# Check for test framework indicators
ls tests/ test/ spec/ __tests__/ 2>/dev/null || \
  find . -name "*_test.go" -o -name "*.test.js" -o -name "test_*.py" 2>/dev/null | head -5
```

**If no test suite exists and the project config defines a `test_command`:** run it and treat a non-zero exit as failure.

**If no test suite and no `test_command` defined:** set up the test framework following the language skill's conventions before writing any tests. Log the setup steps to `logs/coder.log`. If the framework cannot be determined, escalate to the user before proceeding.

Run tests using the `test_command` from the project config (passed by the Orchestrator) or detect from the project stack:

```bash
# From project config (preferred):
<test_command from config>

# Stack detection fallback:
pytest tests/                    # Python
go test ./...                    # Go
npm test                         # Node.js
php artisan test                 # Laravel
```

Capture the full test output.

### 5. Fix Loop
If tests fail:

- Read the failure output carefully before attempting a fix
- Apply a targeted fix — do not rewrite working code
- Re-run tests after each fix
- **Maximum 5 fix attempts per subtask**
- **If the same test fails 3 times in a row on the same assertion**: stop fixing. Discard the implementation for this subtask, log the failure, and restart the subtask from scratch with a different approach.
- After 5 total failed attempts without passing: escalate to the user. Explain what was tried and what failed. Do not continue to the next subtask.

### 6. Log Subtask Completion
Once tests pass, append to `.agentic/runs/<run-folder>/logs/coder.log`:
- Subtask ID and title
- Files created or modified
- Tests written and their pass/fail result
- Number of fix iterations needed
- Any notable implementation decisions

## Implementation Rules

1. **No one-shot generation.** Implement step by step. Read existing code first, then write.
2. **Only change what's needed.** Do not touch files outside the subtask scope.
3. **No speculative features.** If it's not in the acceptance criteria, don't build it.
4. **No comments for obvious code.** Only comment where logic is non-obvious.
5. **Security by default.** No SQL injection, no hardcoded secrets, no XSS, no unsafe deserialization. If in doubt, follow OWASP top 10.
6. **Fail loudly.** Do not swallow exceptions silently. Errors should be logged and surfaced.

## G4A — Reasoning Capture (MANDATORY)

Load `${AGENTIC_FLOW_DIR}/skills/reasoning-capture/SKILL.md` for the full schema. Summary below.

### At session start — before reading plan.json or writing any code

Create `<project-repo>/.agentic/runs/<run-folder>/g4a/coder-reasoning.json`:

```json
{
  "schema_version": "1.0",
  "agent": "coder",
  "run_folder": "<run-folder-name>",
  "task": "Implement: <subtask title or requirement title>",
  "started_at": "<UTC ISO 8601>",
  "context": {
    "trigger": "plan.json approved / low-complexity direct from requirement",
    "files_read": [],
    "key_symbols": []
  },
  "reasoning": {
    "hypothesis": "<initial read on the change before reading any code>",
    "approach": "<implementation strategy — e.g. extend existing service vs create new>",
    "alternatives_considered": [],
    "confidence": 0.7
  },
  "outcome": {
    "status": "in_progress"
  }
}
```

Update `context.files_read` as you read files during implementation.

Create the `g4a/` directory if it does not exist: `mkdir -p <run-folder>/g4a/`

### At session end — after implementation.json is saved

Update the same file:

```json
{
  "schema_version": "1.0",
  "agent": "coder",
  "run_folder": "<run-folder-name>",
  "task": "Implement: <subtask title or requirement title>",
  "started_at": "<UTC ISO 8601>",
  "completed_at": "<UTC ISO 8601>",
  "context": {
    "trigger": "plan.json approved / low-complexity direct from requirement",
    "files_read": ["<every file read during implementation>"],
    "key_symbols": ["<key functions or classes touched>"]
  },
  "reasoning": {
    "hypothesis": "<confirmed or corrected assessment>",
    "approach": "<what was actually done>",
    "alternatives_considered": [
      {
        "option": "<alternative implementation approach>",
        "rejected_because": "<concrete reason>"
      }
    ],
    "why_this_approach": "<the key insight — e.g. 'reusing X avoided duplicating auth logic'>",
    "confidence": 0.9
  },
  "outcome": {
    "status": "complete|partial|blocked",
    "summary": "<paragraph: subtasks completed, key decisions, anything non-obvious>",
    "files_changed": ["<every file created or modified>"],
    "tests_passed": true,
    "follow_up": "<anything the reviewer should pay extra attention to>"
  }
}
```

## Commit Changes

Once all subtasks are complete and all tests pass, commit the implementation:

```bash
git add <files changed across all subtasks>
git commit -m "<ticket-id>: <short requirement title>"
```

Commit message format:
- With Jira ticket: `PROJ-123: Add login button`
- Without Jira ticket: `feat: <short description>` or `fix: <short description>`

If any `git add` or `git commit` fails, stop and report to the Orchestrator before writing `implementation.json`.

## Output

Once the commit succeeds and all subtasks are complete, save:

```
<project-repo>/.agentic/runs/<run-folder>/implementation.json
```

```json
{
  "schema_version": "1.0",
  "run_folder": "<run-folder>",
  "completed_at": "<UTC ISO 8601 timestamp>",
  "subtasks": [
    {
      "id": 1,
      "title": "",
      "status": "complete",
      "files_changed": ["<file path>"],
      "tests_added": ["<test file or test name>"],
      "test_result": "pass",
      "fix_iterations": 0,
      "notes": "<any implementation decisions worth noting>"
    }
  ],
  "all_tests_pass": true,
  "known_risks": ["<anything the reviewer should pay attention to>"],
  "implementation_notes": "<overall summary>"
}
```

## Handoff

Once `implementation.json` is saved and all tests pass, return control to the Orchestrator with:
- Run folder path
- Confirmation that `implementation.json` has been saved and `all_tests_pass: true`

Do not invoke the Code Review agent directly. The Orchestrator owns stage transitions.
