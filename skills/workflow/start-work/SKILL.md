---
name: start-work
description: Use when starting a new development task — accepts a Jira ticket ID or free-text description and launches the full agentic pipeline
---

# Start Work

Entry point to the Mozilor agentic development pipeline.

## Usage

```
/start-work WEBYES-123
/start-work "add export button to reports page"
/start-work WEBYES-123 — add export button to reports page
```

## What Happens

Invokes `skills/orchestrator/SKILL.md` with the provided input as the trigger. The orchestrator runs the full pipeline in order — do not skip any step:

```
Pre-flight         → check for interrupted runs
Clone              → clone or pull all project repos
GitNexus Index     → index the codebase
Requirement        → first human interaction ← user confirms here
Plan               → break into subtasks (medium/high complexity)
Branch             → create feature branch
Code               → implement + tests
Review             → correctness, quality, security
Docs               → update docs
PR                 → create pull request
```

## Input

| Format | Example |
|---|---|
| Ticket ID only | `WEBYES-123` |
| Description only | `"add login button to dashboard"` |
| Both | `WEBYES-123 — add login button to dashboard` |

## Action

Load `skills/orchestrator/SKILL.md` and pass the full input as the trigger. Do not modify, skip, or reorder any orchestrator steps.
