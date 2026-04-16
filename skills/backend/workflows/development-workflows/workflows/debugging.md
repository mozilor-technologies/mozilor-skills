# Debugging Workflow

Use this workflow for incidents, regressions, and unexpected behavior.

## Principle

Fix root causes only.

Forbidden band-aids:

- adding protective limits around broken logic
- adding retries/timeouts instead of fixing flakiness causes
- adding graceful degradation for fixable bugs

Required:

- identify the architectural/code cause
- change implementation so the failure mode is removed

## Debug flow

1. Capture symptom precisely
2. Locate related code paths (`gitnexus_query`)
3. Trace callers/callees (`gitnexus_context`)
4. Check recent regressions (`gitnexus_detect_changes` vs main)
5. Gather evidence (logs, payloads, state)
6. Form hypotheses and disprove alternatives
7. Confirm root cause
8. Implement root-cause fix
9. Verify with tests and reproduction steps

## Decision tree

```text
Issue reported
  |
  +-- Reproducible? ----- No --> gather more evidence/instrumentation
  |
  +-- Yes
       |
       +-- Root cause confirmed? ---- No --> continue hypothesis loop
       |
       +-- Yes
            |
            +-- Fix removes failure mode? -- No --> reject as band-aid
            |
            +-- Yes --> implement + verify
```

## Investigation checklist

- [ ] Symptom, expected behavior, and actual behavior documented
- [ ] Reproduction path identified
- [ ] Impacted symbols/processes mapped
- [ ] Alternative theories ruled out with evidence
- [ ] Root cause statement is falsifiable and tested

## Fix checklist

- [ ] Change directly targets root cause
- [ ] No workaround-only logic introduced
- [ ] Regression tests added/updated
- [ ] Quality gates run after fix

## Reasoning capture payload

Use a concise structured record for debugging sessions:

```json
{
  "agent": "debugger",
  "reasoning": {
    "hypothesis": "Initial suspected cause",
    "alternatives_considered": [
      "Alternative theory A and evidence that ruled it out"
    ]
  },
  "outcome": {
    "summary": "Confirmed root cause and implemented fix"
  }
}
```

## Example

Symptom: export job crashes on large datasets.

Bad fix (reject): cap rows and fail early.

Correct fix: stream rows directly to sink, removing in-memory accumulation path.

## Handoff

After fix verification:

1. Run `quality-gates.md`
2. If needed, run `qa.md` for pre-merge risk review
3. Document cause and resolution in issue/PR notes
