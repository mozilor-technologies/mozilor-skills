---
name: setup-project
description: "One-time setup skill for the start-feature workflow on a new project. Explores the codebase, asks targeted questions, then generates the 4 project-specific skill files (project-architecture, coding-standards, figma-to-code, testing-standards). Run this once when onboarding a new project."
---

# setup-project — One-Time Onboarding

You are setting up the `start-feature` workflow for this project. Your job is to generate **4 project-specific skill files** by exploring the codebase and asking only what you cannot determine from the code.

---

## Step 1 — Check what already exists

Before doing anything else, check which of these files already exist:

- `.claude/skills/project-architecture/SKILL.md`
- `.claude/skills/coding-standards/SKILL.md`
- `.claude/skills/figma-to-code/SKILL.md`
- `.claude/skills/testing-standards/SKILL.md`

If **all 4 exist**, tell the user:
```
All 4 project skill files already exist. Reply:
- **regenerate** — to overwrite all of them
- **[file name]** — to regenerate only a specific one (e.g. "coding-standards")
- **cancel** — to stop
```
Wait for their reply and proceed accordingly.

If **some or none exist**, proceed to Step 2 for the missing ones only.

---

## Step 2 — Explore the codebase

Read these files to understand the project before asking any questions. Collect findings as you go — you will use them in Step 3.

### 2a. Project identity
- `package.json` — framework, key libraries, scripts, test runner
- `README.md` (if exists) — app description
- `CLAUDE.md` (if exists) — any existing guidance

### 2b. Folder structure
- Survey the top 2 levels of `src/` (or equivalent)
- Note how components are organized: atomic design, feature-based, flat, mixed

### 2c. Styling
- `tailwind.config.*` or `tailwind.config.js/ts` — custom prefix, color tokens, theme extensions
- Look at 2–3 existing components for className patterns

### 2d. Routing
- Look at the routes folder or pages folder
- Note any auth guard pattern

### 2e. State management
- Survey `src/stores/` or equivalent
- Note the store pattern (Zustand, Redux, Jotai, Context, etc.)

### 2f. API layer
- Survey `src/services/` or equivalent
- Note how API calls are structured

### 2g. Testing
- Check for `tests/`, `__tests__/`, `*.spec.*`, `*.test.*` files
- Check `playwright.config.*` or `vitest.config.*` or `jest.config.*`
- Check for any auth setup file in tests (e.g. `auth.setup.ts`)
- Check `.env` or `.env.example` for any variables that look test-specific (e.g. `TEST_USER_EMAIL`, `TEST_USER_PASSWORD`, `PLAYWRIGHT_BASE_URL`) — record the exact variable names

### 2h. Design tokens
- Check `tailwind.config.*` for a custom color palette
- Check `src/styles/` or `src/theme/` for any token definitions
- Look at 1–2 components to see what color/spacing classes are actually used

### 2i. Icons
- Check imports in existing components for icon library (lucide-react, react-icons, heroicons, etc.)

### 2j. Existing reusable components
First, detect how the project organizes its components — it may use atomic design (`atoms/`, `molecules/`, `organisms/`), feature folders (`features/[name]/components/`), a flat `components/` directory, or a mix. Adapt your exploration to whatever structure exists.

**For every component directory you find:**
- Glob all files in it (do not sample — list every file)
- For each component: note the file name, its exported component name(s), and its primary purpose in one line
- Flag any components that are easy to duplicate by mistake — look for multiple components serving the same need (e.g. two separate dropdown implementations, two modal patterns, two pagination components)

---

## Step 3 — Ask only what you could not determine

In a single message, ask only about genuine gaps. Do not ask about things you found in Step 2.

Organize by file. Skip any group where you have enough information already.

