# SPARC Quick Reference

## One-Page Cheat Sheet

### When to Use SPARC
✅ New features (multi-component)
✅ Complex refactors (>3 files)
✅ HIGH/MEDIUM risk changes
❌ Simple bug fixes
❌ Single-file changes

---

## The 5 Phases

### S — Specification
**Goal:** Document requirements
**Deliverable:** `docs/specs/<feature-slug>.md`
**Time:** 15-30 min

```markdown
## Problem Statement
## Requirements (Functional + Non-Functional)
## API Design
## Risks & Mitigations
```

**GitNexus:**
```
gitnexus_query({query: "related concept"})
gitnexus_context({name: "RelatedClass"})
```

---

### P — Pseudocode
**Goal:** Algorithm design
**Deliverable:** Pseudocode in spec
**Time:** 10-20 min

```python
def main_function():
    data = load_data()
    results = process(data)
    return results
```

**Test anchors:**
- Unit: `test_process()`
- Integration: `test_main_function()`
- Edge: `test_empty_data()`

---

### A — Architecture
**Goal:** System design
**Deliverable:** Component diagram + file routing
**Time:** 20-40 min

```
Component Flow:
API → Service → Calculator + Client → Response

File Routing:
✚ app/services/my_service.py
✚ app/pipeline/my_calculator.py
✎ app/api/endpoints/my_endpoint.py
✚ tests/test_my_calculator.py
```

**GitNexus:**
```
gitnexus_impact({target: "symbolName", direction: "upstream"})
```

---

### R — Refinement
**Goal:** Implement with quality
**Deliverable:** Production code
**Time:** 1-4 hours

**R1: TDD**
1. Write tests first
2. Implement
3. Tests pass
4. Refactor

**R2: Integration**
- Wire components
- Add API endpoint
- Integration tests

**R3: Security**
- Input validation
- Auth checks
- No secrets

**R4: Optimization**
- Batching
- Caching
- Streaming

---

### C — Completion
**Goal:** Verify readiness
**Deliverable:** Verified feature
**Time:** 10-20 min

```bash
uv run pytest tests/              # All tests
uv run ruff check . && ruff format .  # Lint
gitnexus_detect_changes({scope: "all"})  # Scope
```

**Update G4A:**
```json
{
  "outcome": {
    "status": "complete",
    "summary": "...",
    "files_changed": [...],
    "tests_passed": true
  }
}
```

---

## Agent Pattern

```python
# S: Research
Agent(subagent_type="Explore", description="Research architecture", ...)

# A: Design
Agent(subagent_type="system-architect", description="Design components", ...)

# R: Implement (parallel)
Agent(subagent_type="coder", description="Component A", run_in_background=True, ...)
Agent(subagent_type="coder", description="Component B", run_in_background=True, ...)
Agent(subagent_type="coder", description="Component C", run_in_background=True, ...)

# C: Integrate
Agent(subagent_type="implementer", description="Integration", ...)
```

---

## Checkpoints

| Phase | Checkpoint Question |
|-------|---------------------|
| S | Do I understand the problem and requirements? |
| P | Can I explain the algorithm to someone? |
| A | Do I know which files to create/modify? |
| R1 | Are tests written before code? |
| R2 | Are components integrated? |
| R3 | Is security validated? |
| R4 | Is performance acceptable? |
| C | Does everything pass verification? |

---

## Common Mistakes

❌ **Skip S** — Jump straight to code
✅ **Do S** — Understand before building

❌ **No pseudocode** — Dive into syntax
✅ **Write P** — Think algorithm first

❌ **Code then test** — Tests as afterthought
✅ **TDD** — Tests drive design

❌ **Big bang integration** — All at once
✅ **Incremental** — Component by component

❌ **Forget security** — Add later
✅ **R3** — Security in refinement

❌ **No verification** — Assume it works
✅ **C** — Full verification suite

---

## Time Budget (Typical Feature)

| Phase | Time | % |
|-------|------|---|
| S | 30 min | 10% |
| P | 20 min | 7% |
| A | 40 min | 13% |
| R | 3 hours | 60% |
| C | 30 min | 10% |
| **Total** | **5 hours** | **100%** |

---

## Success Criteria

✅ Spec document exists and is clear
✅ Pseudocode shows algorithm flow
✅ Architecture diagram shows components
✅ Impact analysis run on all changes
✅ Tests written before implementation
✅ Security review completed
✅ Performance meets targets
✅ All tests passing
✅ Lint clean
✅ GitNexus scope verified
✅ G4A reasoning updated

---

## Resources

- [sparc-workflow.md](sparc-workflow.md) — Full methodology
- [sparc-agents.md](sparc-agents.md) — Agent definitions
- [workflow-feature.md](workflow-feature.md) — Feature workflow integration
