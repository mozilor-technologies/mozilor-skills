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
| `shopify.app.toml` | Shopify app (check if file exists) |
| `package.json` | Framework, key libraries, scripts, test runner |
| `pyproject.toml` | Python framework, test runner |
| `requirements.txt` | Python packages |
| `go.mod` | Go modules |
| `composer.json` | PHP packages |
| `README.md` (if exists) | Project description |
| `CLAUDE.md` (if exists) | Project overrides |

**Shopify indicator**: `shopify.app.toml` exists at project root → set **[IS_SHOPIFY]** = `true`, **[STACK]** = `fullstack`, **[BACKEND_LANG]** = `nodejs`. Skip remaining stack detection.

**If not Shopify:**

**Frontend indicators** (package.json deps): `react`, `next`, `vue`, `@angular/core`, `svelte`, `solid-js`, `@remix-run`, `preact`, `gatsby`, `astro`

**Backend Node.js indicators**: `express`, `fastify`, `koa`, `@nestjs/core`, `hono`

**Python backend**: `fastapi`, `django`, `flask`, `starlette` in requirements

**Go backend**: go.mod exists

**PHP backend**: composer.json with `laravel/framework`

Set **[STACK]** = `frontend` / `backend` / `fullstack`
Set **[BACKEND_LANG]** = `nodejs` / `python` / `go` / `php` / `none`
Set **[IS_SHOPIFY]** = `false`

### Shopify Plugin Check *(IS_SHOPIFY only)*

If **[IS_SHOPIFY]** = `true`, check whether the Shopify AI Toolkit plugin is installed and enabled:
```bash
claude plugin list | grep -A3 "shopify-plugin" | grep "✔ enabled"
```

If the command returns no output (plugin missing or disabled), warn the user:
```
⚠️  Shopify plugin not detected. For best results, install it:

  /plugin marketplace add Shopify/shopify-ai-toolkit
  /plugin install shopify-plugin@shopify-plugin

Continuing without it — skill files will still be generated from your codebase.
```

Then continue to Step 3.

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

### 3i-shopify. Shopify-specific exploration *(IS_SHOPIFY only)*

- `shopify.app.toml` — app name, scopes, auth strategy
- `app/shopify.server.js` (or `.ts`) — auth setup, session storage, API client config
- `app/routes/` — list all route files; note which use `authenticate.admin()`, `authenticate.storefront()`, or webhook handlers
- `extensions/` — list all extensions (UI extensions, Functions, Theme extensions) with their type and purpose
- `app/db.server.*` — ORM and session model if present
- Note: which Shopify APIs are used (Admin GraphQL, Storefront GraphQL, REST)
- Note: any Polaris component imports in existing route files

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
```

Wait for reply before proceeding.

---

## Step 5 — Generate and write skill files

Use all information from Steps 3 and 4. Write each file directly — no further confirmation needed.

Create directories if missing. Only generate files that were missing or requested for regeneration.

### Shopify — additional sections to include *(IS_SHOPIFY only)*

When **[IS_SHOPIFY]** = `true`, include the following sections in the relevant generated files:

**In `project-architecture`** — add after Tech Stack table:
```markdown
## Shopify App Structure
- Auth strategy: [from shopify.app.toml — e.g. merchant-installed]
- Scopes: [list from shopify.app.toml]
- Session storage: [from shopify.server.js]
- Extensions: [list from extensions/ with type + purpose, or "none"]

## Route Conventions
- Admin-authenticated routes: use `authenticate.admin(request)` from shopify.server.js
- Webhook handlers: [file pattern and registration approach]
- Public/unauthenticated routes: [if any]
```

**In `api-contracts`** — add after Base URL:
```markdown
## Shopify API Usage
- Admin GraphQL: [yes/no — note client setup from shopify.server.js]
- Storefront GraphQL: [yes/no]
- REST Admin API: [yes/no]

## GraphQL Patterns
[How Admin API queries/mutations are structured in this app — include 1 example from existing routes]

## Webhook Patterns
[How webhooks are registered and handled]
```

**In `coding-standards`** — add under Component / Module Patterns:
```markdown
## Shopify / Polaris Rules
- UI components: use Polaris — import from `@shopify/polaris`
- Page layout: use AppProvider > Page > Layout structure
- Loading states: use Polaris Skeleton components
- Never use raw HTML elements where a Polaris equivalent exists
- App bridge: use `useAppBridge()` hook for redirect/modal/toast
```

---

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

### Shopify — testing-standards additions *(IS_SHOPIFY only)*

When **[IS_SHOPIFY]** = `true`, add the following section inside the generated `testing-standards` file, after the Auth in Tests section:

```markdown
## Shopify-Specific Test Patterns

### Mocking the Shopify Session
Use the `shopify` test helper from `@shopify/shopify-app-remix/testing` to mock authenticated sessions:
```typescript
import { mockShopifySession } from "@shopify/shopify-app-remix/testing";
// Set up mock session before each test that hits an authenticated route
```

### Testing GraphQL Calls
- Mock `admin.graphql()` responses in unit tests — do not make real API calls in CI.
- Always include a `userErrors: []` field in mock responses to test the happy path.
- Add a separate test case with non-empty `userErrors` to verify error handling.

### Testing Webhooks
- Use the Shopify CLI (`shopify app webhook trigger`) to send test webhook payloads locally.
- In unit tests, construct a valid HMAC signature using the test secret from `.env`.
- Test both valid and invalid HMAC scenarios.

### Environment Variables Required for Tests
- `SHOPIFY_API_KEY` — test app API key
- `SHOPIFY_API_SECRET` — used for HMAC validation in webhook tests
- `SHOPIFY_APP_URL` — app host URL for session validation
```

---

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

## Step 6 — Report

```
Project setup complete.

Stack detected: [STACK]
Files written:
[List of files actually written]

Assumptions to verify:
[List any inferred values to double-check — colors, test user credentials, env var names]

You're ready to use /start-feature.
```
