# Mozilor Skills — Claude Code Plugin

Mozilor Technologies' shared skill library and agentic workflow commands for Claude Code.

---

## Installation

Run these two commands once per machine:

```bash
/plugin marketplace add mozilor-technologies/mozilor-skills
/plugin install workflow@mozilor-skills
```

To keep it up to date:

```bash
/plugin marketplace update mozilor-skills
```

---

## Commands

After installing, two slash commands are available in every project:

### `/setup-project`

Run once per project before starting any feature work. Explores your codebase and writes project-specific skill files that all agents use.

```bash
/setup-project
```

Re-run options:
```bash
/setup-project regenerate          # overwrite all skill files
/setup-project coding-standards    # regenerate a specific file
```

### `/start-feature`

Run for every new feature. Auto-detects your stack and orchestrates the full pipeline.

```bash
/start-feature <description>
/start-feature <description> <confluence-url>
/start-feature <description> <figma-url>
/start-feature <description> <confluence-url> <figma-url>
```

---

## Bundled Skills

Skills are loaded automatically by Claude Code when relevant — no manual invocation needed.

### Frontend Skills

| Skill | When Claude uses it |
|-------|-------------------|
| **react-best-practices** | Writing or reviewing React / Next.js code — 68 performance rules covering waterfalls, bundle size, re-renders, SSR |
| **composition-patterns** | Designing or refactoring React components — compound components, variants, context patterns |
| **accessibility-compliance** | Any UI work — WCAG 2.2 AA/AAA, ARIA, keyboard navigation, screen reader support |
| **react-view-transitions** | Adding animations or page transitions — native View Transition API patterns |
| **web-design-guidelines** | Reviewing UI code — Vercel Web Interface Guidelines compliance |

### Backend Skills

| Skill | When Claude uses it |
|-------|-------------------|
| **api-design** | Designing REST or GraphQL APIs — resource naming, status codes, versioning, pagination |
| **nodejs-backend** | Writing Node.js server code — layered architecture, error handling, queues, caching |
| **fastapi-python** | Writing Python backend code — FastAPI patterns and standards |
| **typescript** | TypeScript-heavy codebases — type safety patterns |
| **postgres** | Database work — schema design, indexing strategy, query optimisation |
| **go** | Go backend code — language standards and patterns |
| **laravel-woo** | PHP projects — Laravel and WooCommerce patterns |
| **security** | Any implementation — OWASP Top 10, auth, input validation, secrets |

### Shopify Skills

`mozilor-skills` bundles a curated set of Shopify skills sourced from [Shopify's official AI Toolkit](https://github.com/Shopify/Shopify-AI-Toolkit) (MIT-licensed), plus a Mozilor-authored `shopify-polaris-react` skill specifically for Remix-template apps using `@shopify/shopify-app-remix`.

| Skill | When Claude uses it |
|-------|-------------------|
| **shopify-polaris-react** *(Mozilor)* | Polaris React UI in Remix admin apps — `AppProvider`, `NavMenu`, `useAppBridge`, save bar/toast/modal, GraphQL via `admin.graphql()`, webhook handlers |
| **shopify-polaris-admin-extensions** | UI extensions under `extensions/` — admin blocks and action menu items (these legitimately use `s-*` web components) |
| **shopify-admin** | Authoring Admin GraphQL queries and mutations |
| **shopify-custom-data** | Metafields and metaobjects |
| **shopify-liquid** | Liquid theme code — sections, blocks, snippets |
| **shopify-use-shopify-cli** | Shopify CLI tasks — config validation, store auth, executing queries |
| **shopify-dev** | Catch-all docs search across `shopify.dev` |
| **shopify-app-store-review** | Pre-submission App Store compliance check |

On a Shopify Remix repo (detected via `@shopify/shopify-app-remix` in `package.json`), `/setup-project` writes a managed section to `CLAUDE.md` pointing agents at the `shopify-polaris-react` skill and pinning auth, API, and webhook conventions for the framework.

### Workflow Skills

| Skill | When Claude uses it |
|-------|-------------------|
| **development-workflows** | Running worktree, peer review, QA gate, debugging, or autofix workflows |
| **gitnexus** | Semantic code analysis — blast-radius impact, call chain tracing, safe renames |
| **sparc-developer** | Multi-file backend features — Specification → Pseudocode → Architecture → Refinement → Completion |
| **reasoning-capture** | Every implementation session — writes `.g4a/.current_reasoning.json` artifact |

---

## Requirements

- Claude Code latest — run `claude --version` to check, update with `npm i -g @anthropic-ai/claude-code@latest`
- GitHub account with access to `mozilor-technologies/mozilor-skills`
