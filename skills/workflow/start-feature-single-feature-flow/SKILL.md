---
name: start-feature-single-feature-flow
description: "Single-feature flow for the start-feature workflow. Runs research → requirements alignment → design → code generation → validation → iteration for one feature. Invoked by the start-feature entry point."
---

# Single-Feature Flow

## Inputs (from the entry point)

- **Requirement** — feature description
- **Confluence URL** — or "none"
- **Figma URL(s)** — or "none"
- **GitNexus available** — yes / no

---

## ⚠️ Sub-Agent Invocation Rule — Read This First

Every sub-agent in this workflow **must** be invoked using the `Agent` tool with `subagent_type: "general-purpose"`. **Never** use the `Skill` tool to invoke sub-agents.

**Pattern for every sub-agent call:**

Each sub-agent is defined in `.claude/agents/`. The orchestrator does **not** Read the agent file — instead, it tells the sub-agent where to find its own instructions. This keeps the orchestrator context clean.

```
Agent(
  subagent_type: "general-purpose",
  description: "[short description]",
  prompt: "Read `.claude/agents/[agent-file].md` for your full instructions, then execute with the ARGUMENTS below.\n\nARGUMENTS:\n[inputs for this agent]"
)
```

The sub-agent reads its own definition at the start of its fresh context. The orchestrator carries only the ARGUMENTS.

---

## Pre-load Phase — Run Before Everything Else

Before the complexity check or any phase:

### 1. Extract CODING_RULES_DIGEST
Read `.claude/skills/coding-standards/SKILL.md` once. Extract a condensed digest covering:
- Tailwind `sf-` prefix rule
- TypeScript naming conventions
- Component patterns and atomic design placement
- Do-Not list (common violations)
- Form/state/routing/API patterns

**Cap: 1000 tokens.** Store as `[CODING_RULES_DIGEST]`. Do not re-read coding-standards in any sub-agent — pass this digest instead.

### 2. Extract ARCH_DIGEST
Read `.claude/skills/project-architecture/SKILL.md` once. Extract a condensed digest covering:
- Component hierarchy (atoms/molecules/organisms/templates — what goes where)
- Routing conventions (TanStack Router, file-based routing, route groups)
- State management (Zustand store location, naming conventions, persist usage)
- API patterns (restClient location, service layer location, auth token handling)
- Key path aliases and env var conventions

**Cap: 500 tokens.** Store as `[ARCH_DIGEST]`. Do not re-read project-architecture in any sub-agent — pass this digest instead.

### 3. Set FIGMA_AVAILABLE flag
- Figma URL(s) provided → `FIGMA_AVAILABLE = "yes"`
- No Figma URL → `FIGMA_AVAILABLE = "no"`

Store as `[FIGMA_AVAILABLE]`. Pass to design and codegen agents.

---

## Step 1.5 — Complexity Check

**Simple** — ALL must be true: no URLs provided, change targets a single component/prop/state/file.

**Complex** — any of: has a URL, spans multiple files, new route, new store, or ambiguous scope.

→ Simple: skip Phase 1, go directly to Phase 1.5 with the raw requirement text.
→ Complex: run Phase 1 first.

---

## Phase 1 — Research *(complex only)*

Spawn the Research Agent:
- `subagent_type`: `"general-purpose"`
- `description`: `"Research: [feature name]"`
- `prompt`:
  ```
  Read `.claude/agents/sf-research-agent.md` for your full instructions, then execute with the ARGUMENTS below.

  ARGUMENTS:
  Requirement: [REQUIREMENT]
  Confluence URL: [URL OR "none"]
  Figma URL(s): [URL(S) OR "none"]
  ARCH_DIGEST: [ARCH_DIGEST]
  ```

Wait for the structured research summary before proceeding to Phase 1.5.

---

## Phase 1.5 — Requirements Alignment *(orchestrator — no subagent)*

Present your understanding directly to the user. **Do not proceed to Phase 2 until the user confirms.**

