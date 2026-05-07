---
description: "One-time onboarding for the start-feature workflow. Auto-detects frontend, backend, or fullstack. Explores the codebase, asks targeted questions, then generates project-specific skill files that all agents rely on."
argument-hint: "[optional: regenerate | cancel | specific-skill-name]"
---

# /setup-project — One-Time Onboarding

Generate project-specific skill files by exploring the codebase and asking only what cannot be determined from the code.

**Frontend projects** get 4 files: `project-architecture`, `coding-standards`, `figma-to-code`, `testing-standards`
**Backend projects** get 4 files: `project-architecture`, `coding-standards`, `api-contracts`, `testing-standards`
**Fullstack projects** get 5 files: all of the above

---

## Step 1 — Check what already exists

Check which of these files already exist:
- `.claude/skills/project-architecture/SKILL.md`
- `.claude/skills/coding-standards/SKILL.md`
- `.claude/skills/figma-to-code/SKILL.md`
- `.claude/skills/api-contracts/SKILL.md`
- `.claude/skills/testing-standards/SKILL.md`

If **all relevant files exist**, tell the user:
```
All project skill files already exist. Reply:
- **regenerate** — to overwrite all
- **[file name]** — to regenerate only one (e.g. "coding-standards")
- **cancel** — to stop
```
Wait for reply. If some/none exist, proceed to Step 2 for missing ones only.

---

## Step 2 — Auto-Detect Stack

Read the following files using the Read tool:

| File | What to detect |
|------|---------------|
| `package.json` | Framework, key libraries, scripts, test runner |
| `pyproject.toml` | Python framework, test runner |
| `requirements.txt` | Python packages |
| `go.mod` | Go modules |
| `composer.json` | PHP packages |
| `README.md` (if exists) | Project description |
| `CLAUDE.md` (if exists) | Project overrides |

**Frontend indicators** (package.json deps): `react`, `next`, `vue`, `@angular/core`, `svelte`, `solid-js`, `@remix-run`, `preact`, `gatsby`, `astro`

**Backend Node.js indicators**: `express`, `fastify`, `koa`, `@nestjs/core`, `hono`

**Python backend**: `fastapi`, `django`, `flask`, `starlette` in requirements

**Go backend**: go.mod exists

**PHP backend**: composer.json with `laravel/framework`

Set **[STACK]** = `frontend` / `backend` / `fullstack`
Set **[BACKEND_LANG]** = `nodejs` / `python` / `go` / `php` / `none`

### Shopify Remix sub-detection

After STACK is set, also check `package.json` dependencies for `@shopify/shopify-app-remix`.

- If present: set **[SHOPIFY_REMIX]** = `yes`. Treat the project as `fullstack` (Remix loaders/actions = backend; Polaris React = frontend).
- Else: set **[SHOPIFY_REMIX]** = `no`.

---

## Step 3 — Explore the codebase

### 3a. Folder structure
- Survey top 2 levels of `src/` (or equivalent: `app/`, `lib/`, etc.)
- Note how code is organized: feature-based, layer-based (controllers/services/repos), atomic design, flat, etc.

### 3b. Styling (frontend/fullstack only)
- `tailwind.config.*` — custom prefix, color tokens, theme extensions
- 2–3 existing components for className patterns

### 3c. Routing
- Routes/pages folder, auth guard pattern
- For backend: route file structure, middleware chain

### 3d. State management (frontend/fullstack only)
- `src/stores/` or equivalent — store pattern (Zustand, Redux, Jotai, Pinia, etc.)

### 3e. API layer
- `src/services/` or equivalent (frontend: how API calls are structured)
- `src/routes/` + `src/controllers/` + `src/services/` (backend: layer structure)

### 3f. Testing
- `tests/`, `__tests__/`, `*.spec.*`, `*.test.*` files
- `playwright.config.*`, `vitest.config.*`, `jest.config.*`, `pytest.ini`, `go test`, `phpunit.xml`
- Auth setup files in tests
- `.env` or `.env.example` for test-specific variables — record exact names

### 3g. Design tokens (frontend/fullstack only)
- `tailwind.config.*` for custom color palette
- `src/styles/` or `src/theme/`
- 1–2 components to see what color/spacing classes are used

### 3h. Icons (frontend/fullstack only)
- Check imports for icon library (lucide-react, react-icons, heroicons, etc.)

### 3i. API contracts (backend/fullstack only)
- Read 2–3 existing route files to understand the endpoint structure
- Note: HTTP method conventions, request/response patterns, error format, auth middleware
- Check for OpenAPI/Swagger docs if present

