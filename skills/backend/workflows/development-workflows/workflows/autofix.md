# Autofix Workflow

Use this workflow when the user asks for autonomous end-to-end fixing.

## Preconditions

- Clear issue statement exists
- Repro/test path is available
- Sandbox tooling is available
- Safe to execute unattended within defined scope

If preconditions are not met, fall back to `debugging.md` and request missing context.

## Execution flow

1. Create isolated sandbox instance
2. Sync repository into sandbox
3. Reproduce issue and gather evidence
4. Identify root cause (not symptom)
5. Implement root-cause fix
6. Run tests/lint in sandbox
7. Sync validated changes back
8. Run local quality gates
9. Commit or hand off with explicit status
10. Destroy sandbox

## Guardrails

Forbidden in autofix:

- workaround-only changes
- adding limits/timeouts/retries to hide defects
- skipping tests to force completion

Required:

- root-cause elimination
- reproducible verification
- explicit report of what changed and why

Suggested sandbox operations:

- create instance
- sync workspace to instance
- run debug/fix/verify loop
- sync instance changes back
- destroy instance

## Decision tree

```text
Autofix requested
  |
  +-- Preconditions met? -- No --> switch to guided debugging
  |
  +-- Yes
       |
       +-- Reproduce issue? -- No --> collect more evidence; do not guess-fix
       |
       +-- Yes
            |
            +-- Root cause confirmed? -- No --> continue investigation loop
            |
            +-- Yes --> fix -> verify -> sync -> quality gates
```

## Verification requirements

- [ ] Issue no longer reproduces
- [ ] Relevant tests pass
- [ ] No new failing tests/lint errors
- [ ] Scope of changes is intentional
- [ ] Final quality gates completed

## Example autonomous cycle

```text
- create sandbox
- run failing test to reproduce
- trace root cause with code context
- apply fix
- rerun failing test + focused suite
- run lint
- sync patch to workspace
- run full quality gates locally
- report outcome and residual risks
```

## Output format

Provide concise execution report:

- problem summary
- confirmed root cause
- files changed and why
- verification evidence (tests/checks)
- residual risks or follow-up tasks

## Reasoning capture payload

```json
{
  "agent": "autofix",
  "reasoning": {
    "hypothesis": "Suspected root cause",
    "approach": "Autonomous execution in sandbox"
  },
  "outcome": {
    "summary": "Fix applied with verification evidence",
    "tests_passed": true
  }
}
```

## Success checklist

- [ ] Autonomous run stayed within scope
- [ ] Root cause fixed (no band-aid)
- [ ] Verification passed in sandbox and locally
- [ ] Cleanup completed