### Alignment block

```
## Requirements Alignment

**Feature:** [name]
**Goal:** [one-sentence synthesis]

**What I will build:**
- [Key functional thing]
- ...

**UI/UX scope:**
- Screens/views: [list or "inferred from requirement"]
- New components: [list or "none identified"]
- Modified components: [list or "none identified"]

**State and data scope:**
- New store: [yes — [name] / no]
- New API endpoints: [yes — [list] / no / unclear]
- Existing stores touched: [list or "none"]

**Assumptions — correct me if wrong:**
1. [Assumption]

**Open questions:**
1. [Question]

Reply with answers/corrections, or **confirmed** to proceed.
```

For **simple features** use the short version:
```
## Requirements Alignment

I understand you want to: [one sentence].
- File/component: [target]
- Change: [exact behavior change]
[One assumption if applicable]

Reply **confirmed** to proceed, or correct me.
```

### Question selection rules

Only ask where a genuine gap exists (1–4 questions, max 6):

| Signal | Question |
|---|---|
| Ambiguous scope ("update X") | "Does this replace existing X or add alongside it?" |
| Contradictory Figma frames | "Frame 1 shows [A], frame 2 shows [B] — which is current?" |
| New route, no nav context | "Should this appear in the sidebar? What label and position?" |
| Feature touches auth/permissions | "Any role-based visibility rules?" |
| Multiple "existing X" candidates | "Found [X1] and [X2] — which does this modify?" |
| Thin acceptance criteria | "What does 'done' look like at minimum?" |
| Complex UI, no Figma | "Should I infer UI from the text, or do you have a mockup?" |

### Iteration

Wait for user reply. Integrate corrections, re-present only changed sections, wait again. Exit when the user confirms (any natural confirmation: "confirmed", "yes", "looks good", "proceed", "approved").

### Handoff to Phase 2

Compile a **Confirmed Requirements block** to pass to the Design Agent:
```
**Confirmed Requirements:**
- Goal: [confirmed]
- Functional scope: [confirmed bullets]
- UI/UX scope: [confirmed]
- State/data scope: [confirmed]
- Resolved assumptions: [list]
- Q&A: [pairs]
```

---

## Phase 2 — Design Document

Spawn the Design Agent:
- `subagent_type`: `"general-purpose"`
- `model`: `"sonnet"`
- `description`: `"Design: [feature name]"`
- `prompt`:
  ```
  Read `.claude/agents/sf-design-agent.md` for your full instructions, then execute with the ARGUMENTS below.

  ARGUMENTS:
  [CONFIRMED REQUIREMENTS BLOCK]
  [RESEARCH SUMMARY OR RAW REQUIREMENT]
  CODING_RULES_DIGEST: [CODING_RULES_DIGEST]
  ARCH_DIGEST: [ARCH_DIGEST]
  FIGMA_AVAILABLE: [FIGMA_AVAILABLE]
  ```

Wait for the agent to return `DESIGN_PATH: ai-context/designs/[feature-name].md`.

Store as `[DESIGN_PATH]`. Derive `[ITERATION_STATE_PATH]` = `ai-context/iteration-state/[feature-name].json`.

---

## Phase 3 — Architecture Review *(human checkpoint)*

```
Design document written to: `[DESIGN_PATH]`

How would you like to review it?
- **here** — review inline in this conversation
- **pr** — create a pull request for team review
- **[feedback]** — provide feedback directly to revise now
```

**Wait.**

→ **here**: Tell the user to review `[DESIGN_PATH]` and reply **approved** or with feedback.

