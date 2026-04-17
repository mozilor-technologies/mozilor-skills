---
name: go-code-review
description: Go code review checklist and patterns — error handling, concurrency, interfaces, resource management, and Kubernetes operator conventions from the OpenShift Lightspeed Operator.
---

# Go Code Review

## Quick Reference

| Issue Type | Section |
|------------|---------|
| Missing error checks, wrapped errors | [Error Handling](#error-handling) |
| Race conditions, channel misuse | [Concurrency](#concurrency) |
| Interface pollution, naming | [Interfaces](#interfaces) |
| Resource leaks, defer misuse | [Common Mistakes](#common-mistakes) |

## Review Checklist

- [ ] All errors are checked (no `_ = err`)
- [ ] Errors wrapped with context (`fmt.Errorf("...: %w", err)`)
- [ ] Resources closed with `defer` immediately after creation
- [ ] No goroutine leaks (channels closed, contexts canceled)
- [ ] Interfaces defined by consumers, not producers
- [ ] Interface names end in `-er` (Reader, Writer, Handler)
- [ ] Exported names have doc comments
- [ ] No naked returns in functions > 5 lines
- [ ] Context passed as first parameter
- [ ] Mutexes protect shared state, not methods

### Kubernetes Operator Specific

- [ ] Owner references set with `controllerutil.SetControllerReference()`
- [ ] Finalizers added/removed safely (check for DeletionTimestamp)
- [ ] Context propagated through reconcile loops
- [ ] Client errors handled (distinguish NotFound vs other errors)
- [ ] Status updates separate from spec changes
- [ ] Reconcile functions are idempotent
- [ ] Resource updates check semantic equality first (`apiequality.Semantic.DeepEqual`)
- [ ] Return `ctrl.Result{Requeue: true}` for transient issues, errors for permanent failures
- [ ] RBAC markers (`//+kubebuilder:rbac`) present for all resource access in controllers

## Valid Patterns (Do NOT Flag)

- **`_ = err` with reason comment** — intentionally ignored with explanation
  ```go
  _ = conn.Close() // Best effort cleanup, already handling primary error
  ```
- **Empty `interface{}`** — acceptable for truly generic code in pre-generics codebases
- **Naked returns in short functions** — acceptable in functions < 5 lines with named returns
- **Channel without close** — when consumer stops via context cancellation, not channel close
- **Mutex protecting struct fields** — correct encapsulation even if accessed only via methods
- **`//nolint` with reason** — acceptable with explanation
  ```go
  //nolint:errcheck // Error logged but not returned per API contract
  ```
- **Defer in loop** — when function-scope cleanup is intentional (e.g. batch file processing)

## Context-Sensitive Rules

| Issue | Flag ONLY IF |
|-------|-------------|
| Missing error check | Error return is actionable (can retry, log, or propagate) |
| Goroutine leak | No context cancellation path exists for the goroutine |
| Missing defer | Resource isn't explicitly closed before next acquisition or return |
| Interface pollution | Interface has > 1 method AND only one consumer exists |

---

## Error Handling

### Critical Anti-Patterns

#### 1. Ignoring Errors

```go
// BAD
file, _ := os.Open("config.json")
data, _ := io.ReadAll(file)

// GOOD
file, err := os.Open("config.json")
if err != nil {
    return fmt.Errorf("opening config: %w", err)
}
defer file.Close()
```

#### 2. Unwrapped Errors

```go
// BAD - raw error loses context
if err != nil {
    return err
}

// GOOD - wrap with context
if err != nil {
    return fmt.Errorf("loading user %d: %w", userID, err)
}
```

#### 3. String Errors Instead of Wrapping

```go
// BAD - breaks errors.Is/As chain
return fmt.Errorf("failed: %s", err.Error())

// GOOD
return fmt.Errorf("failed: %w", err)
```

#### 4. Panic for Recoverable Errors

```go
// BAD
func GetConfig(path string) Config {
    data, err := os.ReadFile(path)
    if err != nil {
        panic(err)
    }
}

// GOOD
func GetConfig(path string) (Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return Config{}, fmt.Errorf("reading config: %w", err)
    }
}
```

#### 5. Checking Error String Instead of Type

```go
// BAD - brittle
if err.Error() == "file not found" { }

// GOOD
if errors.Is(err, os.ErrNotExist) { }
```

#### 6. Non-Zero Value Returned Alongside Error

```go
// BAD - -1 is a valid integer, confusing callers
return -1, errors.New("empty string")

// GOOD - zero value on error
return 0, errors.New("empty string")
```

### Sentinel Errors Pattern

```go
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
)

func GetUser(id int) (*User, error) {
    user := db.Find(id)
    if user == nil {
        return nil, ErrNotFound
    }
    return user, nil
}

// Caller
if errors.Is(err, ErrNotFound) {
    http.Error(w, "User not found", 404)
}
```

### Review Questions

1. Are all error returns checked (no `_`)?
2. Are errors wrapped with context using `%w`?
3. Are sentinel errors used for expected error conditions?
4. Does the code use `errors.Is/As` instead of string matching?
5. Does it return zero values alongside errors?

---

## Concurrency

### Critical Anti-Patterns

#### 1. Goroutine Leak

```go
// BAD - no way to stop the goroutine
func startWorker() {
    go func() {
        for { doWork() }
    }()
}

// GOOD
func startWorker(ctx context.Context) {
    go func() {
        for {
            select {
            case <-ctx.Done():
                return
            default:
                doWork()
            }
        }
    }()
}
```

#### 2. Unbounded Channel Send

```go
// BAD - blocks if nobody reads
ch <- result

// GOOD
select {
case ch <- result:
case <-ctx.Done():
    return ctx.Err()
}
```

#### 3. Closing Channel Multiple Times

```go
// BAD - potential double close panics
close(ch)
close(ch)

// GOOD - only sender closes, exactly once
func produce(ch chan<- int) {
    defer close(ch)
    for i := 0; i < 10; i++ {
        ch <- i
    }
}
```

#### 4. Race Condition on Shared State

```go
// BAD - concurrent map access
var cache = make(map[string]int)
func Get(key string) int { return cache[key] }

// GOOD
var (
    cache   = make(map[string]int)
    cacheMu sync.RWMutex
)
func Get(key string) int {
    cacheMu.RLock()
    defer cacheMu.RUnlock()
    return cache[key]
}
```

#### 5. Missing WaitGroup

```go
// BAD - may exit before goroutines finish
for _, item := range items {
    go process(item)
}
return

// GOOD
var wg sync.WaitGroup
for _, item := range items {
    wg.Add(1)
    go func(item Item) {
        defer wg.Done()
        process(item)
    }(item)
}
wg.Wait()
```

#### 6. Loop Variable Capture (pre-Go 1.22)

```go
// BAD
for _, item := range items {
    go func() { process(item) }()  // all see last item
}

// GOOD
for _, item := range items {
    go func(item Item) { process(item) }(item)
}
```

#### 7. Context Not Propagated

```go
// BAD
func Handler(ctx context.Context) error {
    result := doWork()  // ignores ctx
    return nil
}

// GOOD
func Handler(ctx context.Context) error {
    result, err := doWork(ctx)
    return err
}
```

### Worker Pool Pattern

```go
func processItems(ctx context.Context, items []Item) error {
    const workers = 5
    jobs := make(chan Item)
    errs := make(chan error, 1)

    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for item := range jobs {
                if err := process(ctx, item); err != nil {
                    select {
                    case errs <- err:
                    default:
                    }
                    return
                }
            }
        }()
    }

    go func() {
        wg.Wait()
        close(errs)
    }()

    for _, item := range items {
        select {
        case jobs <- item:
        case err := <-errs:
            return err
        case <-ctx.Done():
            return ctx.Err()
        }
    }
    close(jobs)
    return <-errs
}
```

### Review Questions

1. Are all goroutines stoppable via context?
2. Are channels always closed by the sender?
3. Is shared state protected by mutex or sync types?
4. Are WaitGroups used to wait for goroutine completion?
5. Is context passed through the call chain?

---

## Interfaces

### Critical Anti-Patterns

#### 1. Premature Interface Definition (in producer package)

```go
// BAD - defined in producer
package storage
type UserRepository interface { Get(id int) (*User, error) }

// GOOD - defined in consumer
package service
type UserGetter interface { Get(id int) (*User, error) }
func NewUserService(users UserGetter) *UserService { ... }
```

#### 2. Fat Interfaces

```go
// BAD
type UserStore interface {
    Get(id int) (*User, error)
    GetAll() ([]*User, error)
    Save(user *User) error
    Delete(id int) error
    // ... more methods
}

// GOOD - focused, composable
type UserGetter interface { Get(id int) (*User, error) }
type UserSaver interface { Save(user *User) error }
type UserStore interface { UserGetter; UserSaver }
```

#### 3. Wrong Interface Names

```go
// BAD
type IUserService interface { ... }       // Java-style
type UserServiceInterface interface { ... } // redundant

// GOOD - verb ending in -er
type UserReader interface { ReadUser(id int) (*User, error) }
type UserWriter interface { WriteUser(user *User) error }
```

#### 4. Returning Interface Instead of Concrete Type

```go
// BAD
func NewServer(addr string) Server { return &httpServer{addr: addr} }

// GOOD
func NewServer(addr string) *HTTPServer { return &HTTPServer{addr: addr} }
```

#### 5. Empty Interface Overuse

```go
// BAD
func Process(data interface{}) interface{} { ... }

// GOOD - use generics (Go 1.18+)
func Process[T string | int](data T) T { ... }
```

### Accept Interfaces, Return Structs

```go
func WriteData(w io.Writer, data []byte) error {   // accepts interface
    _, err := w.Write(data)
    return err
}

func NewBuffer() *bytes.Buffer {                   // returns concrete type
    return &bytes.Buffer{}
}
```

### Review Questions

1. Are interfaces defined where they're used (consumer side)?
2. Are interfaces minimal (1–3 methods)?
3. Do interface names end in `-er`?
4. Are concrete types returned from constructors?
5. Is `interface{}` avoided in favor of generics or specific types?

---

## Common Mistakes

### Resource Leaks

#### 1. Missing defer for Close

```go
// BAD - file never closed on early return
f, err := os.Open(path)
if err != nil { return nil, err }
data, err := io.ReadAll(f)
if err != nil { return nil, err }  // leak!
f.Close()

// GOOD - defer immediately after open
f, err := os.Open(path)
if err != nil { return nil, err }
defer f.Close()
return io.ReadAll(f)
```

#### 2. Defer in Loop

```go
// BAD - files stay open until function returns
for _, path := range paths {
    f, _ := os.Open(path)
    defer f.Close()
    process(f)
}

// GOOD - wrap in closure
for _, path := range paths {
    func() {
        f, _ := os.Open(path)
        defer f.Close()
        process(f)
    }()
}
```

#### 3. HTTP Response Body Not Closed

```go
// BAD - connection pool exhaustion
resp, err := http.Get(url)
if err != nil { return err }
data, _ := io.ReadAll(resp.Body)  // body never closed

// GOOD
resp, err := http.Get(url)
if err != nil { return err }
defer resp.Body.Close()
data, _ := io.ReadAll(resp.Body)
```

### Naming and Style

#### 4. Stuttering Names

```go
// BAD
package user
type UserService struct { ... }  // user.UserService

// GOOD
package user
type Service struct { ... }  // user.Service
```

#### 5. Missing Doc Comments on Exports

```go
// BAD
func NewServer(addr string) *Server { ... }

// GOOD
// NewServer creates a new HTTP server listening on addr.
func NewServer(addr string) *Server { ... }
```

#### 6. Naked Returns in Long Functions

```go
// BAD
func process(data []byte) (result string, err error) {
    // 50 lines...
    return  // unclear what's returned
}

// GOOD
func process(data []byte) (string, error) {
    // 50 lines...
    return processedString, nil
}
```

### Initialization

#### 7. Init Function Overuse

```go
// BAD - hidden side effects
var db *sql.DB
func init() {
    db, _ = sql.Open("postgres", os.Getenv("DATABASE_URL"))
}

// GOOD - explicit initialization
type App struct{ db *sql.DB }
func NewApp(dbURL string) (*App, error) {
    db, err := sql.Open("postgres", dbURL)
    if err != nil {
        return nil, fmt.Errorf("opening db: %w", err)
    }
    return &App{db: db}, nil
}
```

#### 8. Global Mutable State

```go
// BAD - race conditions, untestable
var config Config
func GetConfig() Config { return config }

// GOOD - dependency injection
type Server struct{ config Config }
func NewServer(cfg Config) *Server { return &Server{config: cfg} }
```

### Performance

#### 9. String Concatenation in Loop

```go
// BAD - O(n²)
var result string
for _, s := range items { result += s + ", " }

// GOOD
var b strings.Builder
for _, s := range items { b.WriteString(s); b.WriteString(", ") }
result := b.String()
```

#### 10. Slice Preallocation

```go
// BAD - repeated reallocations
var results []Result
for _, item := range items { results = append(results, process(item)) }

// GOOD
results := make([]Result, 0, len(items))
for _, item := range items { results = append(results, process(item)) }
```

### Testing

#### 11. Table-Driven Tests

```go
// BAD - repetitive
func TestAdd(t *testing.T) {
    if Add(1, 2) != 3 { t.Error("1+2 should be 3") }
    if Add(0, 0) != 0 { t.Error("0+0 should be 0") }
}

// GOOD
func TestAdd(t *testing.T) {
    tests := []struct{ a, b, want int }{
        {1, 2, 3},
        {0, 0, 0},
        {-1, 1, 0},
    }
    for _, tt := range tests {
        got := Add(tt.a, tt.b)
        if got != tt.want {
            t.Errorf("Add(%d, %d) = %d, want %d", tt.a, tt.b, got, tt.want)
        }
    }
}
```

### Review Questions

1. Is `defer Close()` called immediately after opening resources?
2. Are HTTP response bodies always closed?
3. Are package-level names free of package-name stuttering?
4. Do exported symbols have doc comments?
5. Is mutable global state avoided?
6. Are slices preallocated when size is known?
