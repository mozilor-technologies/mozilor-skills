# SPARC Agents for Store Robo Import/Export

## Overview

Specialized agents for each SPARC phase, tailored to the Store Robo import/export backend.

## Agent Definitions

### SR-Spec (Specification Agent)

**Phase:** SPARC S — Specification

**Purpose:** Research existing architecture and create comprehensive specification document.

**Tools:** Read, Grep, Glob, GitNexus (query, context, impact)

**Prompt Template:**
```
Research the existing architecture for [feature area] and create a specification for [feature name].

Steps:
1. Use gitnexus_query to find related code
2. Read key files in app/pipeline/[analyzers|importers|exporters]
3. Understand existing patterns and constraints
4. Document:
   - Problem statement
   - Functional requirements
   - Non-functional requirements (performance, memory, security)
   - API design (endpoints, schemas)
   - Architecture (component flow)
   - Impact analysis results
   - Risks and mitigations

Output: Markdown specification document for docs/specs/<feature-slug>.md
```

**Example Usage:**
```python
Agent(
    subagent_type="Explore",  # Use built-in Explore agent
    description="Research import pipeline for preview feature",
    prompt="""Research the import pipeline to understand how to add a preview summary feature.

Investigate:
1. How CSV files are analyzed (app/pipeline/analyzers/)
2. How imports are executed (app/pipeline/importers/)
3. Existing preview/dry-run logic
4. How to compare incoming CSV vs existing Shopify data

Create specification document with:
- Problem statement
- Requirements
- API design
- Architecture
- Impact analysis
""",
)
```

---

### SR-Architect (Architecture Agent)

**Phase:** SPARC A — Architecture

**Purpose:** Design system boundaries, file routing, and integration points.

**Tools:** Read, Grep, Glob, GitNexus (context, impact), Write

**Prompt Template:**
```
Design the architecture for [feature name] following Store Robo patterns.

Requirements from spec:
[paste requirements]

Tasks:
1. Create component diagram showing data flow
2. Define file routing:
   - New files to create
   - Existing files to modify
   - Test files needed
3. Run gitnexus_impact on all symbols to be modified
4. Identify dependencies:
   - Existing services to reuse
   - New components needed
5. Design integration points
6. Add architecture section to spec document

Follow patterns:
- Registry pattern for importers/exporters
- Service layer for orchestration
- Celery for background tasks
- FastAPI for endpoints
```

**Example Usage:**
```python
Agent(
    subagent_type="system-architect",  # Built-in architect
    description="Design import summary architecture",
    prompt="""Design architecture for import preview summary feature.

Based on spec at docs/specs/import-preview-summary.md:
1. Component flow diagram
2. File routing (what to create/modify)
3. Run gitnexus_impact on ProductImporter, import_process.py
4. Dependencies needed
5. Integration with DataProvider, ShopifyClient

Add architecture to spec document.
""",
)
```

---

### SR-Coder-TDD (TDD Implementation Agent)

**Phase:** SPARC R1 — Refinement (TDD)

**Purpose:** Implement code using test-driven development.

**Tools:** Read, Write, Edit, Bash (pytest, ruff)

**Prompt Template:**
```
Implement [component name] using TDD for Store Robo import/export backend.

Spec: [link to spec]
Architecture: [component details]

TDD Steps:
1. Write tests first in tests/test_[name].py
2. Run tests (should fail): uv run pytest tests/test_[name].py -v
3. Implement in app/[path]/[name].py
4. Run tests (should pass)
5. Refactor if needed
6. Lint: uv run ruff check --fix && ruff format

Requirements:
- Use uv (never pip)
- Follow existing patterns in codebase
- Keep files under 500 lines
- No hardcoded secrets
- Input validation at boundaries
- Type hints for all functions

Do NOT create files unless explicitly needed.
```