→ **pr**:
1. Read `.claude/skills/bitbucket-pr/SKILL.md`.
2. Spawn the PR agent using the `Agent` tool:
   - `subagent_type`: `"general-purpose"`
   - `description`: `"Create PR: [feature name]"`
   - `prompt`: Full content of the skill file, followed by:
     ```
     ARGUMENTS:
     BRANCH_NAME: design/[feature-name]
     FILES: [DESIGN_PATH]
     COMMIT_MESSAGE: design: [feature name] — architecture review
     PR_TITLE: Design: [feature name]
     PR_DESCRIPTION: Architecture design document for **[feature name]**.

     [1-2 sentence summary of what this feature covers]

     This is a design-only PR for team review before implementation begins. Review the design doc at `[DESIGN_PATH]` and approve or request changes.
     DESTINATION_BRANCH: main
     ```

When the agent returns `PR_URL`, tell the user:
```
PR created: [PR_URL]

Review the design doc and reply:
- **approved** — to proceed with implementation
- **[feedback]** — to revise specific sections
```

Wait for their reply.

→ **[feedback]** directly: spawn a Design Agent revision immediately:
- `subagent_type`: `"general-purpose"`
- `description`: `"Revise design: [feature name]"`
- `prompt`:
  ```
  Read `.claude/agents/sf-design-agent.md` for your full instructions, then execute with the ARGUMENTS below.

  ARGUMENTS:
  Read [DESIGN_PATH], apply this feedback: [FEEDBACK]. Preserve all other sections. Return DESIGN_PATH when done.
  CODING_RULES_DIGEST: [CODING_RULES_DIGEST]
  ARCH_DIGEST: [ARCH_DIGEST]
  FIGMA_AVAILABLE: [FIGMA_AVAILABLE]
  ```

Tell the user: "Updated. Re-review `[DESIGN_PATH]` and reply **approved** or further feedback."

Repeat until approved.

---

## Phase 4 — Initialize Iteration Tracking

Write to `[ITERATION_STATE_PATH]` (create `ai-context/iteration-state/` if needed):
```json
{ "iteration": 0, "max_iterations": 5, "status": "in_progress", "feature": "[FEATURE NAME]", "issues": [] }
```

Initialize `[FEEDBACK_HISTORY]` as an empty string. This will accumulate across Phase 7 iterations.

---

## Phase 4.5 — Context Compaction

Before spawning the codegen agent, declare a compact handoff. Everything from Phases 1–4 (research summary, requirements alignment thread, design agent output, user approval exchange) is now superseded by the approved design doc at `[DESIGN_PATH]`.

**From this point forward, carry only these values:**
- `[FEATURE_NAME]`
- `[DESIGN_PATH]`
- `[ITERATION_STATE_PATH]`
- `[CODING_RULES_DIGEST]`
- `[ARCH_DIGEST]`
- `[FIGMA_AVAILABLE]`
- `[FEEDBACK_HISTORY]` (empty at this point)

Do not reference or re-read any prior phase content. The design doc is the single source of truth from Phase 5 onward.

---

## Phase 5 — Code Generation

Spawn the Code Generation Agent:
- `subagent_type`: `"general-purpose"`
- `model`: `"sonnet"`
- `description`: `"Codegen: [feature name]"`
- `prompt`:
  ```
  Read `.claude/agents/sf-codegen-agent.md` for your full instructions, then execute with the ARGUMENTS below.

  ARGUMENTS:
  Design path: [DESIGN_PATH]
  CODING_RULES_DIGEST: [CODING_RULES_DIGEST]
  FIGMA_AVAILABLE: [FIGMA_AVAILABLE]
  ```

Wait for the files list — or surface BLOCKED to the user.

---

## Phase 6 — Parallel Validation

Launch **three agents simultaneously** in one message:

**Agent A** — Test Agent (`model: "sonnet"`):
- `subagent_type`: `"general-purpose"`
- `description`: `"Tests: [feature name]"`
- `prompt`:
  ```
  Read `.claude/agents/sf-test-agent.md` for your full instructions, then execute with the ARGUMENTS below.

  ARGUMENTS:
  Design path: [DESIGN_PATH]
  Feature name: [feature-name]
  ```

