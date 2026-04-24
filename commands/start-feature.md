---
description: "MANDATORY for ANY request to build, implement, create, or add a new feature, page, route, component, endpoint, or service. Auto-detects frontend (React/Next.js), backend (Node.js/Python/Go/Laravel), and fullstack stacks, then runs the correct pipeline automatically."
argument-hint: "[feature description] [optional: confluence-url] [optional: figma-url]"
---

# /start-feature

## Input

$ARGUMENTS

---

## Step 0 — GitNexus Setup Check

Call `gitnexus_list_repos()`. If it fails, tell the user:

```
GitNexus is not set up. This workflow uses it for blast-radius analysis.

To install:
  1. npx gitnexus analyze
  2. claude mcp add gitnexus -- npx -y gitnexus@latest mcp
  3. Restart Claude Code, then re-run /start-feature

Reply: skip gitnexus — to continue without it.
```

Wait. If "skip gitnexus" → proceed without it. Otherwise wait for setup.

---

## Step 1 — Parse Input

Extract:
- **Requirement** — the feature description
- **Confluence URL** — optional, contains `atlassian.net` or `confluence`
- **Figma URL(s)** — optional, contains `figma.com`

---

## Step 2 — Auto-Detect Stack

Read the following files at the project root to identify the stack. Use the Read tool for each that exists.

**Signals to collect:**

| File | What to look for |
|------|-----------------|
| `shopify.app.toml` | Shopify app (check if file exists) |
| `package.json` | `dependencies` + `devDependencies` key names |
| `pyproject.toml` | `[tool.poetry.dependencies]` or `[project]` dependencies |
| `requirements.txt` | Package names |
| `go.mod` | module line |
| `composer.json` | `require` key |

**Shopify indicator**: `shopify.app.toml` exists → set **[IS_SHOPIFY]** = `true`, **[STACK]** = `fullstack`, **[BACKEND_LANG]** = `nodejs`. Skip remaining stack detection and go directly to the Shopify Plugin Check below.

**If not Shopify:**

**Frontend indicators** (in package.json): `react`, `next`, `vue`, `@angular/core`, `svelte`, `solid-js`, `@remix-run`, `preact`, `gatsby`, `astro`, `@nuxt/`

**Backend Node.js indicators** (in package.json): `express`, `fastify`, `koa`, `@nestjs/core`, `hono`, `@hapi/hapi`, `@apollo/server`, `@trpc/server`

**Python backend**: any of `fastapi`, `django`, `flask`, `starlette`, `tornado` in requirements

**Go backend**: `go.mod` exists

**PHP backend**: `composer.json` with `laravel/framework`

### Stack decision

| Condition | STACK |
|-----------|-------|
| Frontend indicators only | `frontend` |
| Backend indicators only | `backend` |
| Both frontend + backend indicators | `fullstack` |
| `next` / `nuxt` / `remix` / `sveltekit` — check if `src/app/api/` or `pages/api/` exists | If yes: `fullstack` / If no: `frontend` |
| pyproject.toml / go.mod / composer.json (no frontend signals) | `backend` |
| Cannot determine | prompt user |

**Backend language** (if backend or fullstack):

| Signal | BACKEND_LANG |
|--------|-------------|
| fastapi / django / flask in requirements | `python` |
| express / nestjs / hono in package.json | `nodejs` |
| go.mod | `go` |
| laravel in composer.json | `php` |

Set **[STACK]** = `frontend` / `backend` / `fullstack`
Set **[BACKEND_LANG]** = `nodejs` / `python` / `go` / `php` / `none`
Set **[IS_SHOPIFY]** = `false`

### Shopify Plugin Check *(IS_SHOPIFY only)*

If **[IS_SHOPIFY]** = `true`, run this exact command — do not modify it:
```bash
claude plugin list 2>&1 | grep -A3 "shopify-plugin" | grep -c "✔ enabled"
```

This outputs `1` if the plugin is installed and enabled, `0` otherwise.

**If the output is `0` — you MUST stop here. Do not read any more files. Do not proceed to stack confirmation. Do not attempt the feature. Output this message exactly and return:**

