---
name: claude-md
description: Create or review a CLAUDE.md file following LLM onboarding best practices
when-to-use: When creating a new CLAUDE.md, or reviewing/auditing an existing one for quality
user-invocable: true
allowed-tools: [Read, Write, Edit, Glob, Grep, Bash]
effort: medium
---

# CLAUDE.md Skill

Guides creation of high-quality `CLAUDE.md` files and reviews existing ones against proven best practices. Operates in two modes — auto-detected from context.

---

## Mode Detection

| Condition | Mode |
|---|---|
| No `CLAUDE.md` exists in project root | **Creation** |
| `CLAUDE.md` exists | **Review** |
| User explicitly asks to regenerate/rewrite | **Creation** (override) |

---

## Core Principles

These apply to both modes. A great `CLAUDE.md`:

1. **Answers WHAT, WHY, HOW** — tech stack + structure, project purpose, key commands
2. **Is short** — target under 60 lines; hard cap 300. Every line appears in every session.
3. **Is universally relevant** — nothing task-specific that distracts during unrelated work
4. **Uses progressive disclosure** — detailed docs live in `agent_docs/` (or equivalent); CLAUDE.md points there
5. **Delegates style enforcement** — no code style rules; use linters (Biome, Prettier, Ruff, etc.)
6. **Is hand-crafted** — never auto-generated; this is the highest-leverage config in the harness
7. **Points, doesn't embed** — reference `file:line` locations rather than copying code snippets (snippets go stale)

> **Why brevity matters:** Claude Code's system prompt already contains ~50 instructions. LLMs can reliably follow ~150-200 total. Every line you add competes with Claude's own instructions. Instruction-following quality degrades as count grows.

---

## Creation Mode

Interview the developer one question at a time. Do not ask multiple questions at once.

### Interview Questions (in order)

1. **WHY** — What does this project do and why does it exist? (1-2 sentences)
2. **WHAT: stack** — What is the tech stack? (language, framework, database, key libraries)
3. **WHAT: structure** — What are the key directories and what does each contain?
4. **HOW: commands** — What commands are used to run, test, build, lint, and type-check?
5. **Conventions** — Any non-obvious patterns, naming rules, or architectural constraints Claude must know?
6. **Never do** — What should Claude never do in this project? (e.g. never modify migrations directly, never use ORM X)
7. **Agentic workflow** — Which skills does this project use? Where do the skill files live?
8. **Non-negotiables** — What are the mandatory steps for this project's workflow that cannot be skipped?
9. **Reference docs** — Are there detailed docs Claude should read before starting work? (architecture, schema, etc.)

### After the Interview

Generate the `CLAUDE.md` using the template below. Then:

- Count lines. If over 60, identify what to move to `agent_docs/`.
- Check that no code style rules snuck in.
- Present the file to the user for approval before writing.

### Template

```markdown
# [Project Name]

## Purpose
[1-2 sentences: what this project does and why it exists]

## Tech Stack
- [language + version]
- [framework]
- [database]
- [key libraries]

## Project Structure
- `src/` — [what lives here]
- `tests/` — [what lives here]
- [other key dirs]

## Commands
```bash
# Run
[command]

# Test
[command]

# Build
[command]

# Lint / Type-check
[command]
```

## Conventions
- [Non-obvious rule 1]
- [Non-obvious rule 2]

## Agentic Workflow
Skills used: [list skill names or paths]
Skill files: [path to skills directory]
Workflow: [brief description — e.g. "all tasks go through brainstorm → plan → implement → review"]

## Non-Negotiables
The following steps are mandatory and cannot be skipped:
1. [step]
2. [step]

## Never Do
- [Constraint 1]
- [Constraint 2]

## Reference Docs
Before starting work, read relevant files in `agent_docs/`:
- `agent_docs/architecture.md` — [what it covers]
- `agent_docs/database-schema.md` — [what it covers]
```

---

## Review Mode

Read the existing `CLAUDE.md` and score it against the checklist below. Output:
1. A scored report with specific line callouts
2. A prioritized list of improvements
3. A rewrite suggestion for any flagged sections

### Review Checklist

**Length**
- [ ] Under 60 lines *(target)*
- [ ] Under 300 lines *(hard cap — fail if exceeded)*

**Coverage**
- [ ] WHY — project purpose is stated
- [ ] WHAT — tech stack is listed
- [ ] WHAT — key folder structure is described
- [ ] HOW — run/test/build/lint commands are present

**Quality**
- [ ] No code style rules (formatting, naming conventions) — these belong in linter config
- [ ] No task-specific instructions that distract during unrelated work
- [ ] No embedded code snippets that can go stale — use `file:line` references instead
- [ ] No auto-generated boilerplate (generic headings, filler sentences)
- [ ] Uses progressive disclosure — detailed docs live elsewhere and are referenced

**Agentic workflow**
- [ ] Agentic workflow context is present (skills used, workflow philosophy)
- [ ] Non-negotiables are documented
- [ ] Skill file locations are referenced, not their full contents

### Scoring

| Issues found | Rating |
|---|---|
| 0 | Excellent |
| 1-2 minor | Good |
| 3-4 | Needs improvement |
| 5+ or hard cap exceeded | Requires rewrite |

---

## agent_docs/ Strategy

If the CLAUDE.md is growing too large, recommend splitting content into:

```
agent_docs/
  architecture.md       — system design, component relationships
  commands.md           — detailed run/test/build instructions
  conventions.md        — coding patterns, naming rules, anti-patterns
  database-schema.md    — schema overview, key tables/models
  service-dependencies.md — external services, environment variables
```

Instruct Claude to identify and read the relevant file(s) before beginning work.
Add this to the `CLAUDE.md`:

```markdown
## Reference Docs
Before starting work, identify and read relevant files in `agent_docs/`.
```

---

## Anti-Patterns to Flag

- ❌ Code style rules ("use camelCase", "always add JSDoc") — use a linter
- ❌ Copying architecture diagrams or schema directly — link to `agent_docs/`
- ❌ Task-specific instructions ("when building the auth module, remember to...") — use slash commands or task-level context
- ❌ Overly long "personality" or "tone" instructions — waste of instruction budget
- ❌ Auto-generated content (identifiable by generic structure and filler text)
- ❌ Duplicate information already in skill files — point to the skill, don't repeat it
