# Quality Gates Workflow

Run this workflow before every commit and before PR readiness.

## Rule

- Mandatory: do not skip quality gates
- Commits are blocked until required checks pass

## Checklist

## 1) Impact and scope verification

Goal: confirm only intended scope changed.

```text
gitnexus_detect_changes({scope: "all"})
```

Checks:

- [ ] Only planned files/symbols changed
- [ ] No unexpected execution-flow impact
- [ ] Blast radius still acceptable

If unexpected scope appears:

- investigate unexpected files/symbols
- keep intended changes only
- rerun impact check

## 2) Tests

Run repo test command (usually `uv run pytest tests/`).

Checks:

- [ ] Test suite passes
- [ ] Failing tests fixed (not skipped)
- [ ] New behavior has adequate tests

## 3) Lint and formatting

Run repo lint/format commands (usually Ruff).

Checks:

- [ ] No lint errors
- [ ] Code formatted
- [ ] No unused imports/dead code introduced

## 4) Security and safety

Checks:

- [ ] No hardcoded secrets/credentials
- [ ] No sensitive logs added
- [ ] Input boundaries validated
- [ ] No obvious injection/path traversal risks

## 5) Build verification (when applicable)

Checks:

- [ ] Build/container image succeeds
- [ ] Dependency resolution is clean
- [ ] No new critical build warnings

## 6) Documentation and clarity

Checks:

- [ ] Complex new logic documented where needed
- [ ] Plan/spec comments updated if behavior changed
- [ ] Commit message scope matches real changes

## 7) Final scope lock

Before commit:

- [ ] Working tree contains only intended files
- [ ] Generated artifacts are intentional
- [ ] No temporary debugging leftovers

## Decision tree

```text
Run quality gates
  |
  +-- Any check fails? -- Yes --> Fix root cause -> rerun full gates
  |
  +-- All checks pass? -- Yes --> Safe to commit
```

## Example run

```text
1) gitnexus_detect_changes(scope=all) -> expected 4 files only
2) uv run pytest tests/ -> all pass
3) uv run ruff check . && uv run ruff format . -> clean
4) secret scan/manual check -> clean
5) final git status -> expected changes only
=> commit allowed
```

## One-command verification (if repo provides it)

If the repository defines a combined verify command (for example `./scripts/verify.sh`), run it in addition to targeted checks.

## Configuration

Typical command configuration expected by this workflow:

```json
{
  "commands": {
    "test": "uv run pytest tests/",
    "lint": "uv run ruff check . && uv run ruff format .",
    "verify": "./scripts/verify.sh"
  },
  "impact_analysis": {
    "tool": "gitnexus"
  }
}
```

## Integration with SPARC

Use this workflow at SPARC completion:

```text
SPARC Refinement complete
  ->
Run quality gates
  ->
All checks pass? -- Yes --> commit/PR-ready
                 -- No  --> fix root cause and rerun
```

## Common failures and fixes

### Failure: tests failing

- Run the test command and inspect first failure
- Fix the root cause (do not skip or mute tests)
- Rerun until suite is green

### Failure: lint errors

```bash
uv run ruff check --fix .
uv run ruff format .
```

Then fix any remaining issues manually and rerun lint.

### Failure: unexpected files changed

- Run scope verification again: `gitnexus_detect_changes({scope: "all"})`
- Remove unrelated edits from the commit scope
- Re-check that only intended files remain

### Failure: scope too large

- Reassess blast radius and risks
- Split into smaller PRs when possible
- Route through `peer-review.md` for HIGH/MEDIUM risk

## Integration points

- Use after implementation and after bug fixes
- Use after peer-review-driven plan updates before merge work
- Pair with `qa.md` when reviewer-style risk signoff is needed

## Success checklist

- [ ] Impact/scope verified
- [ ] Tests passing
- [ ] Lint/format clean
- [ ] Security checks clean
- [ ] Docs/clarity updated
- [ ] Final scope lock complete
