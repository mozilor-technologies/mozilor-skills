---
name: go-integration
description: Writing LLM provider integrations for the Braintrust Go SDK — TDD workflow, middleware/HTTP/callback patterns, VCR cassette testing, streaming, agentic spans, token normalization, orchestrion auto-instrumentation, and golangci-lint compliance.
---

# Writing Go SDK Integrations

Claude acts as the engineer implementing new LLM provider integrations to the Go SDK.

## When to Use This Skill

- Adding support for OpenAI-like or Anthropic-like providers
- Implementing LLM framework integrations (LangChain, etc.)
- Writing streaming/non-streaming tracer tests
- Setting up VCR cassettes and orchestrion auto-instrumentation

---

## Integration Patterns

Choose the pattern that matches your provider SDK's design:

| Pattern | Reference | Use When |
|---------|-----------|----------|
| **Middleware** | `trace/contrib/openai/` | SDK supports `option.WithMiddleware()` |
| **Middleware** | `trace/contrib/anthropic/` | SDK supports `option.WithMiddleware()` |
| **HTTP Wrapper** | `trace/contrib/genai/` | SDK accepts custom `*http.Client` |
| **HTTP Wrapper** | `trace/contrib/github.com/sashabaranov/go-openai/` | SDK accepts custom `*http.Client` |
| **Callback** | `trace/contrib/langchaingo/` | SDK has callback/handler interface |

**Before starting**: Examine the provider library's docs and source to identify ALL methods that call LLM APIs.

### Pattern 1: Middleware-Based

- Reference: `trace/contrib/openai/traceopenai.go`
- Key components: `NewMiddleware()`, `middlewareConfig` struct, URL router
- Uses: `trace/internal.Middleware()` helper with a router function
- Endpoint tracers: separate files per endpoint (`chatcompletions.go`, `responses.go`)

### Pattern 2: HTTP Client Wrapper

- Reference: `trace/contrib/genai/tracegenai.go`
- Key components: `WrapClient()`, custom `roundTripper` implementing `http.RoundTripper`
- Intercepts request/response at transport level

### Pattern 3: Callback-Based

- Reference: `trace/contrib/langchaingo/tracelangchaingo.go`
- Key components: handler struct implementing SDK's callback interface
- Manages span stack for nested calls (chain → llm → tools)

---

## Required Components (in order)

- [ ] **Core tracer**: `trace/contrib/yourprovider/traceyourprovider.go`
- [ ] **Endpoint parsers**: `trace/contrib/yourprovider/messages.go` (etc.)
- [ ] **Tests**: `trace/contrib/yourprovider/traceyourprovider_test.go`
- [ ] **VCR cassettes**: `trace/contrib/yourprovider/testdata/cassettes/`
- [ ] **Orchestrion config**: `trace/contrib/yourprovider/orchestrion.yml`
- [ ] **Orchestrion deps**: `trace/contrib/yourprovider/orchestrion.go`
- [ ] **Update all package**: Add import to `trace/contrib/all/all.go`
- [ ] **Run generate**: `make generate` to update combined orchestrion.yml
- [ ] **Customer example**: `examples/yourprovider/main.go`
- [ ] **Internal example**: `examples/internal/yourprovider/main.go`

---

## Endpoint-Specific Tracers

Follow the `internal.MiddlewareTracer` interface pattern:
- Reference: `trace/internal/middleware.go`
- Example: `trace/contrib/anthropic/messages.go`

Key methods:
- `StartSpan()` — parse request, start span, set input/metadata attributes
- `TagSpan()` — parse response, set output/metrics attributes

---

## Streaming

Aggregate chunks and capture final usage for streaming responses.

- Reference: `trace/contrib/anthropic/messages.go` (streaming handling)
- Reference: `trace/contrib/openai/chatcompletions.go` (tool call aggregation)

---

## Test Coverage

1. Non-streaming requests (basic + attributes + metrics)
2. Streaming requests (full consumption)
3. Early stream termination (close without reading)
4. Error handling (network errors, API errors)
5. **All critical features**:
   - Tool/function calling — verify `span_attributes.type = "tool"`, input args, output result
   - Agentic spans — if SDK has callback/handler system, capture tool calls and subagent invocations
   - Images/vision (if supported)
   - System messages (if supported)
   - Multiple messages/chat history
   - Provider-specific features (reasoning, caching, etc.)
6. Token usage edge cases (cached tokens, reasoning tokens)
7. Multiple APIs (if provider has multiple endpoints)

---

## Agentic Spans

When the SDK has an event/callback system, capture all steps as spans:

- **Tool calls**: `span_attributes.type = "tool"`. Set `input` to tool arguments, `output` to result, `metadata.name` to tool name.
- **Subagents / nested agents**: capture as child spans with type `"function"` or `"task"`, include subagent name.
- **Graph nodes / chains**: capture per-node callbacks (retrievers, embedders, rerankers) with appropriate span type.

**Key pattern**: Dispatch on the SDK's callback input/output type to determine span kind. Fall through to `return ctx` for unknown types.

**Internal example must cover**: at minimum one full agentic turn — model call → tool execution → model incorporating result.

---

## VCR Testing

Use `internal/vcr` and `internal/oteltest` for HTTP recording/replay and span verification.

```go
func setUpTest(t *testing.T) (*ProviderClient, *oteltest.Exporter) {
    exporter := oteltest.Setup(t)
    httpClient := vcr.NewHTTPClient(t)  // cassette auto-named from t.Name()
    client := NewClient(WithHTTPClient(httpClient))
    return client, exporter
}
```

**Key patterns:**
- Use dummy API key in replay mode, real key in record/off modes
- Use `oteltest.NewTimer()` and `timer.Tick()` for timing assertions
- Use `exporter.FlushOne()` or `exporter.Flush()` to get spans
- Span helpers: `AssertNameIs()`, `AssertInTimeRange()`, `Metadata()`, `Metrics()`, `Input()`, `Output()`

**VCR Modes:**
- `VCR_MODE=replay` (default) — use recorded cassettes
- `VCR_MODE=record` — record new cassettes (requires API keys)
- `VCR_MODE=off` — live API calls (requires API keys)

**Cassette location**: `testdata/cassettes/<TestFunctionName>.yaml`

---

## Orchestrion Auto-Instrumentation

Provides compile-time tracing injection with zero code changes.

**Required files:**
1. `orchestrion.yml` — join-points and advice (OpenAI pattern for middleware, GenAI for HTTP wrapper)
2. `orchestrion.go` — blank imports ensuring dependencies are in module graph
3. Add import to `trace/contrib/all/all.go`
4. Run `make generate` to update combined orchestrion.yml

---

## Examples

**Customer example** (`examples/yourprovider/main.go`):
- Concise, shows basic usage with manual middleware
- Creates root span, makes API call, prints permalink
- **MUST use real model SDK** — no mocks, stubs, or fake responses

**Internal example** (`examples/internal/yourprovider/main.go`):
- Comprehensive feature coverage for CI validation
- Must cover: non-streaming, streaming, tool calling (agentic turn), multiple providers
- Skip sections gracefully when optional API keys are not set:
  ```go
  if key := os.Getenv("ANTHROPIC_API_KEY"); key != "" { ... }
  ```
- Read API keys from environment variables only

> **Rule**: Always use real provider SDKs with real API keys. Never use mock models, stub implementations, or hardcoded fake responses.

---

## TDD Workflow

After **every major change**: test → lint → fix → commit

1. Write one failing test
2. Implement minimal code to pass
3. Run tests: `make test` (VCR replay mode)
4. Record cassettes when needed: `VCR_MODE=record go test -v -run=TestName ./path`
5. Lint: `make lint`
6. Run CI: `make ci` before committing
7. Repeat: basic → streaming → errors → tools → tokens

---

## Token Normalization

Normalize provider-specific token fields to standard Braintrust metric names:

| Metric | Description |
|--------|-------------|
| `prompt_tokens` | Input tokens (`input_tokens` or `prompt_tokens`) |
| `completion_tokens` | Output tokens (`output_tokens` or `completion_tokens`) |
| `tokens` | Total tokens |
| `prompt_cached_tokens` | Cache read tokens |
| `prompt_cache_creation_tokens` | Cache write tokens |
| `completion_reasoning_tokens` | Reasoning tokens |
| `time_to_first_token` | Streaming latency (seconds) |

---

## Span Attributes

| Attribute | Description |
|-----------|-------------|
| `braintrust.input_json` | Request input (messages array) |
| `braintrust.output_json` | Response output (content) |
| `braintrust.metadata` | Provider, model, parameters |
| `braintrust.metrics` | Token counts, timing |
| `braintrust.span_attributes` | Span type info |

---

## Defensive Coding

- Nil checks before accessing nested fields
- Type assertions with ok checks: `if v, ok := m["key"].(string); ok { ... }`
- Error handling with proper span status
- JSON serialization safety (handle marshal errors)
- Graceful handling of missing/unexpected response fields

---

## Linting & CI

```bash
make lint   # Run golangci-lint
make fmt    # Format code
make test   # Run tests (VCR replay)
make ci     # Full CI: lint + test + build
```
