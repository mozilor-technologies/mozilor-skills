# Peer Review Workflow

Use this workflow when plan risk is HIGH/MEDIUM or user requests review before coding.

## When to use

Offer peer review when:

- Impact analysis is HIGH/MEDIUM
- More than 10 direct dependents are affected
- Changes touch auth, billing, import/export core paths, or architecture
- User explicitly asks for plan review

Skip when:

- Impact is LOW and isolated
- User explicitly declines review

## Phase 1: Offer review choice

After impact analysis, ask:

```text
Impact Analysis: HIGH/MEDIUM
Affected scope:
- <X> direct dependents
- <Y> execution flows
- Key risks: <list>

Would you like to submit this plan for peer review before implementation?
1. Yes, create PR for review
2. No, proceed with implementation
```

## Phase 2: Create review PR

### 2.1 Ensure branch context

- Must be on non-main feature/refactor/fix branch
- Worktree is recommended for isolation

### 2.2 Create plan doc

Path: `docs/plans/<feature-slug>.md`

Template:

```markdown
# Plan: <Feature Title>

**Branch**: `<branch-name>`
**Date**: <YYYY-MM-DD>
**Status**: PENDING_REVIEW

## Objective
<What this change delivers>

## Impact Analysis
- Risk Level: HIGH/MEDIUM
- Direct Dependents: <count>
- Affected Processes: <list>
- Key Risks:
  - <risk 1>
  - <risk 2>

## Implementation Plan
### Files to Modify
| File | Change Type | Description |
|---|---|---|
| path/to/file.py | Modify | <description> |

### Files to Create
| File | Purpose |
|---|---|
| path/to/new.py | <purpose> |

### Step-by-step
1. <step 1>
2. <step 2>

## Test Strategy
- [ ] Unit tests
- [ ] Integration tests

## Rollback Plan
<revert approach>

## Open Questions
- <question>
```

### 2.3 Commit and push plan

```bash
git add docs/plans/<feature-slug>.md
git commit -m "docs: add implementation plan for <feature>"
git push -u origin <branch-name>
```

### 2.4 Create Bitbucket PR

Title format:

- `[PLAN REVIEW] <Feature Title>`

Use Bitbucket tooling with repository config values (`workspace`, `repository`, `source_branch`, `destination_branch`).

## Phase 3: Critical-thinking review loop

Do not accept every comment by default.

Decision tree:

```text
Comment received
  |
  +-- unclear? --------> ask clarifying question
  |
  +-- architecture/scope changing?
  |       |
  |       +-- yes -> run impact analysis again
  |
  +-- evaluate validity/trade-offs
          |
          +-- valid -------> update plan + reply resolved
          +-- partial -----> apply subset + explain rationale
          +-- invalid/risky -> explain concerns + propose alternative
```

Review criteria for each comment:

- Improves correctness, reliability, or maintainability?
- Reduces risk materially?
- Matches repository patterns and constraints?
- Introduces hidden costs or regressions?

### Response patterns

Accepted:

```markdown
Resolved: Updated plan to <change>. Benefit: <reason>.
See `docs/plans/<slug>.md` section: <section>.
```

Needs clarification:

```markdown
Clarification needed: I understand <interpretation>, but need detail on <gap>.
Questions:
1. <q1>
2. <q2>
```

Rejected/alternative:

```markdown
Analysis: I evaluated <suggestion> and found risks:
- <risk 1>
- <risk 2>
Alternative: <alternative>
Please confirm preferred direction.
```

### 3.4 Update and push revisions

```bash
git add docs/plans/<feature-slug>.md
git commit -m "docs: refine plan from review feedback"
git push
```

### 3.5 Resolve comments in PR

- Reply with disposition (accepted/partial/rejected)
- Link to exact updated plan section
- Keep unresolved blocking comments visible

### 3.6 Exit criteria

Proceed to implementation only when:

- Blocking review comments resolved, or
- Reviewer/user explicitly approves proceeding

## Phase 4: Hand-off

After approval:

1. Freeze reviewed plan scope
2. Implement per approved plan
3. Run `quality-gates.md` before commit

## Example: valid vs invalid feedback

Valid feedback example:

- Suggestion: stream large export rows instead of accumulating in memory
- Action: accepted and updated plan to streaming architecture

Potentially invalid feedback example:

- Suggestion: cache all Shopify responses globally
- Concern: stale data risk and cache invalidation complexity for one-time import/export jobs
- Action: reject or limit to narrow summary-cache use case with rationale

## Success checklist

- [ ] Peer review offered for HIGH/MEDIUM risk
- [ ] Plan doc created in `docs/plans/`
- [ ] `[PLAN REVIEW]` PR created on Bitbucket
- [ ] Every comment handled with critical analysis
- [ ] Plan updated only where justified
- [ ] Blocking comments resolved before implementation
