# Impact Analysis with GitNexus

## When to Use

- "Is it safe to change this function?"
- "What will break if I modify X?"
- "Show me the blast radius"
- "Who uses this code?"
- Before making non-trivial code changes
- Before committing — to understand what your changes affect

## Workflow

```
1. gitnexus_impact({target: "X", direction: "upstream"})  → What depends on this
2. READ gitnexus://repo/{name}/processes                   → Check affected execution flows
3. gitnexus_detect_changes()                               → Map current git changes to affected flows
4. Assess risk and report to user
```

> If "Index is stale" → run `npx gitnexus analyze` in terminal.

## Checklist

```
- [ ] gitnexus_impact({target, direction: "upstream"}) to find dependents
- [ ] Review d=1 items first (these WILL BREAK)
- [ ] Check high-confidence (>0.8) dependencies
- [ ] READ processes to check affected execution flows
- [ ] gitnexus_detect_changes() for pre-commit check
- [ ] Assess risk level and report to user
```

## Understanding Output

### Depth Levels

| Depth | Risk Level       | Meaning                  | What to do                           |
| ----- | ---------------- | ------------------------ | ------------------------------------ |
| d=1   | **WILL BREAK**   | Direct callers/importers | MUST update these if interface changes |
| d=2   | LIKELY AFFECTED  | Indirect dependencies    | Test thoroughly after changes        |
| d=3   | MAY NEED TESTING | Transitive effects       | Run integration tests                |

### Confidence Levels

- **1.0 (100%)**: Certain relationship, found via static analysis
- **0.8-0.99**: High confidence, likely correct
- **<0.8**: Fuzzy match, review carefully

## Risk Assessment

| Affected                       | Risk     | Required Action                    |
| ------------------------------ | -------- | ---------------------------------- |
| <5 symbols, few processes      | LOW      | Proceed, update callers            |
| 5-15 symbols, 2-5 processes    | MEDIUM   | Proceed carefully, test thoroughly |
| >15 symbols or many processes  | HIGH     | Consider preserving interface      |
| Critical path (auth, payments) | CRITICAL | **STOP** — ask user before proceeding |

## Tool Usage

### gitnexus_impact

The primary tool for symbol blast radius analysis.

**Example:**
```javascript
gitnexus_impact({
  target: "validateUser",
  direction: "upstream",
  minConfidence: 0.8,
  maxDepth: 3
})
```

**Returns:**
```
risk: MEDIUM
summary: 5 direct callers, 2 processes affected, 1 module affected

affected_processes:
  - LoginFlow (step 2/5): loginHandler → validateUser → checkToken
  - TokenRefresh (step 1/3): refreshHandler → validateUser → verifyToken

affected_modules:
  - auth (direct)
  - api (indirect via middleware)

byDepth:
  d=1 (WILL BREAK):
    - loginHandler (src/auth/login.ts:42) [CALLS, 100%]
    - apiMiddleware (src/api/middleware.ts:15) [CALLS, 100%]

  d=2 (LIKELY AFFECTED):
    - authRouter (src/routes/auth.ts:22) [CALLS, 95%]
    - sessionManager (src/auth/sessions.ts:67) [CALLS, 90%]
```

**Parameters:**
- `target` (required): Symbol name or file path to analyze
- `direction` (required): "upstream" (what depends on this) or "downstream" (what this depends on)
- `maxDepth` (optional): Max relationship depth (default: 3)
- `minConfidence` (optional): Minimum confidence 0-1 (default: 0.7)
- `includeTests` (optional): Include test files (default: false)
- `relationTypes` (optional): Filter edge types - CALLS, IMPORTS, EXTENDS, IMPLEMENTS, HAS_METHOD, HAS_PROPERTY, OVERRIDES, ACCESSES
- `repo` (optional): Repository name if multiple repos indexed

**When to use:**
- Before modifying any existing symbol
- To understand blast radius of a change
- To decide if interface preservation is needed
- To identify all code that must be updated

