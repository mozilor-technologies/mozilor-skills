# SPARC Workflow for Store Robo Import/Export

## Overview

**SPARC** is a systematic methodology for complex feature development:
- **S**pecification — Clarify objectives, scope, constraints
- **P**seudocode — High-level logic with TDD anchors
- **A**rchitecture — System design, service boundaries, file routing
- **R**efinement — TDD, debugging, security, optimization
- **C**ompletion — Integration, docs, verification

## When to Use SPARC

Use SPARC for:
- ✅ New features with multiple components (endpoint + service + importer)
- ✅ Complex refactors affecting >3 files
- ✅ HIGH/MEDIUM risk changes (from `gitnexus_impact`)
- ✅ Features requiring architectural decisions

Skip SPARC for:
- ❌ Simple bug fixes
- ❌ Single-file changes
- ❌ LOW risk modifications

## SPARC Phases

### Phase S: Specification

**Goal:** Create clear, actionable requirements document.

**Steps:**
1. **Run GitNexus exploration** — Understand existing architecture
   ```
   gitnexus_query({query: "related feature concept"})
   gitnexus_context({name: "RelatedClass"})
   ```

2. **Document requirements**:
   ```markdown
   ## Overview
   Brief feature description

   ## Problem Statement
   What issue does this solve?

   ## Requirements
   ### Functional
   - Requirement 1
   - Requirement 2

   ### Non-Functional
   - Performance: < 30s for 1000 items
   - Memory: Stream processing

   ## API Design
   New endpoints with request/response schemas

   ## Architecture
   Component flow diagram

   ## Risks & Mitigations
   | Risk | Mitigation |
   ```

3. **Save spec** to `docs/specs/<feature-slug>.md`

**Deliverable:** Specification document

---

### Phase P: Pseudocode

**Goal:** Design high-level algorithm and data flow.

**Steps:**
1. **Design core algorithm** in pseudocode:
   ```python
   # Example
   def generate_import_summary(job_id):
       identifiers = load_all_identifiers()
       stats = empty_stats()

       for batch in batch_identifiers(identifiers, size=50):
           mapped_products = map_csv_to_shopify(batch)
           existing_products = fetch_from_shopify(batch)

           for mapped, existing in zip(mapped_products, existing_products):
               diff = calculate_diff(mapped, existing)
               accumulate_stats(stats, diff)

       return stats
   ```

2. **Identify test anchors**:
   - Unit test for `calculate_diff`
   - Integration test for `generate_import_summary`
   - Edge cases: empty batch, API failure

3. **Add to spec** under "## Pseudocode" section

**Deliverable:** Algorithm design with test anchors

---

### Phase A: Architecture

**Goal:** Define system boundaries, file structure, dependencies.

**Steps:**
1. **Component diagram**:
   ```
   ┌─────────────────┐
   │  API Endpoint   │
   │  /summary/{id}  │
   └────────┬────────┘
            │
            ▼
   ┌─────────────────┐
   │ ImportSummary   │
   │    Service      │
   └────────┬────────┘
            │
       ┌────┴────┐
       │         │
       ▼         ▼
   ┌───────┐  ┌────────────┐
   │ Diff  │  │  Shopify   │
   │Calc   │  │  Client    │
   └───────┘  └────────────┘
   ```

2. **File routing**:
   ```
   app/
     pipeline/importers/diff_calculator.py     # New: ProductDiffCalculator
     services/import_summary_service.py        # New: ImportSummaryService
     api/v1/endpoints/import_process.py        # Modified: add /summary endpoint
     schemas/api_responses.py                  # Modified: add response schemas
   tests/
     test_diff_calculator.py                   # New: unit tests
   docs/
     specs/import-preview-summary.md           # Spec document
   ```

3. **Run impact analysis** on files to modify:
   ```
   gitnexus_impact({target: "import_process", direction: "upstream"})
   ```

4. **Identify dependencies**:
   - Existing: `DataProvider`, `ProductImporter.prefetch_preview_context`
   - New: `ProductDiffCalculator`

**Deliverable:** Architecture doc with file plan and impact assessment

---

### Phase R: Refinement

**Goal:** Implement with TDD, optimize, secure.

**Sub-phases:**

#### R1: Specification → Code (TDD)
1. **Write tests first** (test_diff_calculator.py)
2. **Implement core logic** (diff_calculator.py)
3. **Run tests**: `uv run pytest tests/test_diff_calculator.py -v`
4. **Iterate** until tests pass