**Agent B** — Code Review Agent (`model: "haiku"`):
- `subagent_type`: `"general-purpose"`
- `model`: `"haiku"`
- `description`: `"Code review: [feature name]"`
- `prompt`:
  ```
  Read `.claude/agents/sf-code-review-agent.md` for your full instructions, then execute with the ARGUMENTS below.

  ARGUMENTS:
  Design path: [DESIGN_PATH]
  CODING_RULES_DIGEST: [CODING_RULES_DIGEST]
  ```

**Agent C** — Security Agent (`model: "haiku"`):
- `subagent_type`: `"general-purpose"`
- `model`: `"haiku"`
- `description`: `"Security review: [feature name]"`
- `prompt`:
  ```
  Read `.claude/agents/sf-security-agent.md` for your full instructions, then execute with the ARGUMENTS below.

  ARGUMENTS:
  Design path: [DESIGN_PATH]
  ```

---

## Phase 7 — Iteration Loop (Evaluator-Optimizer)

**Blocking:** test failures, code review `critical`/`major`, security `critical`/`major`.
`minor` = non-blocking (note, don't iterate).

**If no blocking issues:** go to Phase 8.

**If blocking:**

Maintain `[FEEDBACK_HISTORY]` — a running block that accumulates the record of every iteration. Update it after each fix+revalidation cycle:

```
Iteration [N]:
  Blocking issues: [list with file, line, severity]
  Fix attempted: [what the fix agent changed and why]
  Outcome: [what revalidation found — resolved / new failure / same failure]
```

This history is passed to the fix agent each iteration so it does not repeat failed approaches.

**For each iteration:**

1. Read `[ITERATION_STATE_PATH]`, increment `iteration`, write back.
2. If `iteration >= 5`: set `status: "max_iterations_reached"`, report to user, stop.

3. Spawn the Fix Agent:
   - `subagent_type`: `"general-purpose"`
   - `description`: `"Fix: [feature name] iteration [N]"`
   - `prompt`:
     ```
     Read `.claude/agents/sf-fix-agent.md` for your full instructions, then execute with the ARGUMENTS below.

     ARGUMENTS:
     Design path: [DESIGN_PATH]
     Current blocking issues: [LIST WITH FILE, LINE, DESCRIPTION, SEVERITY]
     Feedback history (prior attempts — do not repeat these):
     [FEEDBACK_HISTORY]
     ```

4. If STUCK returned: surface to user, stop.

5. Append to `[FEEDBACK_HISTORY]`:
   ```
   Iteration [N]:
     Blocking issues: [the issues list passed to the fix agent]
     Fix attempted: [summary of what the fix agent changed]
     Outcome: [pending — will update after revalidation]
   ```

6. Re-run only agents that had blocking issues:
   - Tests failed → re-run **Agent A + Agent B** (same prompt pattern as Phase 6)
   - Code review only → re-run **Agent B only**
   - Security issues → re-run **Agent C only** (iteration 1 only)

7. Update `[FEEDBACK_HISTORY]` for this iteration's `Outcome` field with what revalidation found.

Repeat until no blocking issues or max iterations reached.

---

## Phase 8 — Complete

1. Write `[ITERATION_STATE_PATH]`: `{ "status": "complete", ... }`

2. Report:
```
Feature implementation complete.

Iterations used: X / 5
Files created: [LIST]
Files modified: [LIST]
Tests: [N] passing
Minor issues (non-blocking): [LIST OR "none"]

Ready for your review.
```

3. Run notification (GitNexus reindex only if available):
```bash
osascript -e 'display notification "Feature is ready for your review" with title "start-feature complete" sound name "Glass"'; (npx gitnexus analyze > /dev/null 2>&1 && osascript -e 'display notification "Index refreshed for next session" with title "GitNexus reindex done" sound name "Glass"') &
```

If GitNexus unavailable:
```bash
osascript -e 'display notification "Feature is ready for your review" with title "start-feature complete" sound name "Glass"'
```
