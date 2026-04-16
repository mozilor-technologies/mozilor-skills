---
name: sparc-developer
description: Structured SPARC workflow for multi-file features with impact analysis, TDD, and completion checks.
triggers:
  - "use sparc"
  - "sparc workflow"
  - "sparc methodology"
  - "plan feature"
  - "structured implementation"
auto_workflow: none
requires_repo_config: true
config_path: .claude/config/repo.json
---

# SPARC Developer

Use SPARC when a change is non-trivial and needs a disciplined, phase-based flow.

SPARC phases:
- S: Specification
- P: Pseudocode
- A: Architecture
- R: Refinement
- C: Completion

## When To Use

Use SPARC for:
- multi-file features
- HIGH/MEDIUM risk changes
- refactors that alter service boundaries or pipeline behavior

Skip SPARC for:
- small one-file fixes
- low-risk typo or config-only changes

## Phase S: Specification

Goal: define scope, constraints, and success criteria.

Required output:
- `docs/specs/<feature-slug>.md`
- requirements (functional + non-functional)
- API shape and risks

Example:
- Input: `Build import preview summary`
- Output:
```markdown
# Import Preview Summary

## Objective
Show create/update/unchanged counts before import.

## API
POST /api/v1/jobs/{job_id}/preview

## Constraints
- Reuse existing job ownership checks
- Keep memory bounded via batching
```

Checklist:
- [ ] related code explored
- [ ] requirements documented
- [ ] explicit risks listed

## Phase P: Pseudocode

Goal: design logic before implementation.

Required output:
- pseudocode flow in spec
- test anchors (unit + integration + edge)

Example:
- Input: `Design preview diff flow`
- Output:
```python
def build_preview(job_id):
    rows = load_rows(job_id)
    stats = empty_stats()
    for batch in chunk(rows, 50):
        mapped = map_batch(batch)
        existing = fetch_existing(mapped)
        stats = accumulate_diff(stats, mapped, existing)
    return stats
```

Checklist:
- [ ] algorithm steps are explicit
- [ ] failure paths identified
- [ ] tests mapped to logic blocks

## Phase A: Architecture

Goal: map responsibilities and file boundaries.

Required output:
- file-level change plan
- dependency map
- impact analysis for touched symbols

Example:
- Input: `Plan files for preview feature`
- Output:
```text
Create:
- app/services/import_preview_service.py
- tests/test_import_preview_service.py

Modify:
- app/api/v1/endpoints/import_process.py
- app/schemas/api_responses.py

Impact checks:
- gitnexus_impact(target="import_process", direction="upstream")
```

Checklist:
- [ ] clear create/modify list
- [ ] integration points identified
- [ ] blast radius assessed

## Phase R: Refinement

Goal: implement with TDD, security, and performance discipline.

Execution order:
1. Write tests first.
2. Implement minimum logic to pass tests.
3. Integrate components.
4. Run security checks on inputs/auth/data scope.
5. Optimize only after correctness.

Example:
- Input: `Implement approved preview architecture`
- Output:
```text
R1 TDD: add failing tests -> implement -> tests pass
R2 Integration: wire endpoint/service/schema
R3 Security: validate auth + ownership checks
R4 Optimization: batch external lookups, avoid full in-memory loads
```

Checklist:
- [ ] tests drive implementation
- [ ] no band-aid fixes
- [ ] security checks done
- [ ] performance assumptions validated

## Phase C: Completion

Goal: verify readiness and communicate outcome.

Required output:
- quality checks passed
- scope verified
- concise completion summary

Example:
- Input: `Finalize preview feature`
- Output:
```text
Verification:
- uv run pytest tests/
- uv run ruff check .
- uv run ruff format .
- gitnexus_detect_changes(scope="all")

Summary:
- files changed
- tests status
- residual risks
```

Checklist:
- [ ] tests/lint green
- [ ] scope matches intent
- [ ] summary ready for PR/review

## Quick Start

For a new feature request:
1. Run S and create spec.
2. Add pseudocode in P.
3. Produce file plan and impact in A.
4. Implement via TDD in R.
5. Run verification and summarize in C.

## References

- [references/methodology.md](references/methodology.md): full SPARC workflow details
- [references/quickref.md](references/quickref.md): one-page checklist
- [references/agents.md](references/agents.md): optional agent role patterns