```
🛑  Shopify plugin required but not installed.

/start-feature needs the Shopify AI Toolkit to research APIs, validate
GraphQL queries, and generate correct Shopify code.

Install it first:
  /plugin marketplace add Shopify/shopify-ai-toolkit
  /plugin install shopify-plugin@shopify-plugin

Then restart Claude Code and re-run /start-feature.
```

**If the output is `1` — continue.**

**Present to user:**
```
## Stack Detected: [STACK]
Framework: [detected framework and version if visible]
[If backend or fullstack] Backend language: [BACKEND_LANG]
[If IS_SHOPIFY] Platform: Shopify (shopify.app.toml detected)

Reply **confirmed** to proceed, or correct me.
```

Wait for confirmation before proceeding.

---

## Step 3 — Fullstack Feature Scope *(fullstack only)*

If **[STACK]** = `fullstack` and it is not obvious from the requirement which part is being built:

```
Is this feature:
- **ui** — UI component/page only (frontend pipeline)
- **api** — Backend endpoint/service only (backend pipeline)
- **both** — Full feature spanning UI + API (run backend pipeline first, then frontend)

Reply with one of the above.
```

Wait for reply. Set **[PIPELINE]** based on response.

For `frontend` and `backend` stacks: **[PIPELINE]** = stack type directly.

---

## Pre-load Phase — Do This Before Spawning Any Agent

### 1. Extract coding rules digest

Read `.claude/skills/coding-standards/SKILL.md`. Extract and condense into **[CODING_RULES_DIGEST]** (target: under 300 tokens):
- CSS/Styling critical rule — verbatim (skip if backend-only)
- Language rules (TypeScript / Python / Go / PHP) — bullet list
- Naming conventions — file naming only
- Do Not list — verbatim

**Do not have sub-agents re-read the full coding-standards file** — pass this digest to all implementation and review agents.

### 2. Set flags

- **[FIGMA_AVAILABLE]** = "yes" if Figma URLs provided AND pipeline includes `frontend`, else "no"
- **[GITNEXUS_AVAILABLE]** = yes / no from Step 0
- **[IS_SHOPIFY]** = yes / no from Step 2

---

## Step 3.5 — Complexity Check

**Simple** — ALL must be true: no URLs, targets a single component / function / endpoint / file.
**Complex** — any: has URL, spans multiple files, new route or endpoint, new store or model, ambiguous scope.

→ Simple: skip Phase 1, go directly to Phase 1.5.
→ Complex: run Phase 1 first.

---

## Phase 1 — Research *(complex only)*

### If PIPELINE includes `frontend`:

Read `${CLAUDE_PLUGIN_ROOT}/agents/frontend/research-agent.md` with the Read tool, then spawn:

```
Agent(
  subagent_type: "general-purpose",
  model: "sonnet",
  description: "Research (frontend): [feature name]",
  prompt: [full content of frontend/research-agent.md] + """

ARGUMENTS:
Requirement: [REQUIREMENT]
Confluence URL: [URL OR "none"]
Figma URL(s): [URL(S) OR "none"]
IS_SHOPIFY: [IS_SHOPIFY]
"""
)
```

Wait for the structured research summary (≤800 tokens). Store as **[FRONTEND_RESEARCH]**.

### If PIPELINE includes `backend` (spawn in parallel with frontend research if pipeline = `both`):

Read `${CLAUDE_PLUGIN_ROOT}/agents/backend/researcher.md` with the Read tool, then spawn:

```
Agent(
  subagent_type: "general-purpose",
  model: "opus",
  description: "Research (backend): [feature name]",
  prompt: [full content of backend/researcher.md] + """

Research Task:
Feature: [REQUIREMENT]
Confluence URL: [URL OR "none"]
IS_SHOPIFY: [IS_SHOPIFY]

Focus on: similar features in this codebase, patterns to follow (service structure, error handling, auth), and impact points (what changing existing code would break).
"""
)
```

