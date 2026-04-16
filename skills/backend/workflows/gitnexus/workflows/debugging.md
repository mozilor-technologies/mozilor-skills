# Debugging with GitNexus

## When to Use

- "Why is this function failing?"
- "Trace where this error comes from"
- "Who calls this method?"
- "This endpoint returns 500"
- Investigating bugs, errors, or unexpected behavior
- Tracing execution paths that lead to failures

## Workflow

```
1. gitnexus_query({query: "<error or symptom>"})            → Find related execution flows
2. gitnexus_context({name: "<suspect>"})                    → See callers/callees/processes
3. READ gitnexus://repo/{name}/process/{name}                → Trace execution flow
4. gitnexus_cypher({query: "MATCH path..."})                 → Custom traces if needed
```

> If "Index is stale" → run `npx gitnexus analyze` in terminal.

## Checklist

```
- [ ] Understand the symptom (error message, unexpected behavior)
- [ ] gitnexus_query for error text or related code
- [ ] Identify the suspect function from returned processes
- [ ] gitnexus_context to see callers and callees
- [ ] Trace execution flow via process resource if applicable
- [ ] gitnexus_cypher for custom call chain traces if needed
- [ ] Read source files to confirm root cause
```

## Debugging Patterns

| Symptom              | GitNexus Approach                                          |
| -------------------- | ---------------------------------------------------------- |
| Error message        | `gitnexus_query` for error text → `context` on throw sites |
| Wrong return value   | `context` on the function → trace callees for data flow    |
| Intermittent failure | `context` → look for external calls, async deps            |
| Performance issue    | `context` → find symbols with many callers (hot paths)     |
| Recent regression    | `detect_changes` to see what your changes affect           |
| Unexpected behavior  | `query` for feature → trace process → find divergence      |
| Null/undefined error | `context` on the function → trace data flow backward       |

## Tool Usage

### gitnexus_query

Find code related to an error message or symptom using semantic search.

**Example:**
```javascript
gitnexus_query({query: "payment validation error"})
```

**Returns:**
```
Processes: CheckoutFlow, ErrorHandling
Symbols:
  - validatePayment (src/payments/validator.ts:23)
  - handlePaymentError (src/errors/payment.ts:45)
  - PaymentException (src/exceptions/payment.ts:12)
```

**Parameters:**
- `query` (required): Error text, symptom, or related concept
- `repo` (optional): Repository name if multiple repos indexed
- `limit` (optional): Max number of results (default: 10)
- `threshold` (optional): Similarity threshold 0-1 (default: 0.7)

**When to use:**
- Starting point when you have an error message
- Finding code related to a symptom or behavior
- Discovering which processes might be involved

**Tips:**
- Include specific error text: "TypeError: Cannot read property 'x'"
- Use domain terms: "payment validation failure"
- Try symptom descriptions: "checkout returns 500"

### gitnexus_context

Full context for a suspect function - see all callers, callees, and processes.

**Example:**
```javascript
gitnexus_context({name: "validatePayment"})
```

**Returns:**
```
Incoming calls:
  - processCheckout (src/payments/checkout.ts:67) [CALLS, 100%]
  - webhookHandler (src/api/webhooks.ts:45) [CALLS, 100%]

Outgoing calls:
  - verifyCard (src/payments/card.ts:23) [CALLS, 100%]
  - fetchRates (src/external/rates.ts:12) [CALLS, 100%]  ← External API!

Processes:
  - CheckoutFlow (step 3/7)
  - WebhookHandler (step 2/4)
```

**Parameters:**
- `name` (optional): Symbol name
- `uid` (optional): Direct symbol UID for zero-ambiguity lookup
- `file_path` (optional): File path to disambiguate
- `include_content` (optional): Include source code (default: false)
- `repo` (optional): Repository name if multiple repos indexed

**When to use:**
- Understanding what calls a failing function
- Tracing what a function calls (data flow)
- Finding external dependencies that might fail
- Seeing which execution flows touch the suspect

**Notes:**
- ACCESSES edges included with reason 'read' or 'write'
- Look for external API calls (common source of intermittent failures)
- Check for async operations that might race

### gitnexus_cypher

Custom graph queries for advanced call chain tracing.

