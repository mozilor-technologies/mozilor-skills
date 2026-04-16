---
name: start-feature
description: "MANDATORY: Invoke this skill for ANY request to build, implement, create, or add a new feature, page, route, component, or section — especially when Figma URLs are provided. This is a BLOCKING REQUIREMENT: do NOT write any code, read any files for implementation, or invoke coding-standards/figma-to-code skills directly before invoking this skill first. Trigger phrases include (but are not limited to): 'implement', 'build', 'create', 'add', 'make', 'design', 'set up a new route/page/screen', 'add X to sidebar', 'implement these Figma designs', 'build feature X with mockdata'. This skill orchestrates: research → requirements alignment → design doc → user approval → code generation → validation. Never skip straight to code."
---

# /start-feature — Entry Point

## Input

$ARGUMENTS

---

## Step 0 — GitNexus Setup Check

Call `gitnexus_list_repos()`. If it fails with "tool not found" or any MCP error, tell the user:

```
GitNexus is not set up. This workflow uses it for blast-radius analysis and call-chain debugging.

To install:
  1. npx gitnexus analyze
  2. claude mcp add gitnexus -- npx -y gitnexus@latest mcp
  3. Restart Claude Code, then re-run /start-feature

Reply: skip gitnexus  — to continue without it.
```

Wait. If "skip gitnexus" → proceed without it (skip all GitNexus steps downstream). Otherwise wait for setup.

---

## Step 1 — Parse Input

Extract from the raw input:
- **Requirement** — the feature description
- **Confluence URL** — optional, contains `atlassian.net` or `confluence`
- **Figma URL(s)** — optional, one or more URLs containing `figma.com`

Then invoke the `start-feature-single-feature-flow` skill:

> Inputs:
> - Requirement: [REQUIREMENT]
> - Confluence URL: [URL OR "none"]
> - Figma URL(s): [URL(S) OR "none"]
> - GitNexus available: [yes / no]