**Notes:**
- Default relationTypes use CALLS/IMPORTS/EXTENDS/IMPLEMENTS (usage-based)
- For class members, include HAS_METHOD and HAS_PROPERTY
- For field access analysis, include ACCESSES
- ACCESSES excluded by default to focus on direct usage

### gitnexus_detect_changes

Git-diff based impact analysis - maps your uncommitted changes to affected symbols and processes.

**Example:**
```javascript
gitnexus_detect_changes({scope: "staged"})
```

**Returns:**
```
changed_symbols:
  - validateUser (src/auth/validator.ts:23): modified
  - checkToken (src/auth/tokens.ts:45): modified
  - UserSchema (src/models/user.ts:12): modified

affected_processes:
  - LoginFlow: validateUser (step 2/5), checkToken (step 3/5)
  - TokenRefresh: checkToken (step 2/3)
  - APIMiddlewarePipeline: validateUser (step 1/4)

risk_summary:
  risk: MEDIUM
  symbols_changed: 5
  processes_affected: 3
  files_modified: 3
```

**Parameters:**
- `scope` (optional): What to analyze - "unstaged" (default), "staged", "all", or "compare"
- `base_ref` (optional): Branch/commit for "compare" scope (e.g., "main")
- `repo` (optional): Repository name if multiple repos indexed

**When to use:**
- Before committing changes
- To verify your changes affect only expected code
- To identify which processes need testing
- To catch unexpected side effects

**Scope options:**
- `unstaged`: Analyze working directory changes not yet staged
- `staged`: Analyze changes in staging area (git add)
- `all`: Analyze all uncommitted changes (staged + unstaged)
- `compare`: Compare current branch to base_ref (e.g., main)

## Example: "What breaks if I change validateUser?"

### Step 1: Run impact analysis
```javascript
gitnexus_impact({
  target: "validateUser",
  direction: "upstream"
})
```

**Response:**
```
risk: MEDIUM

byDepth:
  d=1 (WILL BREAK):
    - loginHandler (src/auth/login.ts:42) [CALLS, 100%]
    - apiMiddleware (src/api/middleware.ts:15) [CALLS, 100%]

  d=2 (LIKELY AFFECTED):
    - authRouter (src/routes/auth.ts:22) [CALLS, 95%]
    - sessionManager (src/auth/sessions.ts:67) [CALLS, 90%]

affected_processes:
  - LoginFlow
  - TokenRefresh
```

### Step 2: Check affected processes
```
READ gitnexus://repo/my-app/processes
→ See full list of execution flows

READ gitnexus://repo/my-app/process/LoginFlow
→ LoginFlow (5 steps):
  1. authRouter → loginHandler
  2. loginHandler → validateUser  ← STEP AFFECTED
  3. validateUser → checkToken
  4. checkToken → getUserById
  5. loginHandler → createSession
```

### Step 3: Assess risk

**Analysis:**
- 2 direct callers (d=1): loginHandler, apiMiddleware
- 2 indirect callers (d=2): authRouter, sessionManager
- 2 affected processes: LoginFlow, TokenRefresh
- Critical path: Authentication (CRITICAL)

**Risk: CRITICAL** (authentication is critical path)

**Recommendation:**
- If changing signature: Update both d=1 callers (loginHandler, apiMiddleware)
- If preserving signature: Proceed with internal changes
- Test: Run LoginFlow and TokenRefresh test suites
- **STOP**: Ask user before proceeding (critical path)

## Patterns

### Pre-Edit Safety Check
```javascript
// Before editing existingFunction
gitnexus_impact({target: "existingFunction", direction: "upstream"})

// Assess:
// - How many d=1 callers?
// - High confidence or fuzzy matches?
// - Affected processes critical?
// - Decision: preserve interface or update callers?
```

### Pre-Commit Verification
```javascript
// After making changes, before committing
gitnexus_detect_changes({scope: "all"})

// Verify:
// - Only expected files changed?
// - Affected processes match expectations?
// - No unexpected side effects?
```

