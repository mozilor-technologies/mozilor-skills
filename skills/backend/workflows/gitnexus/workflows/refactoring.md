# Refactoring with GitNexus

## When to Use

- "Rename this function safely"
- "Extract this into a module"
- "Split this service"
- "Move this to a new file"
- Any task involving renaming, extracting, splitting, or restructuring code
- Safe large-scale code reorganization

## Workflow

```
1. gitnexus_impact({target: "X", direction: "upstream"})  → Map all dependents
2. gitnexus_query({query: "X"})                            → Find execution flows involving X
3. gitnexus_context({name: "X"})                           → See all incoming/outgoing refs
4. Plan update order: interfaces → implementations → callers → tests
```

> If "Index is stale" → run `npx gitnexus analyze` in terminal.

## Checklists

### Rename Symbol

```
- [ ] gitnexus_rename({symbol_name: "oldName", new_name: "newName", dry_run: true}) — preview all edits
- [ ] Review graph edits (high confidence) and text_search edits (review carefully)
- [ ] If satisfied: gitnexus_rename({..., dry_run: false}) — apply edits
- [ ] gitnexus_detect_changes() — verify only expected files changed
- [ ] Run tests for affected processes
```

### Extract Module

```
- [ ] gitnexus_context({name: target}) — see all incoming/outgoing refs
- [ ] gitnexus_impact({target, direction: "upstream"}) — find all external callers
- [ ] Define new module interface
- [ ] Extract code, update imports
- [ ] gitnexus_detect_changes() — verify affected scope
- [ ] Run tests for affected processes
```

### Split Function/Service

```
- [ ] gitnexus_context({name: target}) — understand all callees
- [ ] Group callees by responsibility
- [ ] gitnexus_impact({target, direction: "upstream"}) — map callers to update
- [ ] Create new functions/services
- [ ] Update callers
- [ ] gitnexus_detect_changes() — verify affected scope
- [ ] Run tests for affected processes
```

### Move to New File

```
- [ ] gitnexus_context({name: target}) — understand dependencies
- [ ] gitnexus_impact({target, direction: "upstream"}) — find all importers
- [ ] Create new file with extracted code
- [ ] Update all imports (use gitnexus_rename if renaming)
- [ ] gitnexus_detect_changes() — verify scope
- [ ] Run tests
```

## Tool Usage

### gitnexus_rename

Automated multi-file rename using knowledge graph + text search.

**Example:**
```javascript
// Preview first
gitnexus_rename({
  symbol_name: "validateUser",
  new_name: "authenticateUser",
  dry_run: true
})
```

**Returns:**
```
edits: 12 across 8 files

graph edits (high confidence): 10
  - src/auth/validator.ts:23: function validateUser → authenticateUser
  - src/auth/login.ts:45: validateUser(data) → authenticateUser(data)
  - src/api/middleware.ts:67: import { validateUser } → import { authenticateUser }
  ...

text_search edits (review carefully): 2
  - config.json:12: "validator": "validateUser" → "authenticateUser"
  - README.md:89: `validateUser()` → `authenticateUser()`

files_affected:
  - src/auth/validator.ts
  - src/auth/login.ts
  - src/api/middleware.ts
  - config.json
  - README.md
  ...
```

**Parameters:**
- `symbol_name` (optional): Current symbol name to rename
- `symbol_uid` (optional): Direct symbol UID from prior results (zero-ambiguity)
- `new_name` (required): The new name for the symbol
- `file_path` (optional): File path to disambiguate common names
- `dry_run` (optional): Preview edits without modifying files (default: true)
- `repo` (optional): Repository name if multiple repos indexed

**When to use:**
- Renaming functions, classes, methods, variables
- When symbol has multiple references across files
- Safer than find-and-replace (understands code structure)

**Workflow:**
1. Always preview first with `dry_run: true`
2. Review graph edits (high confidence, safe to accept)
3. Review text_search edits carefully (might include comments, strings, config)
4. Apply with `dry_run: false` if satisfied
5. Run `gitnexus_detect_changes` to verify
6. Test affected processes

**Confidence tags:**
- `graph`: Found via knowledge graph relationships (high confidence, safe)
- `text_search`: Found via regex text search (lower confidence, review carefully)

**Notes:**
- Graph-based edits understand imports, call chains, and produce high-confidence changes
- Text search catches dynamic references, configs, documentation
- Always review text_search edits before applying

### gitnexus_impact

Map all dependents before refactoring to understand what will be affected.

**Example:**
```javascript
gitnexus_impact({
  target: "validateUser",
  direction: "upstream"
})
```

**Returns:**
```
risk: MEDIUM
summary: 5 direct callers, 2 processes affected

byDepth:
  d=1: loginHandler, apiMiddleware, testUtils
  d=2: authRouter, sessionManager

affected_processes:
  - LoginFlow
  - TokenRefresh
```

**Use before:**
- Changing function signatures
- Moving code to new files
- Extracting modules
- Any refactoring that might break callers