```
I've explored the codebase. Here's what I found:
[Brief summary: framework, styling approach, state, test runner, icon library]

I need a few clarifications before generating the skill files:

**For project-architecture:**
[Ask only if unclear: app purpose, non-obvious routing patterns, analytics libraries]

**For coding-standards:**
[Ask only if unclear: forbidden patterns not obvious from code, error handling convention, data-testid naming]

**For figma-to-code:**
[Ask only if the project uses Figma AND tokens aren't clear from tailwind.config: primary brand colors + token names, card border radius, most important reusable components to prefer]
[If no Figma: "Does the project use Figma? If not, I'll skip figma-to-code."]

**For testing-standards:**
[Ask only if unclear: auth setup for tests, required env vars, minimum coverage expectations]
```

Wait for the user's reply before proceeding.

---

## Step 4 — Generate and write the 4 skill files

Use all information from Steps 2 and 3. Write each file directly — do not ask for further confirmation.

Create directories if they don't exist. Only generate the files that were missing or requested for regeneration.

---

### File 1: `.claude/skills/project-architecture/SKILL.md`

```markdown
---
name: project-architecture
description: "Reference for the [APP NAME] app architecture. Use before designing or implementing any new feature to understand tech stack, component hierarchy, routing, state management, and API patterns."
---

# Architecture Overview — [APP NAME]

## What This App Is
[1–2 sentences: purpose and target users]

## Tech Stack
| Layer | Technology |
|-------|-----------|
| Framework | [...] |
| Bundler | [...] |
| Routing | [...] |
| State | [...] |
| Server State | [...] |
| Forms | [...] |
| UI Primitives | [...] |
| Styling | [...] |
[Add rows for any other notable libraries]

## Component / Folder Structure
[Describe how src/ is organized with the actual top-level tree and one-line descriptions]

**Rule:** [Describe the key structural rule, e.g. "Never skip atomic design levels" or "Keep all feature code inside features/[name]/"]

## Routing
[Describe routing approach, file structure, auth guards, how new routes are added]

## State Management
[Describe where stores live, naming conventions, when to use global vs. local state, persistence]

## API Communication
[Describe the base client, service layer, auth token handling, error handling, where new API calls go]

## Environment Variables
[List key env vars with purpose]

## Path Aliases
[List import aliases]

## Analytics / Tracking
[If applicable — events new features must fire]
```

---

### File 2: `.claude/skills/coding-standards/SKILL.md`

```markdown
---
name: coding-standards
description: "Mandatory coding standards for [APP NAME]. Invoke before writing any code. Covers [CSS approach] conventions, TypeScript rules, component patterns, naming, error handling, and do-nots."
---

# Coding Standards — [APP NAME]

## CSS / Styling — CRITICAL RULE
[State the primary rule clearly. Include ✅ correct / ❌ wrong example.]

[List any custom token usage rules]

## TypeScript
- [Rule 1]
- [Rule 2]
- [...]

## Component Patterns

### File naming
- Components: [convention]
- Hooks: [convention]
- Services: [convention]
- Stores: [convention]

### Component structure
[Minimal code example of a well-structured component]

### Component organization
[When something is an atom vs molecule vs organism, or a component vs a page, etc.]

## Forms
[Form pattern with code example]

## State Management
[Store pattern with code example]

## Error Handling — CRITICAL RULE
[How errors must be handled in catch blocks. Be explicit.]

## API / Service Layer
[Service pattern with code example]

## Routing
[Minimal new route example]

## Imports
[Alias conventions with ✅ / ❌ examples]

## Test IDs
[data-testid conventions]

## Do Not
- Do not [...]
- Do not [...]
```

---

### File 3: `.claude/skills/figma-to-code/SKILL.md`

> If the project does not use Figma, write:
> ```markdown
> ---
> name: figma-to-code
> description: "This project does not use Figma. Skip this skill."
> ---
> This project does not use Figma for design handoff. Designs are [described in tickets / provided as screenshots / etc.].
> ```

Otherwise:

