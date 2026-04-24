# Shopify Integration Flow

How the mozilor-skills plugin integrates with the Shopify AI Toolkit to support Shopify app development.

---

## 1. Developer Machine Setup (one time)

```mermaid
flowchart TD
    A([Developer Machine]) --> B[Install mozilor-skills plugin]
    B --> C{Working on a Shopify project?}
    C -->|No| D([✅ Ready for all other stacks])
    C -->|Yes| E[Install Shopify AI Toolkit plugin]
    E --> F([✅ Ready for Shopify projects])

    style D fill:#d4edda,stroke:#28a745
    style F fill:#d4edda,stroke:#28a745
```

> The Shopify AI Toolkit plugin is only needed once per machine, not per project.

---

## 2. Per-Project Setup — `/setup-project`

```mermaid
flowchart TD
    A([/setup-project]) --> B{shopify.app.toml\nexists at root?}

    B -->|No| C[Standard stack detection\nIS_SHOPIFY = false]
    B -->|Yes| D[IS_SHOPIFY = true\nSTACK = fullstack\nBACKEND_LANG = nodejs]

    D --> E{Shopify plugin\ninstalled?}
    E -->|No| F[⚠️ Warn user\nContinue anyway]
    E -->|Yes| G[Shopify-specific\ncodebase exploration]
    F --> G

    G --> H["Scan: shopify.app.toml\napp/shopify.server.js\napp/routes/\nextensions/\napp/db.server.*"]

    C --> I[Standard codebase\nexploration]
    I --> J[Generate standard\nskill files]
    H --> K[Generate Shopify-aware\nskill files]

    J --> L[(".claude/skills/\nproject-architecture\napi-contracts\ncoding-standards\ntesting-standards")]
    K --> M[(".claude/skills/\nproject-architecture ✦ Shopify app structure, auth, scopes, extensions\napi-contracts ✦ GraphQL Admin/Storefront patterns\ncoding-standards ✦ Polaris rules, Remix conventions\ntesting-standards")]

    style F fill:#fff3cd,stroke:#ffc107
    style M fill:#cce5ff,stroke:#004085
```

> `testing-standards` includes Shopify-specific content: session mocking, GraphQL mocking, webhook testing, and required env vars.

---

## 3. Per-Feature Development — `/start-feature`

```mermaid
flowchart TD
    A(["/start-feature 'add product metafield editor'"]) --> B{shopify.app.toml\nexists at root?}

    B -->|No| C[Standard stack detection\nIS_SHOPIFY = false]
    B -->|Yes| D{Shopify plugin\ninstalled?}

    D -->|No| E(["🛑 Hard Stop\nInstall Shopify plugin first"])
    D -->|Yes| F["IS_SHOPIFY = true\nSTACK = fullstack\nBACKEND_LANG = nodejs"]

    C --> G[Confirm stack with user]
    F --> G

    G --> H["Pre-load:\nCODING_RULES_DIGEST\nIS_SHOPIFY flag\nFIGMA_AVAILABLE flag"]

    H --> I["Phase 1 — Research\nbackend/researcher 🔌 active plugin use\nfrontend/research-agent ← IS_SHOPIFY"]

    I --> J["Phase 2 — Spec / Design\nbackend/spec-agent 🔌 active plugin use\nfrontend/design-agent ← IS_SHOPIFY"]

    J --> K["Phase 3 — Architecture Review\n(human checkpoint)"]

    K --> L["Phase 4 — Implementation\nbackend/implementer 🔌 active plugin use\nfrontend/codegen-agent ← IS_SHOPIFY"]

    L --> M["Phase 5 — Validation\nbackend/qa 🔌 active plugin use\nfrontend/code-review, test, security"]

    M --> N{Blocking\nissues?}
    N -->|No| O([✅ Feature complete])
    N -->|Yes| P["Fix Loop\nfrontend/fix-agent ← IS_SHOPIFY\nbackend/debugger ← IS_SHOPIFY"]
    P --> M

    style E fill:#f8d7da,stroke:#dc3545
    style O fill:#d4edda,stroke:#28a745
```

---


## Active vs Passive Plugin Usage

🔌 = agent actively calls `search_docs_chunks` / `fetch_full_docs` at runtime
← = agent receives IS_SHOPIFY flag and applies hardcoded Shopify rules (passive)

| Agent | Plugin usage | What it does with the plugin |
|---|---|---|
| `backend/researcher` | 🔌 Active | Looks up API schemas and docs before codebase research |
| `backend/spec-agent` | 🔌 Active | Looks up exact field names/types before writing GraphQL contracts |
| `backend/implementer` | 🔌 Active | Validates schema before writing every GraphQL query or mutation |
| `backend/qa` | 🔌 Active | Verifies all GraphQL operations in changed files against current schema |
| `frontend/research-agent` | ← Passive | Applies Shopify-aware research patterns (Polaris lookup, constraint flagging) |
| `frontend/design-agent` | ← Passive | Specs Polaris components, GraphQL contracts, Remix loader/action |
| `frontend/codegen-agent` | ← Passive | Enforces Polaris, app bridge, loader/action during code generation |
| `frontend/fix-agent` | ← Passive | Applies Shopify fix constraints (Polaris, auth, userErrors) |
| `backend/debugger` | ← Passive | Applies Shopify-specific root cause patterns |

---

## Plugin Check Gates

| Command | Plugin missing | Reason |
|---|---|---|
| `/setup-project` | ⚠️ Warn and continue | Only reads the codebase — plugin not needed to generate skill files |
| `/start-feature` | 🛑 Hard stop | 4 agents actively call plugin tools — pipeline breaks without it |

Plugin check command (confirmed working):
```bash
claude plugin list | grep -A3 "shopify-plugin" | grep "✔ enabled"
```
