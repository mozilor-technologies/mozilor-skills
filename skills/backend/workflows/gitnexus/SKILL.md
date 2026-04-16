---
name: gitnexus
description: Code intelligence via knowledge graph - impact analysis, bug tracing, architecture exploration, safe refactoring
triggers:
  - "blast radius"
  - "impact"
  - "what breaks"
  - "dependencies"
  - "execution flow"
  - "architecture"
  - "trace"
  - "refactor"
  - "rename"
---

# GitNexus

Code intelligence via precomputed knowledge graph. Understand dependencies, blast radius, execution flows, and refactor safely.

## Quick Start

**For any task involving code understanding, debugging, impact analysis, or refactoring:**

1. **Read `gitnexus://repo/{name}/context`** — codebase overview + check index freshness
2. **Match your task to a workflow below** and follow it
3. **Run the workflow's checklist**

> If step 1 warns the index is stale, run `npx gitnexus analyze` in the terminal first.

## Current Repository Stats

This repository has:
- 5,537 symbols | 14,590 relationships | 300 execution flows | 3,482 embeddings

Semantic search is enabled. Use `gitnexus_query` for pattern-based searches.

## Tools Quick Reference

| Tool             | What it gives you                                                        | When to use                          |
| ---------------- | ------------------------------------------------------------------------ | ------------------------------------ |
| `query`          | Process-grouped code intelligence — execution flows related to a concept | Understanding how code works         |
| `context`        | 360-degree symbol view — categorized refs, processes it participates in  | Deep dive on specific symbol         |
| `impact`         | Symbol blast radius — what breaks at depth 1/2/3 with confidence         | Before changing code                 |
| `detect_changes` | Git-diff impact — what do your current changes affect                    | Before committing                    |
| `rename`         | Multi-file coordinated rename with confidence-tagged edits               | Renaming symbols safely              |
| `cypher`         | Raw graph queries (read `gitnexus://repo/{name}/schema` first)           | Custom queries beyond provided tools |
| `list_repos`     | Discover indexed repos                                                   | Multi-repo navigation                |

## Workflows

Choose the workflow that matches your task:

### [Architecture Exploration](workflows/exploring.md)

**Use when:** "How does X work?", "What calls this?", "Show me the auth flow"

**Quick workflow:**
```
1. READ gitnexus://repo/{name}/context             → Overview
2. gitnexus_query({query: "concept"})               → Find execution flows
3. gitnexus_context({name: "symbol"})               → Deep dive
4. READ gitnexus://repo/{name}/process/{name}       → Full trace
```

### [Impact Analysis](workflows/impact-analysis.md)

**Use when:** "Is it safe to change X?", "What will break?", "Show me the blast radius"

**Quick workflow:**
```
1. gitnexus_impact({target: "X", direction: "upstream"})  → Dependents
2. READ gitnexus://repo/{name}/processes                   → Affected flows
3. gitnexus_detect_changes()                               → Pre-commit check
4. Assess risk and report
```

### [Debugging](workflows/debugging.md)

**Use when:** "Why is X failing?", "Trace this error", "Where does this bug come from?"

**Quick workflow:**
```
1. gitnexus_query({query: "error or symptom"})      → Related flows
2. gitnexus_context({name: "suspect"})              → Callers/callees
3. READ gitnexus://repo/{name}/process/{name}       → Trace execution
4. gitnexus_cypher for custom traces if needed
```

### [Refactoring](workflows/refactoring.md)

**Use when:** "Rename this", "Extract this module", "Split this service", "Refactor safely"

**Quick workflow:**
```
1. gitnexus_impact({target: "X", direction: "upstream"})  → Map dependents
2. gitnexus_rename for symbol renames (preview first)
3. gitnexus_detect_changes after edits                     → Verify scope
4. Run tests for affected processes
```

### [CLI Reference](workflows/cli-reference.md)

**Use when:** Need to index/reindex, check status, clean index, generate wiki

**Common commands:**
```bash
npx gitnexus analyze        # Build/refresh index
npx gitnexus status         # Check freshness
npx gitnexus clean          # Delete index
npx gitnexus wiki           # Generate docs
npx gitnexus list           # Show indexed repos
```

