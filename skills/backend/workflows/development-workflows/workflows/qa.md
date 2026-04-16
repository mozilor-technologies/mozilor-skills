# QA Workflow

Use this workflow for review-style assessment before merge.

## Goal

Identify correctness, reliability, security, and maintainability risks and report findings by severity.

## Review procedure

1. Enumerate actual changes (`gitnexus_detect_changes` or git diff)
2. Assess blast radius on modified symbols
3. Review for correctness and regressions
4. Review security and data-safety concerns
5. Review architecture/pattern compliance
6. Review test adequacy
7. Summarize findings with severity ordering

## Severity model

- Blocking: must fix before merge
- Warning: should fix soon
- Suggestion: optional improvement

## Review checklist

Correctness:

- [ ] Logic matches requirements
- [ ] Edge cases handled
- [ ] Error paths preserve invariants

Security:

- [ ] Input validation at boundaries
- [ ] No sensitive data leakage
- [ ] Auth/permission checks intact

Code quality:

- [ ] Consistent with repository patterns
- [ ] Complexity is justified and understandable
- [ ] Dead code and temporary logic removed

Tests:

- [ ] Critical paths covered
- [ ] Regressions protected by tests
- [ ] Test intent is clear

## Output format

1. Findings first (highest severity first)
2. Open questions/assumptions
3. Short change summary
4. Residual risk/testing gaps

## Decision tree

```text
Review complete
  |
  +-- Blocking findings? -- Yes --> stop merge, request fixes
  |
  +-- No
       |
       +-- Warnings only? -- Yes --> allow merge with follow-up
       |
       +-- None --> approve from QA perspective
```

## Example output skeleton

```text
Findings:
1) [Blocking] <issue> in <file/area> with impact <impact>
2) [Warning] <issue>

Open questions:
1) <question>

Summary:
- <short summary>

Residual risk:
- <risk or "none identified">
```

## Reasoning capture payload

```json
{
  "agent": "qa",
  "reasoning": {
    "hypothesis": "Initial risk assessment from change detection"
  },
  "impact": {
    "risk_level": "LOW|MEDIUM|HIGH"
  },
  "outcome": {
    "follow_up": "List unresolved warnings/suggestions"
  }
}
```

## Success checklist

- [ ] Changes and impact analyzed
- [ ] Findings severity-ranked
- [ ] Test/lint state considered
- [ ] Merge recommendation explicit