### 3j. Existing reusable components/services
For every component or service directory:
- Glob all files (do not sample — list every file)
- Note: file name, exported name, primary purpose in one line
- Flag names easy to duplicate by mistake

---

## Step 4 — Ask only what you could not determine

In a single message, ask only about genuine gaps:

```
I've explored the codebase. Here's what I found:
[Brief summary: stack, framework, styling approach, state, test runner, language]

I need a few clarifications:

**For project-architecture:**
[Only if unclear: app purpose, non-obvious routing patterns, analytics libraries]

**For coding-standards:**
[Only if unclear: forbidden patterns not visible in code, error handling convention, naming rules]

[For frontend/fullstack] **For figma-to-code:**
[Only if Figma AND tokens aren't clear: primary brand colors + token names, card border radius]
[If no Figma: "Does the project use Figma? If not, I'll skip figma-to-code."]

[For backend/fullstack] **For api-contracts:**
[Only if unclear: versioning strategy, pagination format, auth token format]

**For testing-standards:**
[Only if unclear: auth setup for tests, required env vars, minimum coverage expectations]
Do NOT ask about vitest — whether it is present or absent is detected automatically in Step 5.5 and handled without user input.
```

Wait for reply before proceeding.

---

## Step 5 — Generate and write skill files

Use all information from Steps 3 and 4. Write each file directly — no further confirmation needed.

Create directories if missing. Only generate files that were missing or requested for regeneration.

### File 1 (all stacks): `.claude/skills/project-architecture/SKILL.md`

```markdown
---
name: project-architecture
description: "Reference for the [APP NAME] [stack] architecture. Use before designing or implementing any new feature."
---

# Architecture Overview — [APP NAME]

## What This App Is
[1–2 sentences: purpose, domain, target users]

## Tech Stack
| Layer | Technology |
|-------|-----------|
| Framework | |
| [Frontend: Routing] | |
| [Frontend: State] | |
| [Frontend: UI Primitives] | |
| Styling | |
| [Backend: Database] | |
| [Backend: ORM/Query builder] | |
| [Backend: Task queue] | |

## [Frontend] Component / [Backend] Module Structure
[Top-level src/ tree with one-line descriptions]

**Rule:** [Key structural rule — e.g. "feature-based: all code for a feature lives together"]

## Routing
[Approach, file structure, auth guards, how new routes are added]

## [Frontend] State Management / [Backend] Service Layer
[Locations, naming, when to use, patterns]

## API Communication
[Base client, service layer, auth tokens, error handling, where new API calls go]

## Environment Variables
[Key env vars with purpose]

## Path Aliases
[Import aliases — e.g. @/ → src/]
```

### File 2 (all stacks): `.claude/skills/coding-standards/SKILL.md`

```markdown
---
name: coding-standards
description: "Mandatory coding standards for [APP NAME]. Invoke before writing any code."
---

# Coding Standards — [APP NAME]

[Frontend] ## CSS / Styling — CRITICAL RULE
[Primary rule with ✅ correct / ❌ wrong example]

## [Language] Rules
- [Rules]

## Component / Module Patterns
### File naming
- [Conventions]

### Structure
[Minimal well-structured example]

## Error Handling — CRITICAL RULE
[Explicit rule: how errors are caught, logged, surfaced]

## [Backend] API Layer
[Service + repository pattern with example]

## Imports
[Alias conventions with ✅ / ❌]

[Frontend] ## Test IDs
[data-testid conventions]

## Do Not
- Do not [...]
```

### File 3a (frontend/fullstack): `.claude/skills/figma-to-code/SKILL.md`

If no Figma used:
```markdown
---
name: figma-to-code
description: "This project does not use Figma. Skip this skill."
---
This project does not use Figma for design handoff.
```

Otherwise:
```markdown
---
name: figma-to-code
description: "Authoritative guide for translating Figma designs to [APP NAME] code."
---

# Figma → Code Integration Guide

## Pre-Code Checklist
- [ ] Colors translated to tokens — no raw hex/rgb/hsl
- [ ] Font sizes mapped to classes
- [ ] Border radius mapped to tokens
- [ ] Icons from [ICON LIBRARY] — no inline SVGs
- [ ] Existing components checked for reuse
- [ ] No inline styles

## Color Token Map
[Groups with Token / Hex / Usage table]

### Quick-Reference
[hex] → [token class]

## Typography
| Figma px | Class |

## Spacing / Border Radius
| Figma px | Class |

## Icons
**Library: [NAME]** — only this library.

## Existing Components — Reuse Before Creating
| Need | Use |

## Fidelity Rules
- Never add UI elements not in Figma
- Match left-to-right element order exactly
- Copy text verbatim
```