**Example Usage:**
```python
Agent(
    subagent_type="coder",
    description="Implement diff calculator with TDD",
    run_in_background=True,
    prompt="""Create ProductDiffCalculator using TDD.

File: app/pipeline/importers/diff_calculator.py
Tests: tests/test_diff_calculator.py

Requirements:
1. Compare incoming CSV product vs existing Shopify product
2. Detect: price, inventory, title, description, tag, variant, image changes
3. Return ProductDiff with action (create/update/skip/fail) and change flags

TDD workflow:
1. Write tests first (11 test cases from spec)
2. Implement ProductDiffCalculator
3. Run: uv run pytest tests/test_diff_calculator.py -v
4. Lint: uv run ruff check --fix && ruff format

Keep under 220 lines.
""",
)
```

---

### SR-Security (Security Review Agent)

**Phase:** SPARC R3 — Refinement (Security)

**Purpose:** Security audit and hardening.

**Tools:** Read, Grep, Glob, Edit

**Prompt Template:**
```
Perform security review for [feature name] in Store Robo import/export backend.

Check:
1. Input validation
   - User input sanitized
   - File paths validated (no directory traversal)
   - GraphQL injection prevented
2. Authentication/Authorization
   - Shopify JWT verified
   - Job ownership enforced
   - Plan entitlements checked
3. Secrets management
   - No hardcoded credentials
   - No .env committed
   - Shopify tokens from credentials service
4. Rate limiting
   - Shopify API rate limits respected
   - Adaptive throttling enabled
5. SQL injection
   - SQLModel prevents raw SQL
   - No string concatenation in queries
6. XSS/CSRF
   - FastAPI CORS properly configured
   - No user content rendered as HTML

Report findings with severity (CRITICAL/HIGH/MEDIUM/LOW).
Provide fixes for HIGH/CRITICAL issues.
```

**Example Usage:**
```python
Agent(
    subagent_type="security-architect",  # Built-in security agent
    description="Security review of import summary",
    prompt="""Security audit for import summary endpoint.

Files:
- app/api/v1/endpoints/import_process.py (new /summary endpoint)
- app/services/import_summary_service.py

Check:
1. Job ownership validation
2. Plan entitlement checks
3. No hardcoded secrets
4. Shopify rate limiting
5. Input sanitization (job_id, include_details, force_refresh)

Report issues with severity and fixes.
""",
)
```

---

### SR-Optimizer (Performance Optimization Agent)

**Phase:** SPARC R4 — Refinement (Optimization)

**Purpose:** Optimize performance, memory, and API usage.

**Tools:** Read, Edit, Bash (profiling)

**Prompt Template:**
```
Optimize [component name] for Store Robo import/export backend.

Current implementation: [file path]

Optimization targets:
1. Performance
   - Response time: [target]
   - Batch operations
   - Async/await properly used
2. Memory
   - Streaming (no full load)
   - Generator patterns
   - Cleanup after use
3. Shopify API
   - Batch queries (50 items)
   - Adaptive rate limiting
   - Cache results
   - Prefetch patterns

Measure:
- Before: [baseline metrics]
- After: [optimized metrics]

Apply optimizations and verify no regressions.
```

**Example Usage:**
```python
Agent(
    subagent_type="performance-engineer",  # Built-in perf agent
    description="Optimize import summary performance",
    prompt="""Optimize ImportSummaryService for large imports.

File: app/services/import_summary_service.py

Targets:
- < 30s for 1000 products
- Stream processing (don't load all into memory)
- Batch Shopify lookups (50 per query)
- Cache results in job.meta_data

Apply:
1. Batch processing pattern
2. Shopify API batching via prefetch_preview_context
3. Result caching
4. Memory-efficient streaming

Verify no regressions in tests.
""",
)
```

---

### SR-Integrator (Integration Agent)

**Phase:** SPARC C — Completion

**Purpose:** Integrate components and verify end-to-end flow.

**Tools:** Read, Edit, Bash (pytest, verify.sh)