**Example:**
```cypher
MATCH path = (a)-[:CodeRelation {type: 'CALLS'}*1..3]->(b:Function {name: "validatePayment"})
RETURN [n IN nodes(path) | n.name] AS chain, length(path) AS depth
ORDER BY depth ASC
```

**Returns:**
```
chain: ["processCheckout", "validatePayment"], depth: 1
chain: ["checkoutHandler", "processCheckout", "validatePayment"], depth: 2
chain: ["apiRouter", "checkoutHandler", "processCheckout", "validatePayment"], depth: 3
```

**Parameters:**
- `query` (required): Cypher query string
- `repo` (optional): Repository name if multiple repos indexed

**When to use:**
- Finding all paths to a function
- Tracing call chains of specific depth
- Complex relationship queries beyond basic tools
- Understanding circular dependencies

**Common patterns:**
```cypher
# Find all paths to a function (up to 3 hops)
MATCH path = (a)-[:CodeRelation {type: 'CALLS'}*1..3]->(target:Function {name: "X"})
RETURN path

# Find what throws a specific error
MATCH (f)-[:CodeRelation {type: 'THROWS'}]->(e {name: "PaymentError"})
RETURN f.name, f.filePath

# Find external dependencies
MATCH (f)-[:CodeRelation {type: 'CALLS'}]->(ext)
WHERE ext.filePath CONTAINS 'node_modules'
RETURN f.name, ext.name
```

## Example: "Payment endpoint returns 500 intermittently"

### Step 1: Query for payment errors
```javascript
gitnexus_query({query: "payment error handling"})
```

**Response:**
```
Processes: CheckoutFlow, ErrorHandling
Symbols:
  - validatePayment (src/payments/validator.ts:23)
  - handlePaymentError (src/errors/payment.ts:45)
  - processPayment (src/payments/processor.ts:67)
```

### Step 2: Context on validatePayment
```javascript
gitnexus_context({name: "validatePayment"})
```

**Response:**
```
Incoming calls:
  - processCheckout (src/payments/checkout.ts:67)
  - webhookHandler (src/api/webhooks.ts:45)

Outgoing calls:
  - verifyCard (src/payments/card.ts:23)
  - fetchRates (src/external/rates.ts:12)  ← External API!

Processes:
  - CheckoutFlow (step 3/7)
```

### Step 3: Trace CheckoutFlow
```
READ gitnexus://repo/my-app/process/CheckoutFlow
```

**Response:**
```
CheckoutFlow (7 steps):
1. apiRouter → checkoutHandler
2. checkoutHandler → processCheckout
3. processCheckout → validatePayment
4. validatePayment → fetchRates  ← External API call
5. validatePayment → verifyCard
6. processCheckout → chargePayment
7. checkoutHandler → sendConfirmation
```

### Step 4: Read source code
```
Read src/external/rates.ts
```

**Finding:**
```typescript
async function fetchRates() {
  const response = await fetch('https://api.example.com/rates');
  return await response.json();  // No timeout! No error handling!
}
```

**Root cause:** fetchRates calls external API without proper timeout, causing intermittent failures when API is slow.

## Debugging Workflows

### Error Message Debugging
```
1. gitnexus_query({query: "error message text"})
2. Identify suspect functions from results
3. gitnexus_context({name: "suspect"}) for each
4. Look for throw sites or error propagation
5. Read source to understand conditions
```

### Wrong Return Value Debugging
```
1. gitnexus_context({name: "functionReturningWrongValue"})
2. Check outgoing calls (callees) for data sources
3. Trace data flow through callees
4. gitnexus_context on callees to find data origin
5. Read source to find transformation logic
```

### Intermittent Failure Debugging
```
1. gitnexus_context({name: "intermittentlyFailing"})
2. Look for external API calls in outgoing calls
3. Look for async operations without proper handling
4. Check for race conditions (multiple async callees)
5. Read source to verify timeout/retry logic
```

### Performance Issue Debugging
```
1. gitnexus_context({name: "slowFunction"})
2. Check incoming calls - is this a hot path?
3. Check outgoing calls - any N+1 queries?
4. Use cypher to find call chain depth
5. Look for loops calling expensive operations
```

### Recent Regression Debugging
```
1. gitnexus_detect_changes({scope: "compare", base_ref: "main"})
2. Review changed symbols
3. Review affected processes
4. gitnexus_context on changed symbols
5. Trace how changes affect execution flows
```