## MCP Resources Reference

Lightweight reads (~100-500 tokens) for navigation:

| Resource                                       | Content                                   |
| ---------------------------------------------- | ----------------------------------------- |
| `gitnexus://repo/{name}/context`               | Stats, staleness check                    |
| `gitnexus://repo/{name}/clusters`              | All functional areas with cohesion scores |
| `gitnexus://repo/{name}/cluster/{clusterName}` | Area members                              |
| `gitnexus://repo/{name}/processes`             | All execution flows                       |
| `gitnexus://repo/{name}/process/{processName}` | Step-by-step trace                        |
| `gitnexus://repo/{name}/schema`                | Graph schema for Cypher                   |

## Graph Schema

**Nodes:** File, Function, Class, Interface, Method, Community, Process

**Edges (via CodeRelation.type):** CALLS, IMPORTS, EXTENDS, IMPLEMENTS, DEFINES, MEMBER_OF, STEP_IN_PROCESS

**Example Cypher query:**
```cypher
MATCH (caller)-[:CodeRelation {type: 'CALLS'}]->(f:Function {name: "myFunc"})
RETURN caller.name, caller.filePath
```

## Understanding Impact Results

| Depth | Risk Level       | Meaning                  |
| ----- | ---------------- | ------------------------ |
| d=1   | **WILL BREAK**   | Direct callers/importers |
| d=2   | LIKELY AFFECTED  | Indirect dependencies    |
| d=3   | MAY NEED TESTING | Transitive effects       |

### Risk Assessment Guidelines

| Impact Result       | Risk Level | Required Action                  |
| ------------------- | ---------- | -------------------------------- |
| 0-2 d=1 callers     | LOW        | Proceed, update callers          |
| 3-5 d=1 callers     | MEDIUM     | Proceed carefully, test          |
| 6-10 d=1 callers    | HIGH       | Consider preserving interface    |
| >10 d=1 callers     | CRITICAL   | **STOP** — ask user first        |
| Touches auth/billing| CRITICAL   | **STOP** — extra review required |

## Best Practices

### Before Editing Code
1. Always run `gitnexus_impact` on existing symbols to understand blast radius
2. Check d=1 callers — these WILL BREAK if interface changes
3. Use `gitnexus_context` to see what the symbol depends on

### During Refactoring
1. Use `gitnexus_rename` for symbol renames (safer than find-and-replace)
2. Preview first with `dry_run: true`
3. Review graph edits (high confidence) vs text_search edits (review carefully)

### After Editing
1. Run `gitnexus_detect_changes({scope: "all"})` to verify affected scope
2. Check that affected processes match expectations
3. Run tests for all affected execution flows

### For Semantic Search
Use `gitnexus_query` to find patterns:
- `"similar service classes"` — find patterns to follow
- `"how are API endpoints structured"` — understand conventions
- `"error handling patterns"` — find consistent approaches

## Common Patterns

### Exploring Unfamiliar Code
```
READ gitnexus://repo/{name}/context → gitnexus_query → gitnexus_context → READ process
```

### Safety Check Before Changing Code
```
gitnexus_impact({target, direction: "upstream"}) → assess d=1 callers → decide approach
```

### Debugging an Error
```
gitnexus_query({query: "error text"}) → gitnexus_context({name: "suspect"}) → READ process
```

### Safe Rename
```
gitnexus_rename({symbol_name, new_name, dry_run: true}) → review → apply
```

### Pre-Commit Verification
```
gitnexus_detect_changes({scope: "all"}) → confirm affected scope → commit
```

## Troubleshooting

**Index is stale:**
```bash
npx gitnexus analyze
```

**Need semantic search:**
```bash
npx gitnexus analyze --embeddings
```

**Index is corrupt:**
```bash
npx gitnexus clean && npx gitnexus analyze
```

**After git commit/merge:**
The PostToolUse hook auto-runs `analyze` to keep the index fresh.

## Next Steps

1. Read `gitnexus://repo/{name}/context` to verify the index is loaded
2. Choose a workflow above that matches your task
3. Follow the workflow's detailed steps and checklist
4. Use the tools reference above for detailed parameter options
