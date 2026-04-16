---
name: development-workflows
description: Standard development workflows - worktree isolation, plan review, quality gates, debugging, QA. Use throughout feature development lifecycle.
triggers:
  - "worktree"
  - "review pr"
  - "plan review"
  - "quality gates"
  - "verify"
  - "debug"
  - "qa"
  - "autofix"
  - "pre-commit"
---

# Development Workflows

## 1) Overview

This skill is the single entry point for operational development workflows across the lifecycle:

- Start safely in an isolated git workspace
- Review high-risk plans before coding
- Run required pre-commit quality checks
- Debug production or local failures using root-cause-first methods
- Perform QA-style review before merge
- Run autonomous fix loops in sandboxed environments

Progressive disclosure pattern:

- This `SKILL.md` is a workflow router and quick reference
- Detailed execution steps live in `workflows/*.md`
- Invoke only the workflow needed for the current step

## 2) Workflow Selection

Use this table to route quickly.

| Situation | Use This Workflow | Why |
|---|---|---|
| Starting a feature/refactor | [Worktree](workflows/worktree.md) | Isolate risk and keep main workspace clean |
| Plan touches high-risk paths (auth/billing/data architecture) or broad blast radius | [Peer Review](workflows/peer-review.md) | Validate plan before implementation |
| Ready to commit or open PR | [Quality Gates](workflows/quality-gates.md) | Mandatory verification before commit |
| Bug, incident, regression, unexpected behavior | [Debugging](workflows/debugging.md) | Structured root-cause investigation |
| Need code review-style assessment before merge | [QA](workflows/qa.md) | Severity-ranked findings and merge risk |
| User asks for autonomous run-to-completion fixes | [Autofix](workflows/autofix.md) | Controlled autonomous execution in sandbox |

Decision router:

```text
Request received
  |
  +-- "create worktree" / feature start --------> workflows/worktree.md
  |
  +-- "review this plan" / high-risk impact ----> workflows/peer-review.md
  |
  +-- "run quality gates" / "verify" ----------> workflows/quality-gates.md
  |
  +-- "debug" / incident -----------------------> workflows/debugging.md
  |
  +-- "qa" / "review code" --------------------> workflows/qa.md
  |
  +-- "autofix" --------------------------------> workflows/autofix.md
```

## 3) Workflow Links

Open the detailed workflow file directly for execution:

- [workflows/worktree.md](workflows/worktree.md): Git worktree isolation, branch setup, cleanup
- [workflows/peer-review.md](workflows/peer-review.md): Plan PR creation and critical review loop
- [workflows/quality-gates.md](workflows/quality-gates.md): Pre-commit verification checklist
- [workflows/debugging.md](workflows/debugging.md): Investigation and root-cause elimination
- [workflows/qa.md](workflows/qa.md): Review rubric and severity-based reporting
- [workflows/autofix.md](workflows/autofix.md): Autonomous fix cycle in sandbox

## 4) Integration With Other Skills

### `sr-import-export-developer`

Use this as the implementation skill for domain changes; use development workflows as control rails around it:

- Start phase: run [worktree](workflows/worktree.md) for isolation on feature/refactor work
- Planning phase: run [peer-review](workflows/peer-review.md) when risk is HIGH/MEDIUM
- Completion phase: always run [quality-gates](workflows/quality-gates.md)
- Defect handling: switch to [debugging](workflows/debugging.md)

### `sparc-developer`

Recommended mapping into SPARC:

| SPARC stage | Workflow |
|---|---|
| S (Specification) | Worktree decision + setup |
| P (Pseudocode/Plan) | Peer review for risky plans |
| R (Refinement/Build) | Debugging workflow when issues appear |
| C (Completion) | Quality gates (mandatory) + QA |

### GitNexus usage across workflows

- Impact gating: `gitnexus_impact` before risky symbol edits
- Scope checks: `gitnexus_detect_changes` before commit
- Investigation: `gitnexus_query` and `gitnexus_context` during debugging/QA

## 5) Quick Reference

### Trigger tests (expected behavior)

| Phrase | Expected Route | Expected Action |
|---|---|---|
| "create worktree" | `workflows/worktree.md` | Offer worktree/direct choice, then set up branch/worktree if selected |
| "run quality gates" | `workflows/quality-gates.md` | Execute full pre-commit checklist (impact/tests/lint/security/docs) |
| "review this plan" | `workflows/peer-review.md` | Create plan PR and run critical-thinking comment loop |

### Mandatory guardrails

- No band-aid fixes. Eliminate root causes.
- For high-risk scope, do plan review before coding.
- Before commit, run quality gates.
- Keep workflow output explicit: decision, actions taken, blockers, next step.

### Typical end-to-end flow

```text
Feature requested
  -> Worktree setup
  -> Plan + impact analysis
  -> Peer review (if HIGH/MEDIUM risk)
  -> Implement with domain skill
  -> Quality gates
  -> QA review
  -> PR/merge and cleanup
```

### Notes

- Repository is Bitbucket-hosted; use Bitbucket tooling for PR operations.
- Commands and conventions come from repository standards (`AGENTS.md`, repo config).