#### R2: Integration
1. **Build orchestration service** (import_summary_service.py)
2. **Add API endpoint** (import_process.py)
3. **Run integration tests**

#### R3: Security Review
1. **Input validation** — Job ownership, plan entitlements
2. **Rate limiting** — Batch Shopify lookups
3. **No secrets** — Check for hardcoded credentials

#### R4: Optimization
1. **Batching** — 50 products per Shopify query
2. **Caching** — Store results in `job.meta_data`
3. **Streaming** — Don't load all products into memory

**Deliverable:** Tested, optimized code

---

### Phase C: Completion

**Goal:** Integrate, document, verify.

**Steps:**
1. **Run full verification**:
   ```bash
   .claude/skills/sr-import-export-developer/scripts/verify.sh
   ```

2. **GitNexus scope check**:
   ```
   gitnexus_detect_changes({scope: "all"})
   ```

3. **Update reasoning** (`.g4a/.current_reasoning.json`):
   ```json
   {
     "outcome": {
       "status": "complete",
       "summary": "Added import preview summary feature...",
       "files_changed": [...],
       "tests_passed": true
     }
   }
   ```

4. **Create summary** for user:
   ```markdown
   ## Feature Complete: Import Preview Summary

   ✅ 3 files created
   ✅ 2 files modified
   ✅ 11 tests passing
   ✅ API endpoint functional
   ```

**Deliverable:** Production-ready feature with documentation

---

## SPARC Orchestration Pattern

Use Claude Code's Agent tool to spawn specialist agents for each phase:

```python
# Phase S: Specification
Agent(
    subagent_type="Explore",
    description="Explore import pipeline architecture",
    prompt="Analyze app/pipeline/importers/ and app/services/...",
)

# Phase A: Architecture + R: Refinement (parallel)
Agent(
    subagent_type="coder",
    description="Create diff calculator",
    run_in_background=True,
    prompt="Build ProductDiffCalculator in app/pipeline/importers/diff_calculator.py...",
)
Agent(
    subagent_type="coder",
    description="Create summary service",
    run_in_background=True,
    prompt="Build ImportSummaryService in app/services/import_summary_service.py...",
)
Agent(
    subagent_type="coder",
    description="Create response schemas",
    run_in_background=True,
    prompt="Add ImportSummaryData schemas to app/schemas/api_responses.py...",
)
```

---

## Template: SPARC Spec Document

```markdown
# [Feature Name] - SPARC Specification

## Overview
[Brief description]

## Problem Statement
[What issue does this solve?]

## Requirements

### Functional Requirements
1. [Requirement 1]
2. [Requirement 2]

### Non-Functional Requirements
- Performance: [metric]
- Memory: [constraint]
- Security: [requirements]

## API Design

### Endpoint: POST /api/v1/[path]

**Request:**
\`\`\`json
{
  "field": "value"
}
\`\`\`

**Response:**
\`\`\`json
{
  "success": true,
  "data": {}
}
\`\`\`

## Pseudocode

\`\`\`python
def main_function():
    step1()
    step2()
    return result
\`\`\`

### Test Anchors
- Unit: test_step1(), test_step2()
- Integration: test_main_function()
- Edge: test_empty_input(), test_api_failure()

## Architecture

### Component Flow
\`\`\`
[Diagram]
\`\`\`

### File Plan
| File | Action | Purpose |
|------|--------|---------|
| app/... | Create | ... |
| tests/... | Create | ... |

### Impact Analysis
- Modified symbols: [list]
- Risk level: LOW/MEDIUM/HIGH
- Affected processes: [from gitnexus]

## Technical Decisions

### Q: [Decision point]?
**A:** [Chosen approach with rationale]

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| ... | ... |

## Success Criteria
1. [Criterion 1]
2. [Criterion 2]
```

---

## Integration with Skill Workflows

SPARC integrates into the existing **Build Feature** workflow:

1. **Phase 1: Initial Response** — Ask worktree question
2. **Phase 2: Planning** — **SPARC S+A phases** (Spec + Architecture)
3. **Phase 2.5: Plan Refinement** — Peer review if HIGH/MEDIUM risk
4. **Phase 3: Implementation** — **SPARC R phase** (Refinement with TDD)
5. **Phase 4: Pre-Commit** — **SPARC C phase** (Completion verification)
6. **Phase 5: Post-Implementation** — Summary + PR

**Result:** SPARC provides the methodology, existing workflow provides the gates.