Wait for the research report. Store as **[BACKEND_RESEARCH]**.

---

## Phase 1.5 — Requirements Alignment *(no subagent — respond directly)*

Present your understanding. **Do not proceed until the user confirms.**

```
## Requirements Alignment

**Feature:** [name]
**Goal:** [one-sentence synthesis]
**Stack:** [STACK] → [PIPELINE] pipeline

**What I will build:**
- [Key functional thing 1]
- [Key functional thing 2]

**Scope:**
[Frontend scope if applicable:]
- Screens/views: [list or "inferred"]
- New components: [list or "none"]
- Modified components: [list or "none"]

[Backend scope if applicable:]
- New endpoints: [list or "none"]
- New services/models: [list or "none"]
- Schema changes: [yes — describe / no]

**State and data:**
- [Frontend: new store / existing stores touched]
- [Backend: new models / external integrations]
- API endpoints: [yes — [list] / no / unclear]

**Assumptions — correct me if wrong:**
1. [Assumption]

**Open questions:**
1. [Question]

Reply with corrections, or **confirmed** to proceed.
```

For **simple features**:
```
## Requirements Alignment

I understand you want to: [one sentence].
- Target: [file/component/endpoint]
- Change: [exact behavior change]
- Pipeline: [frontend/backend]

Reply **confirmed** to proceed, or correct me.
```

Only ask where a genuine gap exists (1–4 questions, max 6). Wait for confirmation before proceeding.

---

## Phase 2 — Design Document / Feature Spec

### If PIPELINE = `backend`:

Read `${CLAUDE_PLUGIN_ROOT}/agents/backend/spec-agent.md` with the Read tool, then spawn:

```
Agent(
  subagent_type: "general-purpose",
  model: "sonnet",
  description: "Spec: [feature name]",
  prompt: [full content of backend/spec-agent.md] + """

ARGUMENTS:
[CONFIRMED REQUIREMENTS BLOCK]
[BACKEND_RESEARCH OR RAW REQUIREMENT]
CODING_RULES_DIGEST: [CODING_RULES_DIGEST]
BACKEND_LANG: [BACKEND_LANG]
IS_SHOPIFY: [IS_SHOPIFY]
"""
)
```

Wait for `SPEC_PATH: docs/specs/[feature-name].md`.
Store as **[DESIGN_PATH]**. Derive **[ITERATION_STATE_PATH]** = `ai-context/iteration-state/[feature-name].json`.

### If PIPELINE = `frontend` or pipeline includes UI:

Read `${CLAUDE_PLUGIN_ROOT}/agents/frontend/design-agent.md` with the Read tool, then spawn:

```
Agent(
  subagent_type: "general-purpose",
  model: "sonnet",
  description: "Design: [feature name]",
  prompt: [full content of frontend/design-agent.md] + """

ARGUMENTS:
[CONFIRMED REQUIREMENTS BLOCK]
[FRONTEND_RESEARCH OR RAW REQUIREMENT]
CODING_RULES_DIGEST: [CODING_RULES_DIGEST]
FIGMA_AVAILABLE: [FIGMA_AVAILABLE]
IS_SHOPIFY: [IS_SHOPIFY]
"""
)
```

Wait for `DESIGN_PATH: ai-context/designs/[feature-name].md`.
Store as **[DESIGN_PATH]**. Derive **[ITERATION_STATE_PATH]** = `ai-context/iteration-state/[feature-name].json`.

### If PIPELINE = `both`:

Run the backend spec first (it defines the API contracts the frontend will consume), then the frontend design. Wait for backend SPEC_PATH before spawning the frontend design agent, and pass the spec path as context so the design agent knows the API shape.

Store the backend spec path as **[SPEC_PATH]** and the frontend design path as **[DESIGN_PATH]**. Use **[DESIGN_PATH]** for all subsequent phase references.

---

## Phase 3 — Architecture Review *(human checkpoint)*

```
[Design/Spec] document written to: `[DESIGN_PATH]`

How would you like to review it?
- **here** — review inline in this conversation
- **pr** — create a pull request for team review
- **[feedback]** — provide feedback directly to revise now
```