**Prompt Template:**
```
Integrate and verify [feature name] for Store Robo import/export backend.

Components:
[List components]

Tasks:
1. Wire components together
2. Add API endpoint
3. Run integration tests
4. Run full test suite: uv run pytest tests/
5. Lint: uv run ruff check . && ruff format .
6. Run gitnexus_detect_changes to verify scope
7. Update G4A reasoning with outcome

Verification:
- All tests passing
- No lint errors
- GitNexus scope matches expected files
- G4A reasoning updated

Create summary report for user.
```

**Example Usage:**
```python
Agent(
    subagent_type="implementer",  # Built-in integrator
    description="Integrate import summary feature",
    prompt="""Integrate import summary components and verify.

Components:
- ProductDiffCalculator (created)
- ImportSummaryService (created)
- Response schemas (created)

Tasks:
1. Add /summary/{job_id} endpoint to import_process.py
2. Wire service to endpoint
3. Run tests: uv run pytest tests/
4. Lint: uv run ruff check . && ruff format .
5. Verify: gitnexus_detect_changes({scope: "all"})
6. Update .g4a/.current_reasoning.json

Create completion summary.
""",
)
```

---

## Orchestration Pattern

Use these agents in parallel where possible:

```python
# Phase S (sequential - exploration first)
Agent(subagent_type="Explore", description="Research architecture", ...)

# Phase A (after spec complete)
Agent(subagent_type="system-architect", description="Design architecture", ...)

# Phase R (parallel implementation)
Agent(subagent_type="coder", description="Diff calculator", run_in_background=True, ...)
Agent(subagent_type="coder", description="Summary service", run_in_background=True, ...)
Agent(subagent_type="coder", description="Response schemas", run_in_background=True, ...)

# Phase R (sequential - after implementation)
Agent(subagent_type="security-architect", description="Security review", ...)
Agent(subagent_type="performance-engineer", description="Optimize", ...)

# Phase C (sequential - final integration)
Agent(subagent_type="implementer", description="Integration", ...)
```

---

## Built-in Agents to Use

These Claude Code built-in agents work well for SPARC phases:

| SPARC Phase | Built-in Agent | Purpose |
|-------------|----------------|---------|
| S | `Explore` | Research architecture |
| P | Manual | Write pseudocode in spec |
| A | `system-architect` | Design architecture |
| R1 (TDD) | `coder` | Implement with TDD |
| R2 (Integration) | `implementer` | Wire components |
| R3 (Security) | `security-architect` | Security audit |
| R4 (Optimization) | `performance-engineer` | Performance tuning |
| C | `implementer` + `qa` | Integration + verification |

---

## Agent Communication

When spawning multiple agents:
1. **Pass context** — Spec doc location, architecture decisions
2. **Set dependencies** — Block agents on prior completion
3. **Run in background** — Parallel where possible
4. **Wait for results** — Don't poll, trust agent notifications
5. **Review outputs** — Check all agent results before proceeding

Example:
```python
# Specification phase (sequential)
spec_agent = Agent(subagent_type="Explore", ...)
# Wait for spec to complete

# Implementation phase (parallel)
Agent(subagent_type="coder", description="Component A", run_in_background=True, ...)
Agent(subagent_type="coder", description="Component B", run_in_background=True, ...)
Agent(subagent_type="coder", description="Component C", run_in_background=True, ...)
# All run in parallel, will be notified when done

# Integration phase (sequential, after all components complete)
Agent(subagent_type="implementer", description="Integrate A+B+C", ...)
```

---

## Best Practices

1. **One agent per component** — Don't make agents do too much
2. **Clear prompts** — Specify exact files, requirements, success criteria
3. **Trust agent outputs** — Generally reliable, review for correctness
4. **Parallel where possible** — Speed up development
5. **Sequential where needed** — Dependencies, integration
6. **Use built-in agents** — Leverage existing specializations
7. **Pass spec location** — Agents need context
8. **Verify after completion** — Run tests, lint, scope check
