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

### Workflow Skills

| Skill | When Claude uses it |
|-------|-------------------|
| **development-workflows** | Running worktree, peer review, QA gate, debugging, or autofix workflows |
| **gitnexus** | Semantic code analysis — blast-radius impact, call chain tracing, safe renames |
| **sparc-developer** | Multi-file backend features — Specification → Pseudocode → Architecture → Refinement → Completion |
| **reasoning-capture** | Every implementation session — writes `.g4a/.current_reasoning.json` artifact |

---

## Shopify Development

If you're working on a Shopify app, install the official Shopify AI Toolkit plugin alongside this one. It gives Claude access to Shopify's GraphQL API schemas, documentation, and code validation.

Run these two commands once per machine (after the mozilor-skills install above):

```bash
/plugin marketplace add Shopify/shopify-ai-toolkit
/plugin install shopify-plugin@shopify-plugin
```

Then run `/setup-project` as normal in your Shopify repo — it will detect `shopify.app.toml` and generate Shopify-aware skill files automatically.

> `/start-feature` requires the Shopify plugin to be installed before it will run in a Shopify project.

---

## Requirements

- Claude Code latest — run `claude --version` to check, update with `npm i -g @anthropic-ai/claude-code@latest`
- GitHub account with access to `mozilor-technologies/mozilor-skills`