### File 3b (backend/fullstack): `.claude/skills/api-contracts/SKILL.md`

```markdown
---
name: api-contracts
description: "API design conventions and patterns for [APP NAME] backend. Use before adding any new endpoint or service."
---

# API Contracts — [APP NAME]

## Base URL
`/api/[version]/`

## Auth Pattern
[How requests are authenticated — e.g. Bearer token, session cookie, API key]
[Which middleware/decorator is used — e.g. @Auth(), requireAuth(), middleware('auth')]

## Request Conventions
- Content-Type: application/json
- [Pagination approach: offset / cursor]
- [How query filters are passed]

## Response Format
```json
{
  "data": {},
  "meta": { "page": 1, "total": 100 }
}
```

## Error Format
```json
{
  "error": "Human-readable message",
  "code": "MACHINE_CODE",
  "details": {}
}
```

## HTTP Status Code Usage
| Code | When to use |
|------|------------|
| 200 | Success |
| 201 | Created |
| 400 | Validation error |
| 401 | Not authenticated |
| 403 | Not authorized |
| 404 | Resource not found |
| 422 | Business logic error |
| 500 | Unexpected server error |

## Versioning
[How API versions are managed]

## Existing Endpoints — Reuse Patterns
| Resource | Endpoints | Auth |
|----------|-----------|------|
| [resource] | GET /... POST /... | [auth] |
```

### File 4 (all stacks): `.claude/skills/testing-standards/SKILL.md`

```markdown
---
name: testing-standards
description: "[FRAMEWORK] testing standards for [APP NAME]. Covers test structure, selectors, auth, mocking, assertions, and coverage goals."
---

# Testing Standards — [APP NAME]

## Framework
[Framework name and type — e.g. Playwright E2E, Vitest unit, pytest, go test]

## Test Location
[Folder structure]

## Running Tests
```bash
[commands]
```

## Selectors — Priority Order (frontend)
1. [...]

## Required Environment Variables
- `[VAR_NAME]` — [purpose]

## Auth in Tests
[How auth is handled — e.g. storageState, test user fixture, API token]

## Assertions Best Practices
[✅ correct / ❌ wrong]

## Coverage Goals
- [Requirements]
```

---

## Step 5.5 — Append Unit Testing section to testing-standards (frontend/fullstack only)

Skip this step for pure backend stacks.

Run these two Bash commands now to detect vitest:

```bash
grep -q '"vitest"' package.json 2>/dev/null && echo "vitest_in_pkg=true" || echo "vitest_in_pkg=false"
```
```bash
ls vitest.config.* 2>/dev/null && echo "vitest_config=true" || echo "vitest_config=false"
```

Wait for the results, then:

**If both returned true** — also run:
```bash
node -e "const s=require('./package.json').scripts||{}; const k=Object.keys(s).find(k=>s[k].includes('vitest')); console.log(k ? 'npm run '+k : 'npx vitest run')"
```
to resolve the run command.

Then use the **Edit tool** to append this block to `.claude/skills/testing-standards/SKILL.md` (add a blank line before it if the file doesn't already end with one):

```
## Unit Testing

**Status:** enabled
**Framework:** vitest
**Run command:** [resolved run command from above]
**Test file convention:** [co-located *.test.ts(x) next to source — OR — __tests__/ folder mirroring src/ — pick whichever already exists in the project; default to co-located]

Every new or modified exported unit (function, hook, reducer, service, pure utility) must have a test file with at least one positive case and at least one negative case per changed unit. Tests are grouped under `'positive cases'` and `'negative cases'` sub-describes inside a `describe('[unit]')` block. The `/start-feature` orchestrator runs the unit-test stage automatically when the design doc sets `unit_tests_required: true`.
```

**If either returned false** — determine the framework from what was detected in Step 2 (React, Vue, plain TS, etc.), then:
1. Append the appropriate setup guide to `.claude/skills/testing-standards/SKILL.md` (so it's saved for reference).
2. **Also print the same setup steps directly in the conversation** so the developer sees them immediately without opening the file. Use this format in your response:

```
⚠️ Vitest is not configured in this project. The unit-test stage in `/start-feature` will be skipped until you set it up.

Here's how to enable it:
[paste the same steps you wrote to testing-standards]

Once done, run `/setup-project testing-standards` to re-detect and enable the unit-test stage.
```

**For React projects**, append:

```
## Unit Testing

**Status:** disabled — setup required

Vitest is not configured. The `/start-feature` orchestrator skips the unit-test stage until setup is complete.

### Step 1 — Install dependencies
```bash
npm install --save-dev vitest jsdom @testing-library/react @testing-library/user-event @testing-library/jest-dom @vitejs/plugin-react
```

### Step 2 — Create `vitest.config.ts` at the project root
```ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/__tests__/setup.ts'],
  },
})
```

### Step 3 — Create `src/__tests__/setup.ts`
```ts
import '@testing-library/jest-dom'
```

### Step 4 — Add scripts to `package.json`
```json
"test:unit": "vitest run",
"test:unit:watch": "vitest"
```

### Step 5 — Re-run setup
Once the above is done, run:
```
/setup-project testing-standards
```
This will re-detect vitest and update this section to enabled.
```

