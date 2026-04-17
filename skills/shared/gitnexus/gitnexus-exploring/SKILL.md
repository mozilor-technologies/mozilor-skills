# Exploring Codebases with GitNexus

## When to Use

- "How does authentication work?"
- "What's the project structure?"
- "Show me the main components"
- "Where is the database logic?"
- Understanding code you haven't seen before
- Tracing execution flows to understand behavior

## Workflow

```
1. READ gitnexus://repos                          → Discover indexed repos
2. READ gitnexus://repo/{name}/context             → Codebase overview, check staleness
3. gitnexus_query({query: "<what you want to understand>"})  → Find related execution flows
4. gitnexus_context({name: "<symbol>"})            → Deep dive on specific symbol
5. READ gitnexus://repo/{name}/process/{name}      → Trace full execution flow
```

> If step 2 says "Index is stale" → run `npx gitnexus analyze` in terminal.

## Checklist

```
- [ ] READ gitnexus://repo/{name}/context
- [ ] gitnexus_query for the concept you want to understand
- [ ] Review returned processes (execution flows)
- [ ] gitnexus_context on key symbols for callers/callees
- [ ] READ process resource for full execution traces
- [ ] Read source files for implementation details
```

## MCP Resources

| Resource                                | What you get                                            |
| --------------------------------------- | ------------------------------------------------------- |
| `gitnexus://repo/{name}/context`        | Stats, staleness warning (~150 tokens)                  |
| `gitnexus://repo/{name}/clusters`       | All functional areas with cohesion scores (~300 tokens) |
| `gitnexus://repo/{name}/cluster/{name}` | Area members with file paths (~500 tokens)              |
| `gitnexus://repo/{name}/process/{name}` | Step-by-step execution trace (~200 tokens)              |
| `gitnexus://repo/{name}/processes`      | All execution flows (~300 tokens)                       |

## Tool Usage

### gitnexus_query

Find execution flows related to a concept using semantic search.

**Example:**
```javascript
gitnexus_query({query: "payment processing"})
```

**Returns:**
```
Processes: CheckoutFlow, RefundFlow, WebhookHandler
Symbols grouped by flow with file locations
```

**Parameters:**
- `query` (required): Natural language description of what you want to understand
- `repo` (optional): Repository name if multiple repos indexed
- `limit` (optional): Max number of results (default: 10)
- `threshold` (optional): Similarity threshold 0-1 (default: 0.7)

**When to use:**
- Starting point for understanding a feature area
- Finding code related to a concept or domain term
- Discovering execution flows you didn't know existed

### gitnexus_context

360-degree view of a specific symbol - all incoming calls, outgoing calls, and processes it participates in.

**Example:**
```javascript
gitnexus_context({name: "validateUser"})
```

**Returns:**
```
Incoming calls: loginHandler, apiMiddleware
Outgoing calls: checkToken, getUserById
Processes: LoginFlow (step 2/5), TokenRefresh (step 1/3)
```

**Parameters:**
- `name` (optional): Symbol name (e.g., "validateUser", "AuthService")
- `uid` (optional): Direct symbol UID from prior tool results (zero-ambiguity lookup)
- `file_path` (optional): File path to disambiguate common names
- `include_content` (optional): Include full symbol source code (default: false)
- `repo` (optional): Repository name if multiple repos indexed

**When to use:**
- Deep dive on a specific function, class, or method
- Understanding what calls this symbol (incoming)
- Understanding what this symbol calls (outgoing)
- Finding which execution flows touch this symbol

**Notes:**
- If multiple symbols share the same name, returns candidates for you to pick from
- ACCESSES edges (field read/write tracking) included with reason 'read' or 'write'
- CALLS edges resolve through field access chains (e.g., user.address.getCity())

## Example: "How does payment processing work?"

### Step 1: Check repository context
```
READ gitnexus://repo/my-app/context
→ 918 symbols, 45 processes, index up to date
```

### Step 2: Query for payment processing
```javascript
gitnexus_query({query: "payment processing"})
```

**Response:**
```
Process: CheckoutFlow
  - processPayment (src/payments/processor.ts:45)
  - validateCard (src/payments/validator.ts:23)
  - chargeStripe (src/payments/stripe.ts:67)

Process: RefundFlow
  - initiateRefund (src/payments/refunds.ts:12)
  - calculateRefund (src/payments/calculator.ts:34)
  - processRefund (src/payments/processor.ts:89)
```