## Common Bug Patterns

### Null/Undefined Errors
**Symptom:** "Cannot read property 'x' of undefined"

**Approach:**
1. Find function mentioned in stack trace
2. `gitnexus_context` to see what provides the data
3. Trace backward to data source
4. Check for missing null checks

### External API Failures
**Symptom:** Intermittent 500 errors, timeouts

**Approach:**
1. `gitnexus_context` on failing endpoint
2. Look for outgoing calls to external services
3. Check for timeout/retry/error handling
4. Verify API contract assumptions

### Async Race Conditions
**Symptom:** Inconsistent behavior, sometimes works

**Approach:**
1. `gitnexus_context` on suspect function
2. Look for multiple async outgoing calls
3. Check if results are properly awaited
4. Verify execution order assumptions

### N+1 Query Problems
**Symptom:** Performance degrades with data size

**Approach:**
1. `gitnexus_context` to find database calls
2. Look for loops calling query functions
3. Use cypher to find call chain from loop
4. Identify opportunities for batching

### Circular Dependencies
**Symptom:** Initialization errors, undefined values

**Approach:**
```cypher
MATCH path = (a)-[:CodeRelation {type: 'IMPORTS'}*2..]->(a)
RETURN [n IN nodes(path) | n.name] AS cycle
```

### Wrong Data Transformations
**Symptom:** Output data doesn't match expected format

**Approach:**
1. `gitnexus_context` on function producing output
2. Trace outgoing calls to find transformations
3. `gitnexus_context` on each transformer
4. Read source to verify transformation logic

## Advanced Techniques

### Call Chain Analysis
Find all paths from entry point to suspect:
```cypher
MATCH path = (entry:Function {name: "apiHandler"})-[:CodeRelation {type: 'CALLS'}*]->(suspect:Function {name: "failingFunction"})
RETURN [n IN nodes(path) | n.name + " (" + n.filePath + ")"] AS chain
ORDER BY length(path) ASC
LIMIT 5
```

### Find All Error Throw Sites
```cypher
MATCH (f)-[:CodeRelation {type: 'THROWS'}]->(error)
WHERE error.name CONTAINS "PaymentError"
RETURN f.name, f.filePath, error.name
```

### Data Flow Tracing
```cypher
# Trace where data comes from (up to 3 hops backward)
MATCH path = (source)-[:CodeRelation {type: 'CALLS'}*1..3]->(target:Function {name: "processingData"})
RETURN [n IN nodes(path) | n.name] AS dataFlow
```

### External Dependency Finder
```cypher
MATCH (internal)-[:CodeRelation {type: 'CALLS'}]->(external)
WHERE external.filePath CONTAINS 'node_modules' OR external.filePath CONTAINS 'https://'
RETURN internal.name, internal.filePath, external.name
ORDER BY internal.name
```

## Integration with Other Workflows

### Debugging + Impact Analysis
```
1. Find bug with gitnexus_query/context
2. gitnexus_impact to understand blast radius of fix
3. Fix bug with knowledge of affected code
```

### Debugging + Exploring
```
1. Use exploring workflow to understand unfamiliar code
2. Use debugging workflow to trace specific failure
3. Combine knowledge for comprehensive understanding
```

### Debugging + Refactoring
```
1. Debug to find root cause
2. Use impact analysis before fixing
3. Refactor safely with gitnexus_rename if needed
```

## Tips

### Start Broad, Narrow Down
1. Use query to find related code
2. Use context to understand suspects
3. Read source for details
4. Use cypher for custom deep dives

### Look for External Dependencies
External API calls, database queries, file I/O are common sources of intermittent failures.

### Check Process Participation
If a symbol participates in multiple processes, it might have complex interactions.

### Use Confidence Scores
High confidence (>0.8) relationships are very reliable for tracing.

### Read the Source
GitNexus points you to the right code - always read the source for final confirmation.

### Follow the Data
For wrong return values, trace the data flow backward from the error point.

### Check for Async Issues
Intermittent failures often involve async operations without proper error handling.

## Related Workflows

- [Exploring](exploring.md) - Understand unfamiliar code
- [Impact Analysis](impact-analysis.md) - Understand blast radius of fixes
- [Refactoring](refactoring.md) - Refactor after finding bugs