**Wait.**

→ **here**: Tell user to review `[DESIGN_PATH]` and reply **approved** or with feedback.

→ **pr**: Read `.claude/skills/bitbucket-pr/SKILL.md`, spawn a haiku agent to create the PR, wait for `PR_URL`, tell user and wait for **approved**.

→ **[feedback]** directly: Re-read the relevant spec/design agent, spawn a revision with the feedback, re-present when done.

Repeat until approved.

---

## Phase 4 — Initialize Iteration Tracking

Write to **[ITERATION_STATE_PATH]** (create `ai-context/iteration-state/` if needed):
```json
{
  "iteration": 0,
  "max_iterations": 5,
  "status": "in_progress",
  "feature": "[FEATURE NAME]",
  "stack": "[STACK]",
  "pipeline": "[PIPELINE]",
  "issues": []
}
```

---

## Phase 5 — Implementation

### If PIPELINE = `frontend` or includes UI:

Read `${CLAUDE_PLUGIN_ROOT}/agents/frontend/codegen-agent.md` with the Read tool, then spawn:

```
Agent(
  subagent_type: "general-purpose",
  model: "sonnet",
  description: "Codegen: [feature name]",
  prompt: [full content of frontend/codegen-agent.md] + """

ARGUMENTS:
Design path: [DESIGN_PATH]
CODING_RULES_DIGEST: [CODING_RULES_DIGEST]
FIGMA_AVAILABLE: [FIGMA_AVAILABLE]
IS_SHOPIFY: [IS_SHOPIFY]
"""
)
```

Wait for files list — or surface BLOCKED to the user.

### If PIPELINE = `backend`:

Read `${CLAUDE_PLUGIN_ROOT}/agents/backend/implementer.md` with the Read tool, then spawn:

```
Agent(
  subagent_type: "general-purpose",
  model: "sonnet",
  description: "Implement: [feature name]",
  prompt: [full content of backend/implementer.md] + """

Implementation Task:
Implement the feature exactly as specified in: [DESIGN_PATH]

Read the spec document completely. Follow:
- Section 3 (Architecture) — patterns and file plan
- Section 2 (Pseudocode) — logic flow
- Section 5 (API Contracts) — exact endpoint signatures
- Section 4 (File Plan) — all files to create/modify

CODING_RULES_DIGEST: [CODING_RULES_DIGEST]
BACKEND_LANG: [BACKEND_LANG]
IS_SHOPIFY: [IS_SHOPIFY]
"""
)
```

Wait for implementation report — or surface BLOCKED to the user.

### If PIPELINE = `both`:

Run backend implementation first (blocking), then frontend codegen (after backend complete and API is available).

---

## Phase 6 — Parallel Validation

### If PIPELINE = `frontend` or includes UI:

Read all three agent files first:
- `${CLAUDE_PLUGIN_ROOT}/agents/frontend/test-agent.md`
- `${CLAUDE_PLUGIN_ROOT}/agents/frontend/code-review-agent.md`
- `${CLAUDE_PLUGIN_ROOT}/agents/frontend/security-agent.md`

Then in **one message**, spawn all three simultaneously:

**Agent A** (`model: "sonnet"`):
```
Agent(subagent_type: "general-purpose", model: "sonnet", description: "Tests: [feature name]",
  prompt: [test-agent.md content] + "ARGUMENTS: Design path: [DESIGN_PATH] | Feature name: [feature-name]")
```

**Agent B** (`model: "haiku"`):
```
Agent(subagent_type: "general-purpose", model: "haiku", description: "Code review: [feature name]",
  prompt: [code-review-agent.md content] + "ARGUMENTS: Design path: [DESIGN_PATH]\nCODING_RULES_DIGEST: [CODING_RULES_DIGEST]")
```

**Agent C** (`model: "haiku"`):
```
Agent(subagent_type: "general-purpose", model: "haiku", description: "Security review: [feature name]",
  prompt: [security-agent.md content] + "ARGUMENTS: Design path: [DESIGN_PATH]\nIS_SHOPIFY: [IS_SHOPIFY]")
```