```markdown
---
name: figma-to-code
description: "Authoritative guide for translating Figma designs to [APP NAME] code. Invoke before writing any UI from Figma. Covers color tokens, typography, spacing, radius, icons, layout, and component reuse."
---

# Figma → Code Integration Guide

## Pre-Code Checklist
- [ ] Colors translated to tokens — no raw hex/rgb/hsl
- [ ] Font sizes mapped to [CSS framework] classes
- [ ] Spacing mapped correctly
- [ ] Border radius mapped to tokens
- [ ] Icons from [ICON LIBRARY] — no inline SVGs
- [ ] Absolute layout converted to flex/grid
- [ ] Existing components checked for reuse
- [ ] No inline styles

---

## Color Token Map

[For each color group:]
### [Group name]
| Token | Hex | Usage |
|---|---|---|
| `[token]` | `#xxx` | [when] |

### Quick-Reference
```
[hex] → [token class]
```

### Color Opacity
[How to apply opacity — Tailwind `/` modifier or equivalent]

---

## 1. Typography
### Font size (px → class)
| Figma px | Class |
|---|---|
| [px] | [class] |

### Font weight
| Figma weight | Class |
|---|---|
| 400 | [class] |

### Typography components to prefer
[List text atoms/components to use instead of raw elements]

---

## 2. Spacing
[Scale description]
| Figma px | Class |
|---|---|

---

## 3. Border Radius
| Figma px | Class |
|---|---|
> **Cards/panels:** use `[exact class]`

---

## 4. Shadows
| Description | Class |
|---|---|

---

## 5. Icons
**Library: [NAME]** — only this library. No inline SVGs.
[Usage code example]

---

## 6. Layout
[Figma auto-layout → flex/grid mapping table]

---

## 7. Images / Media
[How to reference project-hosted assets]

---

## 8. Existing Components — Reuse Before Creating

### [Base Components]
| Need | Use |
|---|---|

### [Composite Components]
| Need | Use |
|---|---|

### Critical patterns
[Common wrong vs. right choices and why]

---

## 9. Fidelity Rules
- Never add UI elements not in Figma
- Match left-to-right element order exactly
- Copy text verbatim
[Any project-specific rules]
```

---

### File 4: `.claude/skills/testing-standards/SKILL.md`

```markdown
---
name: testing-standards
description: "[FRAMEWORK] testing standards for [APP NAME]. Invoke before writing tests. Covers test structure, selector priority, auth setup, mocking, assertions, and coverage goals."
---

# Testing Standards — [APP NAME]

## Framework
[Framework name and type: E2E / component / unit]

## Test Location
[Folder structure with example]

## Running Tests
```bash
[commands]
```

## Test Structure
[Minimal well-structured test example]

## Selectors — Priority Order
1. [...]
2. [...]
3. [...]

## Required Environment Variables
These must be present in `.env` before running tests — the test agent checks for these before writing any test:
- `[VAR_NAME]` — [purpose]
- `[VAR_NAME]` — [purpose]

> If no test-specific env vars are needed, write: "No additional env vars required beyond the app's standard `.env`."

## Auth in Tests
[How auth is handled — shared setup, how state is loaded]

## Mocking API Calls
[Minimal mock example]

## Assertions Best Practices
[✅ correct / ❌ wrong examples]

## Test Naming Convention
[File name, describe block, test name format]

## Coverage Goals — All Mandatory
- [Requirement 1]
- [Requirement 2]
- At least [N] edge cases
- At least [N] error states
```

---

## Step 5 — Report

After writing all files, tell the user:

```
Project setup complete.

Files written:
- .claude/skills/project-architecture/SKILL.md
- .claude/skills/coding-standards/SKILL.md
- .claude/skills/figma-to-code/SKILL.md
- .claude/skills/testing-standards/SKILL.md

Assumptions to verify:
[List any values you inferred that the user should double-check]

You're ready to use /start-feature.
```