**For Vue projects**, append:

```
## Unit Testing

**Status:** disabled — setup required

Vitest is not configured. The `/start-feature` orchestrator skips the unit-test stage until setup is complete.

### Step 1 — Install dependencies
```bash
npm install --save-dev vitest jsdom @testing-library/vue @testing-library/user-event @testing-library/jest-dom @vitejs/plugin-vue
```

### Step 2 — Create `vitest.config.ts` at the project root
```ts
import { defineConfig } from 'vitest/config'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/__tests__/setup.ts'],
  },
})
```

### Step 3 — Create `src/__tests__/setup.ts`
```ts
import '@testing-library/jest-dom'
```

### Step 4 — Add scripts to `package.json`
```json
"test:unit": "vitest run",
"test:unit:watch": "vitest"
```

### Step 5 — Re-run setup
Once the above is done, run:
```
/setup-project testing-standards
```
This will re-detect vitest and update this section to enabled.
```

**For all other frontend/fullstack projects (plain TypeScript, Next.js without React Testing Library, etc.)**, append:

```
## Unit Testing

**Status:** disabled — setup required

Vitest is not configured. The `/start-feature` orchestrator skips the unit-test stage until setup is complete.

### Step 1 — Install dependencies
```bash
npm install --save-dev vitest
```
Add `jsdom` if you need DOM APIs: `npm install --save-dev jsdom`

### Step 2 — Create `vitest.config.ts` at the project root
```ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    environment: 'node', // change to 'jsdom' if testing DOM code
    globals: true,
  },
})
```

### Step 3 — Add scripts to `package.json`
```json
"test:unit": "vitest run",
"test:unit:watch": "vitest"
```

### Step 4 — Re-run setup
Once the above is done, run:
```
/setup-project testing-standards
```
This will re-detect vitest and update this section to enabled.
```

---

## Step 5.6 — Shopify Remix CLAUDE.md guidance

Run only if **[SHOPIFY_REMIX]** = `yes`. Otherwise skip to Step 6.

Read `CLAUDE.md` at the project root if it exists.

The managed section is fenced by:
```
<!-- mozilor:shopify-remix:start -->
...
<!-- mozilor:shopify-remix:end -->
```

Behavior:
- If `CLAUDE.md` does not exist: create it containing only the managed section below.
- If `CLAUDE.md` exists and contains the fenced section: replace the contents between the markers.
- If `CLAUDE.md` exists but has no fenced section: append the managed section to the end (preceded by a blank line).

**Never** modify content outside the fenced markers.

Section content (write verbatim, including the markers):

```
<!-- mozilor:shopify-remix:start -->
## Shopify Remix App Conventions (managed by mozilor-skills)

This project supports projects that uses`@shopify/shopify-app-remix` for building embedded admin apps in Shopify.

- **UI**: Use Shopify Polaris React components from `@shopify/polaris` — the bundled `shopify-polaris-react` skill has the canonical patterns. Do NOT use Polaris web components (`s-page`, `s-button`, `s-card`, etc.) in the app shell; those belong to UI extensions only (see `shopify-polaris-admin-extensions` for files under `extensions/`).
- **Auth**: Use `authenticate.admin(request)` from `app/shopify.server`. Never roll your own session validation.
- **API**: Use `admin.graphql()` for Admin API calls. Do not call `fetch()` against `*.myshopify.com` directly.
- **Webhooks**: HMAC verification is handled by `authenticate.webhook(request)`. Do not reimplement signature checking.
- **Mixed-pattern files**: If a file already uses an older pattern, match the surrounding file. Do not introduce new patterns mid-file.
<!-- mozilor:shopify-remix:end -->
```

---

## Step 6 — Report

```
Project setup complete.

Stack detected: [STACK]
[If [SHOPIFY_REMIX] = "yes": "Shopify Remix detected — managed section written to CLAUDE.md."]
Files written:
[List of files actually written]

Assumptions to verify:
[List any inferred values to double-check — colors, test user credentials, env var names]

You're ready to use /start-feature.
```
