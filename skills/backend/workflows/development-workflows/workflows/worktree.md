# Worktree Workflow

Use this workflow to isolate feature/refactor development in a dedicated git worktree.

## When to use

Use worktree when:

- New feature or refactor spans multiple files
- Impact analysis is HIGH/MEDIUM or uncertain
- You want safe experimentation without polluting current workspace
- User explicitly asks for isolation

Skip worktree when:

- Tiny, low-risk, single-file fix
- User explicitly prefers direct workspace

## Decision tree

```text
Feature/refactor request?
  |
  +-- No --> Stay in current workspace
  |
  +-- Yes --> Ask user preference
             1) Worktree (recommended)
             2) Direct workspace
```

Mandatory prompt:

```text
This is a feature/refactor request. Choose development environment:
1. Worktree (recommended) - isolated copy with feature branch
2. Direct - continue in current workspace
Which do you prefer?
```

## Branch naming

Use repository conventions:

- Feature: `feature/<short-description>`
- Refactor: `refactor/<short-description>`
- Fix: `fix/<short-description>`

Examples:

- `feature/gift-card-import`
- `refactor/billing-service`
- `fix/rate-limiter-leak`

## Setup flow

### 1) Prepare names

- `repo_name`: repository folder name
- `feature_slug`: short kebab-case identifier
- `branch_name`: based on type (`feature/...`, `refactor/...`, `fix/...`)

### 2) Create worktree

```bash
git worktree add ../<repo-name>-<feature-slug> -b <branch-name>
cd ../<repo-name>-<feature-slug>
```

Example:

```bash
git worktree add ../sr-import-export-backend-gift-card -b feature/gift-card-import
cd ../sr-import-export-backend-gift-card
```

### 3) Verify setup

```bash
pwd
git branch --show-current
git status
```

Expected:

- Current dir is new worktree
- Branch matches intended feature branch
- Working tree starts clean

## Management

List worktrees:

```bash
git worktree list
```

Cleanup after merge:

```bash
cd <original-repo-path>
git worktree remove ../<repo-name>-<feature-slug>
git branch -d <branch-name>
```

Prune stale metadata:

```bash
git worktree prune
```

## Integration sequence

Typical isolated flow:

1. Worktree created
2. Plan written
3. Impact analysis completed
4. If HIGH/MEDIUM: route to `peer-review.md`
5. Implement changes in worktree
6. Run `quality-gates.md`
7. Create PR
8. Cleanup worktree after merge

## Example

User: "Add a new gift card import flow"

Assistant flow:

1. Offer worktree/direct choice
2. User chooses worktree
3. Create `feature/gift-card-import` in dedicated worktree
4. Continue planning and implementation inside that worktree

## Success checklist

- [ ] User preference captured (worktree/direct)
- [ ] Correct branch naming used
- [ ] Worktree created and entered
- [ ] Branch and status verified
- [ ] Next workflow identified (peer review or implementation)
