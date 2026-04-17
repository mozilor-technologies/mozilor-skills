# Documentation Agent

## Purpose

Update project documentation to reflect the completed implementation. Ensures the change is usable, maintainable, and traceable by anyone who works on the project after this run.

## When to Use

- After the Code Review agent has approved the implementation (`review.json` result is `approved`)

## Input

Read from the run folder:

```
<project-repo>/.agentic/runs/<run-folder>/requirement.json
<project-repo>/.agentic/runs/<run-folder>/plan.json
<project-repo>/.agentic/runs/<run-folder>/implementation.json
<project-repo>/.agentic/runs/<run-folder>/review.json
```

## Skill Loading Order

Check for project-specific documentation conventions first:

```
<project-repo>/.claude/skills/documentation/SKILL.md
<project-repo>/skills/documentation/SKILL.md
<project-repo>/CLAUDE.md   ← check for doc conventions, existing doc locations
```

Project-specific conventions override the defaults below.

## Guard

Before doing anything, read `review.json` and verify `result === "approved"`. If not approved, stop immediately and tell the user the code review has not passed — do not update any documentation.

## Step 1 — Discover Existing Documentation

Before writing anything, find what documentation already exists in the project:

```bash
# Common doc locations to check
README.md
docs/
CHANGELOG.md
CLAUDE.md
openapi.yaml / swagger.yaml
*.md files in the root
```

Understand the format and style already in use. Match it. Do not introduce a new documentation format if one already exists.

**If a file that should be updated does not exist** (new or empty repository): create it from scratch following the project's tech stack conventions. Record it in `files_updated` with `sections_changed: ["initial creation"]`. Use these minimal starting points:
- `README.md` — project name, description, setup steps, how to run tests
- `CHANGELOG.md` — start with `## [Unreleased]` header
- `CLAUDE.md` — project conventions, key scripts, environment variables
- `openapi.yaml` — minimal valid OpenAPI 3.0 spec with the new endpoint

## Step 2 — Decide What Needs Updating

Not every change needs every doc updated. Use this to decide:

| Change type | Docs to update |
|---|---|
| New API endpoint | README (if API is documented there), OpenAPI/Swagger spec, CHANGELOG |
| New feature (user-facing) | README, CHANGELOG, CLAUDE.md if it affects dev workflow |
| Internal refactor / bug fix | CHANGELOG only |
| New environment variable or config | README (setup/config section), `.env.example` |
| DB schema change | README (if schema is documented), migration notes in CHANGELOG |
| New dependency added | README (if dependencies are listed), relevant setup docs |
| Breaking change | README, CHANGELOG (clearly marked as breaking), CLAUDE.md |

If unsure whether a doc needs updating, err on the side of updating it — it's easier to trim than to discover missing docs later.

## Step 3 — Update Documentation

### README
- Update only the sections relevant to the change
- Do not rewrite sections unrelated to this implementation
- If a new endpoint, config option, or feature was added — document it clearly
- Keep examples current — if an API signature changed, update the example

### CHANGELOG
Always add an entry. Use this format:

```markdown
## [Unreleased]

### Added
- <what was added and why it matters to users>

### Changed
- <what behaviour changed>

### Fixed
- <what bug was fixed>

### Breaking Changes
- <anything that breaks existing behavior — be explicit>
```

If no CHANGELOG exists, create one.

### CLAUDE.md
Update only if the change affects how developers or agents work in this project:
- New setup steps required
- New environment variables
- Changed conventions or patterns
- New scripts or commands to know about

### OpenAPI / Swagger
If the project has an API spec file and new endpoints were added or existing ones changed:
- Add or update the endpoint definition
- Include request/response schemas
- Document error responses
- Keep the spec valid — run a linter if available

### Other docs
Follow whatever exists in the project. Match the style exactly.

## Step 4 — Review Doc Changes

Before saving output, check:
- [ ] No placeholder text left (`TODO`, `TBD`, `<fill this in>`)
- [ ] Examples actually match the implementation
- [ ] No sensitive data (tokens, passwords, internal URLs) in docs
- [ ] Existing content not accidentally removed or overwritten
- [ ] CHANGELOG entry is clear to someone who wasn't involved in this run

## Output

Save to the run folder:

```
<project-repo>/.agentic/runs/<run-folder>/documentation.json
```

```json
{
  "schema_version": "1.0",
  "run_folder": "<run-folder>",
  "documented_at": "<UTC ISO 8601 timestamp>",
  "files_updated": [
    {
      "file": "<relative path from project root>",
      "sections_changed": ["<section name>"],
      "summary": "<one line: what was added or changed>"
    }
  ],
  "skipped": [
    {
      "file": "<file>",
      "reason": "<why it was not updated>"
    }
  ]
}
```

Append a summary to `.agentic/runs/<run-folder>/logs/documentation.log`.

## Handoff

Once `documentation.json` is saved, return control to the Orchestrator with:
- Run folder path
- Confirmation that `documentation.json` has been saved

Do not invoke the PR Creation agent directly. The Orchestrator owns stage transitions.