### Downstream Dependency Analysis
```javascript
// What does this symbol depend on?
gitnexus_impact({target: "myFunction", direction: "downstream"})

// Use case:
// - Understanding what myFunction needs to work
// - Finding external dependencies
// - Identifying coupling to other modules
```

### Field Access Impact
```javascript
// Include ACCESSES to track field read/write
gitnexus_impact({
  target: "User.email",
  direction: "upstream",
  relationTypes: ["ACCESSES", "CALLS"]
})

// Shows: who reads/writes User.email field
```

### Class Member Impact
```javascript
// Include HAS_METHOD and HAS_PROPERTY for class analysis
gitnexus_impact({
  target: "UserService",
  direction: "upstream",
  relationTypes: ["CALLS", "IMPORTS", "EXTENDS", "IMPLEMENTS", "HAS_METHOD", "HAS_PROPERTY"]
})

// Shows: full class structure impact
```

## Decision Trees

### When d=1 has 0-2 callers (LOW risk)
```
→ Safe to modify
→ Update the callers
→ Run affected process tests
→ Proceed with changes
```

### When d=1 has 3-5 callers (MEDIUM risk)
```
→ Proceed carefully
→ Consider preserving interface if signature change
→ Update all callers if proceeding
→ Run full test suite for affected processes
```

### When d=1 has 6-10 callers (HIGH risk)
```
→ Strongly consider preserving interface
→ If must change signature: plan callers update
→ Consider versioning/deprecation
→ Run integration tests
```

### When d=1 has >10 callers (CRITICAL risk)
```
→ **STOP** — report to user before proceeding
→ Likely need to preserve interface
→ Consider adapter pattern or facade
→ Plan migration strategy if signature must change
```

### When touching auth/billing/payments (CRITICAL risk)
```
→ **STOP** — report to user before proceeding
→ Extra review required
→ Security implications
→ Full test coverage required
```

## Common Scenarios

### "Is it safe to rename this function?"
```javascript
gitnexus_impact({target: "functionName", direction: "upstream"})
// If few callers: use gitnexus_rename
// If many callers: preserve name or plan migration
```

### "Can I change this function signature?"
```javascript
gitnexus_impact({target: "functionName", direction: "upstream"})
// Check d=1 callers - these MUST be updated
// If >5 callers: consider overload or new function
```

### "What will my current changes affect?"
```javascript
gitnexus_detect_changes({scope: "all"})
// Review affected processes
// Verify expected scope
// Run tests for affected flows
```

### "What happens if I delete this function?"
```javascript
gitnexus_impact({target: "functionName", direction: "upstream"})
// d=1 callers will break
// d=2 callers likely affected
// Ensure no callers before deleting
```

## Integration with Other Workflows

### With Refactoring
```
1. gitnexus_impact → understand dependents
2. gitnexus_rename → safe multi-file rename
3. gitnexus_detect_changes → verify scope
```

### With Debugging
```
1. gitnexus_context → understand symbol
2. gitnexus_impact → find affected code
3. Fix bug with knowledge of impact
```

### With Exploring
```
1. gitnexus_query → find execution flows
2. gitnexus_impact → understand dependencies
3. Make informed architectural decisions
```

## Tips

### Focus on d=1 First
d=1 items WILL BREAK if you change the interface. Review these carefully before proceeding.

### Confidence Matters
- 1.0 (100%): Definitely will break
- 0.8-0.99: Very likely will break
- <0.8: Might break, review code to confirm

### Critical Paths
Auth, billing, payments, security are CRITICAL. Always stop and ask user before proceeding.

### Use detect_changes Before Committing
Make it a habit to run detect_changes before git commit to catch unexpected side effects.

### Preserve Interfaces When Possible
If >5 d=1 callers, seriously consider preserving the interface instead of changing it.

## Related Workflows

- [Exploring](exploring.md) - Understand code before changing it
- [Refactoring](refactoring.md) - Safe code restructuring
- [Debugging](debugging.md) - Trace bugs with impact awareness