### If PIPELINE = `backend`:

Read both agent files first:
- `${CLAUDE_PLUGIN_ROOT}/agents/backend/qa.md`
- `${CLAUDE_PLUGIN_ROOT}/agents/frontend/security-agent.md`

Then in **one message**, spawn both simultaneously:

**Agent A** (`model: "opus"`):
```
Agent(subagent_type: "general-purpose", model: "opus", description: "QA: [feature name]",
  prompt: [qa.md content] + "Code to Analyze:\nSpec: [DESIGN_PATH]\nReview all files listed in Section 4 (File Plan) of the spec.\nIS_SHOPIFY: [IS_SHOPIFY]")
```

**Agent B** (`model: "haiku"`):
```
Agent(subagent_type: "general-purpose", model: "haiku", description: "Security review: [feature name]",
  prompt: [security-agent.md content] + "ARGUMENTS: Design path: [DESIGN_PATH]\nIS_SHOPIFY: [IS_SHOPIFY]")
```

---

## Phase 7 — Iteration Loop

**Blocking (must fix):**
- Frontend: test failures, code review `critical`/`major`, security `critical`/`major`
- Backend: QA `CRITICAL`/`HIGH`, security `critical`/`major`

**Non-blocking (report only):** frontend `minor` / backend QA `MEDIUM`/`LOW`

**If no blocking issues:** go to Phase 8.

**If blocking:**
1. Read [ITERATION_STATE_PATH], increment `iteration`, write back.
2. If `iteration >= 5`: set `status: "max_iterations_reached"`, report to user, stop.

### Frontend fix loop:

3. Read `${CLAUDE_PLUGIN_ROOT}/agents/frontend/fix-agent.md`, spawn Fix Agent:
```
Agent(subagent_type: "general-purpose", model: "haiku", description: "Fix: [feature name] iteration [N]",
  prompt: [fix-agent.md content] + "ARGUMENTS: Design path: [DESIGN_PATH]\nIssues: [LIST WITH FILE, LINE, DESCRIPTION, SEVERITY]\nIS_SHOPIFY: [IS_SHOPIFY]")
```
4. If STUCK returned: surface to user, stop.
5. Re-run only agents that had blocking issues:
   - Tests failed → re-run A + B (sonnet + haiku)
   - Code review only → re-run B (haiku)
   - Security → re-run C (haiku, iteration 1 only)

### Backend fix loop:

3. Read `${CLAUDE_PLUGIN_ROOT}/agents/backend/debugger.md`, spawn Debugger Agent:
```
Agent(subagent_type: "general-purpose", model: "opus", description: "Debug: [feature name] iteration [N]",
  prompt: [debugger.md content] + "Issue to Debug:\n[ISSUE DESCRIPTION WITH FILE AND EVIDENCE]\nSpec path: [DESIGN_PATH]\nCODING_RULES_DIGEST: [CODING_RULES_DIGEST]\nIS_SHOPIFY: [IS_SHOPIFY]")
```
4. After fix: re-run QA agent A only (not security on subsequent iterations).
5. If STUCK returned: surface to user, stop.

Repeat until no blocking issues.

---

## Phase 8 — Complete

1. Write [ITERATION_STATE_PATH]: `{ "status": "complete", ... }`

2. Report:
```
Feature implementation complete.

Stack: [STACK] | Pipeline: [PIPELINE]
Iterations used: X / 5
Files created: [LIST]
Files modified: [LIST]
[Frontend] Tests: N passing
[Backend] Full test suite: passing
Minor issues (non-blocking): [LIST OR "none"]

Ready for your review.
```

3. Run notification:
```bash
osascript -e 'display notification "Feature is ready for your review" with title "start-feature complete" sound name "Glass"'
```

If GitNexus available, also run in background:
```bash
(npx gitnexus analyze > /dev/null 2>&1 && osascript -e 'display notification "Index refreshed" with title "GitNexus reindex done" sound name "Glass"') &
```