### Step 3: Deep dive on processPayment
```javascript
gitnexus_context({name: "processPayment"})
```

**Response:**
```
Incoming calls:
  - checkoutHandler (src/api/checkout.ts:56) [CALLS, 100%]
  - webhookHandler (src/api/webhooks.ts:78) [CALLS, 100%]

Outgoing calls:
  - validateCard (src/payments/validator.ts:23) [CALLS, 100%]
  - chargeStripe (src/payments/stripe.ts:67) [CALLS, 100%]
  - saveTransaction (src/db/transactions.ts:45) [CALLS, 100%]

Processes:
  - CheckoutFlow (step 3/7)
  - WebhookHandler (step 2/4)
```

### Step 4: Trace full execution flow
```
READ gitnexus://repo/my-app/process/CheckoutFlow
```

**Response:**
```
CheckoutFlow (7 steps):
1. checkoutHandler → validateCart
2. validateCart → fetchInventory
3. checkoutHandler → processPayment
4. processPayment → validateCard
5. processPayment → chargeStripe
6. processPayment → saveTransaction
7. checkoutHandler → sendConfirmationEmail
```

### Step 5: Read implementation details
```
Read src/payments/processor.ts
→ See actual implementation, error handling, edge cases
```

## Patterns for Exploration

### Understanding a Feature Area
1. Use `gitnexus_query` with feature name or domain term
2. Review returned processes to understand main flows
3. Use `gitnexus_context` on key symbols from each flow
4. Read source files for implementation details

### Tracing Execution Flow
1. Use `gitnexus_context` on entry point (e.g., API handler)
2. Follow outgoing calls to understand call chain
3. READ process resource for full trace if available
4. Use `gitnexus_context` on callees for deeper understanding

### Finding Architectural Boundaries
1. READ `gitnexus://repo/{name}/clusters` for functional areas
2. READ `gitnexus://repo/{name}/cluster/{name}` for area members
3. Use `gitnexus_context` on symbols to see cross-area dependencies
4. Look for high coupling between areas

### Discovering Hidden Dependencies
1. Use `gitnexus_context` on a symbol
2. Review incoming calls for unexpected callers
3. Review outgoing calls for unexpected dependencies
4. Check which processes the symbol participates in

## Common Questions Answered

### "What calls this function?"
```javascript
gitnexus_context({name: "functionName"})
// Look at "Incoming calls" section
```

### "What does this function call?"
```javascript
gitnexus_context({name: "functionName"})
// Look at "Outgoing calls" section
```

### "Where is authentication implemented?"
```javascript
gitnexus_query({query: "authentication"})
// Review returned processes and symbols
```

### "Show me all database operations"
```javascript
gitnexus_query({query: "database operations"})
// Or query for specific patterns like "SQL queries", "transactions"
```

### "What are the main execution flows?"
```
READ gitnexus://repo/{name}/processes
// Lists all detected execution flows
```

## Tips

### Semantic Search Tips
- Use natural language: "payment processing", "user authentication"
- Use domain terms: "checkout flow", "error handling"
- Be specific when needed: "stripe payment integration"
- Try different phrasings if results aren't relevant

### Disambiguating Common Names
If `gitnexus_context({name: "User"})` returns multiple candidates:
1. Use `file_path` parameter: `gitnexus_context({name: "User", file_path: "src/models/user.ts"})`
2. Or use `uid` from previous results: `gitnexus_context({uid: "abc123"})`

### Exploring Large Codebases
1. Start with clusters to understand functional areas
2. Use query to find relevant processes
3. Use context on entry points to trace flows
4. Follow the trail of calls to understand implementation

### Following Complex Chains
For chains like `user.address.getCity().save()`:
- `gitnexus_context` includes ACCESSES edges
- CALLS edges resolve through field access chains
- Each step in the chain is tracked

## Related Workflows

- [Impact Analysis](impact-analysis.md) - Understand blast radius before changes
- [Debugging](debugging.md) - Trace bugs and errors
- [Refactoring](refactoring.md) - Safe code restructuring