**Risk Rules:**
| Risk Factor         | Mitigation                                |
| ------------------- | ----------------------------------------- |
| Many callers (>5)   | Use gitnexus_rename for automated updates |
| Cross-area refs     | Use detect_changes after to verify scope  |
| String/dynamic refs | gitnexus_query to find them               |
| External/public API | Version and deprecate properly            |

### gitnexus_detect_changes

Verify your changes after refactoring to ensure expected scope.

**Example:**
```javascript
gitnexus_detect_changes({scope: "all"})
```

**Returns:**
```
changed_symbols: 8 symbols in 12 files
affected_processes: LoginFlow, TokenRefresh
risk: MEDIUM
```

**Use after:**
- Applying rename
- Extracting modules
- Moving code
- Any refactoring to verify scope

**What to check:**
- Only expected files changed?
- Affected processes match expectations?
- No unexpected side effects?
- Risk level acceptable?

### gitnexus_context

Understand a symbol fully before refactoring.

**Example:**
```javascript
gitnexus_context({name: "validateUser"})
```

**Returns:**
```
Incoming: who calls this (will need updates if interface changes)
Outgoing: what this calls (will move with the code)
Processes: which flows will be affected
```

**Use to:**
- Understand what depends on the code
- See what the code depends on
- Plan extraction boundaries
- Identify coupling

### gitnexus_cypher

Custom reference queries for complex refactoring analysis.

**Example:**
```cypher
# Find all callers of a function, ordered by file
MATCH (caller)-[:CodeRelation {type: 'CALLS'}]->(f:Function {name: "validateUser"})
RETURN caller.name, caller.filePath
ORDER BY caller.filePath
```

**Common patterns:**
```cypher
# Find circular dependencies before refactoring
MATCH path = (a)-[:CodeRelation {type: 'IMPORTS'}*2..]->(a)
RETURN [n IN nodes(path) | n.name] AS cycle

# Find all members of a class to extract
MATCH (c:Class {name: "UserService"})-[:HAS_METHOD]->(m)
RETURN m.name, m.filePath

# Find cross-module dependencies
MATCH (a)-[:CodeRelation]->(b)
WHERE a.filePath CONTAINS '/moduleA/' AND b.filePath CONTAINS '/moduleB/'
RETURN a.name, b.name, type
```

## Example: Rename `validateUser` to `authenticateUser`

### Step 1: Impact analysis
```javascript
gitnexus_impact({
  target: "validateUser",
  direction: "upstream"
})
```

**Response:**
```
risk: MEDIUM
d=1: loginHandler, apiMiddleware (2 callers)
affected_processes: LoginFlow, TokenRefresh
```

### Step 2: Preview rename
```javascript
gitnexus_rename({
  symbol_name: "validateUser",
  new_name: "authenticateUser",
  dry_run: true
})
```

**Response:**
```
12 edits across 8 files

graph edits: 10 (high confidence)
  - validator.ts: function definition
  - login.ts: function call
  - middleware.ts: import statement
  ...

text_search edits: 2 (review)
  - config.json: dynamic reference in config
  - README.md: documentation mention
```

### Step 3: Review edits
Review the 2 text_search edits:
- config.json: Yes, should be updated
- README.md: Yes, documentation should match

### Step 4: Apply rename
```javascript
gitnexus_rename({
  symbol_name: "validateUser",
  new_name: "authenticateUser",
  dry_run: false
})
```

**Response:**
```
Applied 12 edits across 8 files
```

### Step 5: Verify changes
```javascript
gitnexus_detect_changes({scope: "all"})
```

**Response:**
```
changed_symbols: 1 (authenticateUser, renamed from validateUser)
files_modified: 8
affected_processes: LoginFlow, TokenRefresh
risk: MEDIUM
```

### Step 6: Test
```bash
npm test -- --grep "LoginFlow|TokenRefresh"
```

## Refactoring Patterns

### Safe Symbol Rename
```
1. gitnexus_impact to assess blast radius
2. gitnexus_rename with dry_run: true to preview
3. Review graph vs text_search edits
4. Apply with dry_run: false
5. gitnexus_detect_changes to verify
6. Test affected processes
```

### Extract Module
```
1. gitnexus_context on symbols to extract
2. Identify clean boundary (minimal outgoing refs)
3. gitnexus_impact to find all callers
4. Create new module file
5. Move code and update imports
6. gitnexus_detect_changes to verify
7. Test affected processes
```

### Split Large Function
```
1. gitnexus_context to see what function does
2. Group callees by responsibility
3. gitnexus_impact to find callers
4. Create new functions for each responsibility
5. Update callers
6. gitnexus_detect_changes to verify
7. Test affected processes
```

### Move Code to New File
```
1. gitnexus_context to understand dependencies
2. gitnexus_impact to find importers (d=1)
3. Create new file
4. Move code
5. Update imports at all d=1 locations
6. gitnexus_detect_changes to verify
7. Test
```

### Change Function Signature
```
1. gitnexus_impact to find all callers (d=1)
2. If >5 callers: consider keeping old signature
3. If proceeding: update interface
4. Update all d=1 callers
5. gitnexus_detect_changes to verify
6. Test all affected processes
```

