---
name: reasoning-capture
description: Capture and query G4A reasoning artifacts - session reasoning, decisions, learnings. Use throughout development for audit trail.
triggers:
  - "capture reasoning"
  - "record decision"
  - "g4a"
auto_workflow: none
requires_repo_config: false
---

# Reasoning Capture (G4A)

Capture and query reasoning artifacts for audit trail and learning.

**Primary Use**: Invoked throughout all development phases
**Standalone Use**: Manual reasoning capture for decisions

## What is G4A?

**G4A (Git for AI)** is a reasoning layer that captures **why** AI made decisions:

```
git log       → what changed (code diffs)
GitNexus      → what depends on what (symbol graph)
G4A           → why the AI chose this (reasoning artifacts)
```

## When to Capture Reasoning

**Always capture for:**
- ✅ Feature implementation
- ✅ Debugging sessions
- ✅ Refactoring
- ✅ Architectural decisions
- ✅ QA reviews

**Skip for:**
- ❌ Trivial changes (typo fixes)
- ❌ Very small edits

## Reasoning Artifact Structure

### At Session Start

Create `.g4a/.current_reasoning.json`:

```json
{
  "version": "1.0",
  "agent": "implementer|debugger|qa|researcher|main",
  "task": "<one-line: what you are doing>",
  "context": {
    "trigger": "<what prompted this>",
    "files_read": [],
    "symbols_queried": [],
    "gitnexus_processes": []
  },
  "reasoning": {
    "hypothesis": "<initial theory or plan>",
    "approach": "<approach chosen>",
    "alternatives_considered": [],
    "confidence": 0.7
  },
  "impact": {
    "risk_level": "UNKNOWN",
    "gitnexus_impact_run": false
  },
  "outcome": {
    "status": "in_progress"
  }
}
```

### At Session End

Update `.g4a/.current_reasoning.json`:

```json
{
  ...
  "reasoning": {
    "hypothesis": "<confirmed or updated>",
    "approach": "<what you actually did>",
    "alternatives_considered": ["alt1", "alt2"],
    "why_this_approach": "<key insight>",
    "confidence": 0.9
  },
  "impact": {
    "risk_level": "HIGH|MEDIUM|LOW",
    "gitnexus_impact_run": true,
    "affected_symbols": [],
    "affected_processes": []
  },
  "outcome": {
    "status": "complete|partial|blocked",
    "summary": "<paragraph of what was accomplished>",
    "files_changed": ["file1.py", "file2.py"],
    "tests_passed": true,
    "follow_up": "<remaining work or open questions>"
  }
}
```

### The Stop Hook Archives It

When session ends, the Stop hook runs `scripts/g4a_capture.py`:
1. Reads `.g4a/.current_reasoning.json`
2. Moves to `.g4a/logs/YYYY-MM/YYYYMMDD-HHMMSS-slug.json`
3. Updates `.g4a/index.json`
4. Deletes temp file

**You do NOT manually run capture script** — the hook does it automatically.

## Reasoning by Task Type

### Implementation Tasks

```json
{
  "agent": "implementer",
  "task": "Add gift card import support",
  "reasoning": {
    "hypothesis": "Can reuse product importer pattern",
    "approach": "Registry pattern with GiftCardImporter class",
    "alternatives_considered": [
      "Extend ProductImporter (rejected: gift cards are distinct entities)",
      "Generic importer (rejected: entity-specific logic needed)"
    ],
    "why_this_approach": "Registry pattern proven, follows codebase conventions",
    "confidence": 0.85
  },
  "impact": {
    "risk_level": "MEDIUM",
    "gitnexus_impact_run": true,
    "affected_symbols": ["build_registry", "ImporterRegistry"],
    "affected_processes": ["import_job_execution"]
  },
  "outcome": {
    "status": "complete",
    "summary": "Added gift card import with registry integration. Created GiftCardImporter, GiftCardAnalyzer, registered in build_registry. All tests pass.",
    "files_changed": [
      "app/pipeline/importers/gift_card_importer.py",
      "app/pipeline/analyzers/gift_card.py",
      "app/pipeline/importers/registry.py",
      "tests/test_gift_card_importer.py"
    ],
    "tests_passed": true,
    "follow_up": "None"
  }
}
```

### Debugging Tasks

```json
{
  "agent": "debugger",
  "task": "Fix rate limiter memory leak",
  "reasoning": {
    "hypothesis": "Rate limiter not cleaning up old entries",
    "approach": "Add TTL cleanup to rate limiter",
    "alternatives_considered": [
      "Ignore (rejected: leak grows unbounded)",
      "Replace with Redis (rejected: overkill for simple leak)",
      "LRU eviction (rejected: time-based TTL more appropriate)"
    ],
    "why_this_approach": "TTL cleanup simple, solves root cause",
    "confidence": 0.9
  },
  "outcome": {
    "status": "complete",
    "summary": "Confirmed hypothesis: no cleanup of expired entries. Added background cleanup task running every 5 minutes. Memory usage stable after 24h test.",
    "files_changed": ["app/rate_limiter.py", "tests/test_rate_limiter.py"],
    "tests_passed": true,
    "follow_up": "Monitor production metrics for 1 week"
  }
}
```

## Reading Past Reasoning

### View Index

```bash
cat .g4a/index.json | python3 -m json.tool
```

### Read Specific Artifact

```bash
cat .g4a/logs/2026-03/20260331-120000-add-gift-card-import.json
```

### Search by Keyword

```bash
grep -r "gift card" .g4a/logs/ --include="*.json" -l
```

### Query Script (if available)

```bash
scripts/g4a-query.py --search "rate limit"
scripts/g4a-query.py --file bulk_operation.py
scripts/g4a-query.py --recent 5
```

## Never Do

- ❌ NEVER finish session without writing `.g4a/.current_reasoning.json`
- ❌ NEVER leave `hypothesis`, `approach`, or `outcome.summary` empty
- ❌ NEVER write placeholder text ("to be determined", "N/A")
- ❌ NEVER skip `alternatives_considered`

## Always Do

- ✅ ALWAYS create reasoning artifact at session start
- ✅ ALWAYS update it at session end
- ✅ ALWAYS include at least one alternative considered
- ✅ ALWAYS provide real summary (not placeholder)

## Integration with Other Skills

Reasoning capture is used by:
- **sparc-developer** — Updates reasoning at each SPARC phase
- **development-workflows** — Captures worktree, plan review, and quality-gate decisions

## References

- [Workflow](references/workflow.md) — Session start/end patterns
- [Learnings](references/learnings.md) — Recording pitfalls and patterns

## Success Criteria

- [ ] Reasoning artifact created at session start
- [ ] All required fields populated
- [ ] `alternatives_considered` has real alternatives
- [ ] `outcome.summary` is meaningful (not placeholder)
- [ ] Artifact updated at session end
- [ ] Stop hook will archive automatically