### Extract Class
```
1. gitnexus_context on methods to extract
2. Use cypher to find all class members
3. gitnexus_impact on each public method
4. Define new class interface
5. Move methods to new class
6. Update callers
7. gitnexus_detect_changes to verify
```

## Risk Management

### Before Refactoring
- [ ] Run impact analysis to understand blast radius
- [ ] Check for >10 d=1 callers (CRITICAL risk)
- [ ] Check for critical paths (auth, billing, payments)
- [ ] Plan update strategy for all affected code

### During Refactoring
- [ ] Preserve interfaces when possible
- [ ] Update all d=1 callers if interface changes
- [ ] Use gitnexus_rename for symbol renames
- [ ] Keep commits small and focused

### After Refactoring
- [ ] Run detect_changes to verify scope
- [ ] No unexpected files modified
- [ ] Affected processes match expectations
- [ ] Run full test suite for affected processes
- [ ] Update documentation

## Decision Trees

### Should I Use gitnexus_rename?
```
If renaming symbol AND:
  - Multiple files reference it → YES
  - Complex codebase → YES
  - Want high confidence → YES
  - Simple find-replace sufficient → NO (but still safer with tool)
```

### Should I Preserve Interface?
```
d=1 callers:
  0-2   → Safe to change
  3-5   → Consider preserving
  6-10  → Strongly consider preserving
  >10   → Preserve or plan migration
```

### Should I Extract Module?
```
If:
  - Clear responsibility boundary → YES
  - <5 cross-boundary calls → YES
  - Improves cohesion → YES
  - Would create circular deps → NO
```

## Common Scenarios

### Rename for Clarity
```javascript
// Function name unclear
gitnexus_rename({symbol_name: "process", new_name: "processPayment", dry_run: true})
```

### Extract Shared Logic
```
1. Identify duplicated code
2. gitnexus_context on each duplicate
3. Create shared utility
4. Update all call sites
5. verify with detect_changes
```

### Split God Class
```
1. Use cypher to list all methods
2. Group methods by responsibility
3. impact analysis on each public method
4. Create new classes
5. Move methods
6. Update callers
```

### Consolidate Similar Functions
```
1. gitnexus_query to find similar functions
2. context on each to understand usage
3. impact to find all callers
4. Create unified function
5. Update all callers
6. Delete old functions
```

### Move to Better Location
```
1. context to understand dependencies
2. impact to find importers
3. Move code to new file
4. Update all imports
5. detect_changes to verify
```

## Advanced Techniques

### Batch Rename Related Symbols
```javascript
// Rename a family of related functions
for (const symbol of ["validateUser", "validatePayment", "validateOrder"]) {
  gitnexus_rename({
    symbol_name: symbol,
    new_name: symbol.replace("validate", "verify"),
    dry_run: false
  });
}
```

### Extract with Interface Preservation
```
// Keep old function as wrapper
1. Extract new function with better name
2. Update old function to call new function
3. Deprecate old function
4. Gradually migrate callers
```

### Identify Refactoring Candidates
```cypher
# Find functions with many callers (refactoring priority)
MATCH (f:Function)<-[:CodeRelation {type: 'CALLS'}]-(caller)
WITH f, count(caller) as callerCount
WHERE callerCount > 10
RETURN f.name, f.filePath, callerCount
ORDER BY callerCount DESC
```

### Find Tightly Coupled Modules
```cypher
# Find cross-module dependencies
MATCH (a)-[r:CodeRelation]->(b)
WHERE a.filePath CONTAINS '/moduleA/' AND b.filePath CONTAINS '/moduleB/'
RETURN type(r), count(*) as count
ORDER BY count DESC
```

## Integration with Other Workflows

### Refactoring + Impact Analysis
```
1. Always run impact before refactoring
2. Understand blast radius
3. Plan updates for all d=1 callers
4. Verify with detect_changes after
```

### Refactoring + Exploring
```
1. Use exploring to understand code first
2. Identify refactoring opportunities
3. Use impact to assess safety
4. Refactor with knowledge
```

### Refactoring + Debugging
```
1. Debug to find root cause
2. Impact analysis before fix
3. Refactor safely to address root cause
4. Verify with detect_changes
```

## Tips

### Always Preview First
Use `dry_run: true` to see edits before applying. Review carefully.

### Graph Edits Are Safer
Graph-based edits have high confidence. Text search edits need review.

### Update Tests Last
Update production code first, then update tests to match.

### Keep Commits Focused
One refactoring per commit. Easier to review and revert.

### Test Affected Processes
Run tests for all affected execution flows, not just unit tests.

### Watch for Dynamic References
Config files, string templates, dynamic imports might need manual updates.

### Preserve Interface When Possible
If many callers, consider keeping the old interface and creating new functions.

### Use detect_changes Before Committing
Always verify the scope of your changes matches expectations.

## Related Workflows

- [Impact Analysis](impact-analysis.md) - Mandatory before refactoring
- [Exploring](exploring.md) - Understand code before refactoring
- [Debugging](debugging.md) - Debug before refactoring
- [CLI Reference](cli-reference.md) - Reindex after major refactoring
